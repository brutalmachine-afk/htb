# Layover - Hack The Box Walkthrough

> **Platform:** Hack The Box
> **Machine:** Layover
> **Difficulty:** Medium
> **Category:** Linux
> **Techniques:** RDP, wireless packet capture, credential sniffing, Craft CMS, database enumeration, Craft/Yii decryption, SSH, and CUPS privilege escalation

### Attack Path

Contractor credentials

↓

RDP into Linux workstation

↓

Capture local wireless traffic

↓

Recover Jenny's credentials

↓

Access Craft CMS `/admin`

↓

Craft CMS RCE

↓

Find and decrypt mail relay password

↓

SSH as `aporter`
↓
CUPS privilege escalation
↓
Root

## 1. Initial Enumeration
The box gives us the following contractor credentials:

```text
Username: contractor
Password: Contractor2026!
```

A port scan shows SSH and RDP are open. The credentials don't work over SSH, but they do work over RDP.

After logging in, we're dropped onto a Linux desktop.

Opening the browser shows an internal portal that's only reachable while the workstation is connected to the local wireless network. There's a login page, but we don't have credentials for it yet.

Since we're sitting on the same wireless network as other users, the next thing to check is whether we can capture anything useful from their traffic.

## 2. Wireless Network Traffic Capture
First, check the wireless interfaces:

```bash
iwconfig
```

`wlan3` is the interface we're interested in, and it's currently running in managed mode.
Put it into monitor mode:

```bash
sudo airmon-ng start wlan3
```

This creates:

```text
wlan3mon
```

Now start capturing traffic:

```bash
sudo airodump-ng wlan3mon -w CAPTURE
```

We can see an `OPN` wireless network. Let the capture run for a while so clients have time to generate traffic, then stop it with `Ctrl+C`.
The capture gets saved as something like:

```text
CAPTURE-01.cap
```

If needed, filter it down to the target BSSID:

```bash
sudo airdecap-ng -b <TARGET_BSSID> CAPTURE-01.cap
```

Opening the capture in Wireshark and looking through the HTTP traffic reveals credentials being sent across the network:

```text
Username: jenny
Password: Fl1ghtDeck2026!
```

These credentials work on the portal, although there isn't much useful in the normal user-facing portion of the site.
## 3. Finding the Craft CMS Admin Panel

Looking at the application's cookies shows that the site is running **Craft CMS**.
Craft normally exposes its control panel under `/admin`, so I tried:

```text
http://portal.international.htb/admin
```

Jenny's credentials work there too:
```text
Username: jenny
Password: Fl1ghtDeck2026!
```

Once logged in, the Craft version is shown at the bottom of the page.

Looking into vulnerabilities for that version leads to **CVE-2026-44011**, an authenticated RCE affecting Craft CMS.
Reference: https://github.com/4xura/CVE-2026-44011-craftcms-auth-rce/tree/main
## 4. Craft CMS RCE - CVE-2026-44011

The issue is in the way Craft and Yii handle attacker-controlled configuration data during object creation.

The important part for this box is that we already have authenticated access as Jenny, so the vulnerable functionality is reachable.

The public PoC abuses Craft's condition handling and Yii behavior configuration to get command execution on the server.

I started a listener:

```bash
nc -lvnp 4444
```

Then used the PoC with Jenny's account and my HTB VPN IP as the callback address.
Once the payload executed, the listener caught a shell running as:

```text
www-data
```

## 5. Database Enumeration

With access to the web server, I started looking through the Craft configuration.
The project's `.env` file contains the database credentials as well as the Craft security key:

```text
CRAFT_SECURITY_KEY=IGckihiFK64_lrSgJJ6QLkiPz-ow13Lr
```

Using the credentials from `.env`, I connected to MariaDB and looked at the `craft` database:

```sql
SHOW TABLES;
```

There are 72 tables, including a few that stand out:

```text
htbairways_settings
miles_members
sso_identities
users
```
### Craft users

First, I checked the users:
```sql
SELECT * FROM users;
```

This shows two Craft accounts:
| 1 | `admin` | - | `admin@htb-international.htb` | Yes |
| 2 | `jenny` | Jenny Crawford | `jenny.crawford@htb-international.htb` | No |

The passwords are stored as bcrypt hashes, so instead of spending time there I moved on to the application-specific settings.
### HTB Airways settings

```sql
SELECT * FROM htbairways_settings;
```

This gives us mail relay configuration:

```text
mailRelayHost = mail.htbairways.htb
mailRelayPort = 587
mailRelayUser = aporter
mailRelayPassword = <encrypted value>
```
The password isn't plaintext, but we already have something important from `.env`: the Craft security key.
## 6. Decrypting the Mail Relay Password

Craft uses Yii's security functions for encrypted application values. Since we have both the encrypted database value and `CRAFT_SECURITY_KEY`, the password can be recovered using the application's existing Craft/Yii environment.

After decoding the stored value and passing it through Yii's corresponding decryption function with the Craft security key, the mail relay password comes back as:

```text
Skyp0rt_Relay!26
```

So at this point we have:

```text
User: aporter
Password: Skyp0rt_Relay!26
```
## 7. SSH as `aporter`
Checking `/etc/passwd` shows that `aporter` is also a local user.
Since password reuse is always worth checking, I tried the mail relay password over SSH:

```bash
ssh aporter@portal.international.htb
```

The reused password works.
We're now logged in as `aporter` and can grab the user flag.
## 8. Local Enumeration

From here I started going through the normal privilege escalation checks.
`sudo -l` doesn't give us anything useful, and the SUID binaries don't immediately stand out either.

Checking listening services is more interesting:

```bash
ss -tulnp
```
Port `631` is listening locally.
Querying it:
```bash
curl http://localhost:631
```
shows that the box is running **CUPS 2.4.16**. 
## 9. CUPS Privilege Escalation

Researching the installed CUPS version leads to **CVE-2026-34990**.
Reference: https://github.com/0xc4rc3l/CVE-2026-34990-poc/tree/main

The vulnerability gives a local user a path to abuse CUPS and turn access to the printing service into a privileged file write.

For Layover, that provides the final privilege escalation path.

After running the PoC from the repository in the context of the box, the resulting privileged write can be used to give `aporter` the required sudo access.

From there, start a root shell and retrieve the final flag from `/root`.
## 10. Conclusion

Layover was an interesting box because there wasn't really one main vulnerability that got us all the way through.

The contractor account only gets us onto the workstation. From there, the wireless network leaks Jenny's credentials, which also happen to work on the Craft admin panel. The Craft vulnerability gets us onto the web server, where the database and Craft security key expose another password.

That password is reused by the local `aporter` account, giving us SSH access.

Finally, local enumeration finds CUPS listening on port 631. The vulnerable CUPS version gives us the last step needed to escalate from `aporter` to root.
## Key Takeaways

- Don't assume internal or wireless traffic is safe just because it isn't Internet-facing.
- Credentials for a normal application may also work against its administrative interface.
- Version information can quickly narrow down vulnerability research.
- Encrypting application secrets doesn't help much if the encryption key is stored on the same compromised server.
- Password reuse between application and operating system accounts can turn a web compromise into SSH access.
- Local-only services still matter during privilege escalation.
