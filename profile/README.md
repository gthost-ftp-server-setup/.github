
Whether you just spun up your first GTHost VPS or you're migrating from some overpriced shared host, there's one task that trips up a surprising number of people early on: getting FTP — or ideally, SFTP — working so you can actually move files around.

You ordered the server. It provisioned in under 15 minutes (yes, that's real). You got the root credentials in your email. And now you're staring at a blank terminal wondering, "okay, so how do I log into the FTP server and set this thing up?"

This guide walks you through the whole thing — from understanding what FTP actually is on a Linux server, to installing and configuring vsftpd or using SFTP via FileZilla, to fixing the common connection timeout bugs that bite nearly everyone the first time. By the end, you'll have a working, secure file transfer setup on your GTHost server.

---

## Before You Start: What You Actually Need

Here's what to have ready before you open a terminal:

- Your GTHost server's **IP address** (from the provisioning email)
- Your **root username and password** (also in that email)
- An SSH client — **PuTTY** on Windows, or just the built-in **Terminal** on Mac/Linux
- An FTP/SFTP client — **FileZilla** is free and works on everything
- About 15 minutes

👉 [Don't have a server yet? Get started with GTHost here](https://cp.gthost.com/en/join/d2033d997295e5ce2498ba05a9980fdc)

---

## Step 1: Connect to Your GTHost Server via SSH

Before setting up any FTP service, you need to SSH into your server. This is your starting point for everything.

**On Mac or Linux**, open Terminal and run:

bash
ssh root@YOUR_SERVER_IP


Replace `YOUR_SERVER_IP` with the IP from your GTHost provisioning email. Enter the root password when prompted.

**On Windows with PuTTY**: Open PuTTY, enter your server IP in the "Host Name" field, set Port to 22, select SSH, and click Open. Log in as `root` with your password.

Once you're in, you'll see the Linux command prompt. That's your green light to proceed.

---

## Step 2: Understand FTP vs. SFTP — Pick the Right One

Here's where most guides skip something important: **plain FTP and SFTP are completely different protocols**, even though they sound related.

**Plain FTP** (port 21): The classic protocol. Simple, widely supported — but it transmits your credentials and data in plain text. On a public internet connection, that's a real security risk. Avoid it unless you're in a closed, private network.

**SFTP** (SSH File Transfer Protocol, port 22): Not actually FTP — it's a subsystem of SSH. Encrypted by default, uses the same port as your SSH connection, and is far easier to configure correctly on a Linux VPS. This is what most people should use.

**FTPS** (FTP over TLS, port 990 or 21 with STARTTLS): Adds SSL/TLS encryption to traditional FTP. Compatible with many legacy clients, but requires certificate management and careful firewall rules for passive mode.

**Recommendation**: If you're using a GTHost Linux VPS or dedicated server, start with **SFTP**. It works out of the box because SSH is already running. No additional installation needed.

---

## Step 3: Log In via SFTP Instantly (No Installation Required)

Because GTHost automatically deploys your server with SSH enabled, SFTP works immediately after provisioning. No setup needed.

**Using FileZilla:**

1. Open FileZilla
2. Click **File → Site Manager → New Site**
3. Set Protocol to **SFTP – SSH File Transfer Protocol**
4. Enter your GTHost server **IP address** as the Host
5. Port: **22**
6. Logon Type: **Normal**
7. Username: **root** (or a non-root user you've created)
8. Password: your root password
9. Click **Connect**

That's it. FileZilla will connect and show your server's filesystem on the right panel, your local files on the left. Drag and drop to transfer.

**Security note**: Using root for SFTP is fine to get started, but for any production use, create a dedicated non-root user and connect as that user instead.

---

## Step 4: Install a Traditional FTP Server (vsftpd) on Linux

If you specifically need plain FTP — maybe because a legacy application requires it, or you're setting up a shared FTP space for a team — here's how to install **vsftpd**, the most widely used and well-maintained FTP server for Linux.

**On Ubuntu/Debian (most common with GTHost):**

bash
sudo apt update
sudo apt install vsftpd -y


**On CentOS/RHEL:**

bash
sudo yum install vsftpd -y


Verify it's running:

bash
sudo systemctl status vsftpd


You should see `active (running)` in green. If not:

bash
sudo systemctl start vsftpd
sudo systemctl enable vsftpd


---

## Step 5: Configure vsftpd for Secure FTP Access

The default vsftpd configuration is pretty locked down, which is good. But you'll need to tweak it to allow local users to log in and upload files.

Open the config file:

bash
sudo nano /etc/vsftpd.conf


Find and set (or add) these lines:


anonymous_enable=NO
local_enable=YES
write_enable=YES
chroot_local_user=YES
allow_writeable_chroot=YES
pasv_enable=YES
pasv_min_port=30000
pasv_max_port=31000


What these do, briefly:

- `anonymous_enable=NO` — no anonymous logins (important for security)
- `local_enable=YES` — allows your Linux user accounts to log in
- `write_enable=YES` — allows file uploads
- `chroot_local_user=YES` — locks each user into their home directory (they can't browse the whole server)
- `pasv_min_port` / `pasv_max_port` — sets the passive port range (needed for most FTP clients behind NAT)

Save with `Ctrl+O`, then `Ctrl+X`. Restart vsftpd:

bash
sudo systemctl restart vsftpd


---

## Step 6: Open the Right Firewall Ports

This is the step that most people miss — and it causes that frustrating "connection timed out" error when you try to connect with FileZilla.

GTHost servers typically come with UFW (Ubuntu) or firewalld (CentOS) managing the firewall. You need to open ports for FTP traffic.

**On Ubuntu with UFW:**

bash
sudo ufw allow 20/tcp
sudo ufw allow 21/tcp
sudo ufw allow 30000:31000/tcp
sudo ufw reload


**On CentOS with firewalld:**

bash
sudo firewall-cmd --permanent --add-port=20-21/tcp
sudo firewall-cmd --permanent --add-port=30000-31000/tcp
sudo firewall-cmd --reload


Port 20 and 21 are for FTP data and control. The 30000–31000 range is your passive port window — this is critical for passive mode FTP, which is what most FTP clients (including FileZilla) use by default.

---

## Step 7: Create a Dedicated FTP User

You don't want to hand out your root credentials just to give someone FTP access. Create a dedicated user:

bash
sudo adduser ftpuser
sudo passwd ftpuser


Set a strong password when prompted. Now create their FTP root directory:

bash
sudo mkdir -p /home/ftpuser/upload
sudo chown root:root /home/ftpuser
sudo chmod 755 /home/ftpuser
sudo chown ftpuser:ftpuser /home/ftpuser/upload


The parent directory (`/home/ftpuser`) is owned by root and non-writable — this satisfies vsftpd's chroot security requirement. The `upload` subdirectory is where your FTP user can actually write files.

---

## Step 8: Connect with FileZilla Using FTP

Now fire up FileZilla and connect using FTP (not SFTP this time):

1. Open FileZilla
2. **File → Site Manager → New Site**
3. Protocol: **FTP – File Transfer Protocol**
4. Host: your GTHost server IP
5. Encryption: **Use explicit FTP over TLS if available** (for security)
6. Logon Type: **Normal**
7. Username: `ftpuser`
8. Password: the one you set
9. Click **Connect**

If it connects but hangs on the directory listing — that's the passive port issue. In FileZilla, go to **Edit → Settings → FTP → Passive Mode**, and set it to use the server's external IP. That usually fixes it.

---

## Step 9: Verify Your Setup Works

Upload a test file from your local machine to the `/upload` directory on the server. Then download it back. If both succeed, your FTP server is working.

To confirm from the command line:

bash
sudo tail -f /var/log/vsftpd.log


You'll see real-time login attempts and transfers. A successful login looks something like:


Mon Apr 07 10:23:41 2026 [pid 12345] CONNECT: Client "xxx.xxx.xxx.xxx"
Mon Apr 07 10:23:42 2026 [pid 12345] OK LOGIN: Client "xxx.xxx.xxx.xxx", "ftpuser"


If you see `FAIL LOGIN`, double-check the username, password, and that `local_enable=YES` is set in vsftpd.conf.

---

## Troubleshooting Common Issues

**"Connection timed out" on directory listing**
→ Passive port range isn't open. Re-check your firewall rules (Step 6) and make sure the `pasv_min_port` / `pasv_max_port` in vsftpd.conf match what you opened in the firewall.

**"500 OOPS: cannot change directory"**
→ The user's home directory permissions are wrong. Root should own the chroot directory (`/home/ftpuser`), and it should not be writable by the FTP user.

**"530 Login incorrect"**
→ Wrong username/password, or the user is listed in `/etc/ftpusers` (which blocks them). Check: `grep ftpuser /etc/ftpusers`. If they're listed there, remove them.

**FileZilla shows "critical error: Could not connect to server"**
→ Port 21 is probably blocked. Run `sudo ufw status` (Ubuntu) or `sudo firewall-cmd --list-ports` (CentOS) to confirm port 21 is open.

**Connection works but FTP is very slow**
→ This is a known characteristic of plain FTP on some GTHost servers — one real user review noted FTP access can be a bit sluggish. The fix: switch to SFTP (Step 3). SFTP via SSH typically performs better and skips the passive port complexity entirely.

---

## GTHost Server Plans: Full Comparison Table

GTHost offers a range of server types to match different workloads. Here's an overview of the current product categories — all of which support FTP/SFTP setup as described in this guide.

| Server Type | Starting Price | Bandwidth | Key Use Cases | Trial Available | Purchase Link |
|---|---|---|---|---|---|
| **VPS (KVM)** | From $4/mo | Unmetered | Blogs, small apps, dev environments | Yes ($5/day) |  [Get GTHost VPS](https://cp.gthost.com/en/join/d2033d997295e5ce2498ba05a9980fdc) |
| **1G Instant Dedicated** | From $59/mo | 300Mbps–1Gbps unmetered | High-traffic sites, ecommerce, game servers | Yes ($5/day, up to 10 days) |  [Get 1G Dedicated](https://cp.gthost.com/en/join/d2033d997295e5ce2498ba05a9980fdc) |
| **10G Instant Dedicated** | From $149/mo | 2Gbps–10Gbps unmetered | SaaS platforms, CDN, media streaming | Yes ($5/day, up to 10 days) |  [Get 10G Dedicated](https://cp.gthost.com/en/join/d2033d997295e5ce2498ba05a9980fdc) |
| **Storage Dedicated** | Varies | 300Mbps–1Gbps unmetered | Backups, archives, media storage, FTP repositories | Yes |  [Get Storage Server](https://cp.gthost.com/en/join/d2033d997295e5ce2498ba05a9980fdc) |
| **AMD EPYC Dedicated** | From $99/mo (Detroit) | 300Mbps–2Gbps unmetered | Analytics, parallel workloads, databases | Yes |  [Get AMD Server](https://cp.gthost.com/en/join/d2033d997295e5ce2498ba05a9980fdc) |
| **GPU Dedicated** | Varies | Up to 10Gbps | ML/AI inference, rendering, HPC | Yes |  [Get GPU Server](https://cp.gthost.com/en/join/d2033d997295e5ce2498ba05a9980fdc) |

**Current promotions (2026):**
- **Detroit data center** is running some of GTHost's lowest prices: Silver 4116 (12 core), 96GB RAM, 2×960GB SSD from **$79/mo**; AMD EPYC 7452 (32 core), 256GB RAM from **$189/mo**
- **Chicago**: 128GB, dual SSD, unmetered 300M–1Gbps from **$89/mo**
- **AMD EPYC sale** ongoing — check the promotions page for the latest configurations
- **AMD Ryzen 9950X** servers now live in Madrid, Toronto, Los Angeles, and Santa Clara

All plans come with free setup, no long-term contracts, Linux auto-deploy (Ubuntu, Debian, CentOS, Fedora), full root access, and 24/7 support.

👉 [See all GTHost servers and start your trial](https://cp.gthost.com/en/join/d2033d997295e5ce2498ba05a9980fdc)

---

## Which GTHost Plan Should You Choose for FTP Hosting?

**Just need SFTP for a personal project or small site?** The VPS plans starting at $4/month are more than enough. SFTP works instantly via SSH — no additional setup beyond what's described above.

**Running a shared FTP server for a team or clients?** A 1G Instant Dedicated server gives you full root control, dedicated resources, and enough bandwidth for concurrent FTP transfers without performance hits. Detroit and Chicago have the best prices right now.

**Need a dedicated file archive or backup FTP server?** The Storage Dedicated servers are built for exactly this — high-capacity drives, stable throughput, designed for large data volumes.

The one thing worth noting: GTHost servers are **unmanaged**, which means FTP setup is your responsibility. That's what this guide is for. If you can follow these steps (or you already know Linux basics), the unmanaged model saves you a lot of money compared to managed alternatives.

---

## A Few Security Reminders Before You Go Live

Running an FTP server on a public IP means it will get probed. A few things worth doing before you call this production-ready:

**Fail2ban** — Installs easily and automatically blocks IPs that repeatedly fail FTP logins. Highly recommended:

bash
sudo apt install fail2ban -y


**Disable root FTP login** — Keep root for SSH only. Your dedicated FTP user (from Step 7) is what should log into FTP.

**Use FTPS if plain FTP is required** — Add a Let's Encrypt certificate and configure `ssl_enable=YES` in vsftpd.conf so credentials are never transmitted in plain text.

**Prefer SFTP for everything else** — Seriously, unless a specific client or workflow requires FTP, SFTP is simpler, more secure, and already working on your GTHost server right now.

---

## Summary

Setting up FTP on a GTHost VPS or dedicated server is pretty straightforward once you know the steps. The short version: use SFTP if you can (it's already working), install vsftpd if you need traditional FTP, open your firewall ports (especially the passive range), create a dedicated user, and test it with FileZilla.

GTHost makes the server side of this easy — fast provisioning, full root access, and solid network performance give you a clean base to work from. The rest is just Linux config, and now you have the whole map.

👉 [Start with a GTHost trial from $5/day — no long-term commitment](https://cp.gthost.com/en/join/d2033d997295e5ce2498ba05a9980fdc)
