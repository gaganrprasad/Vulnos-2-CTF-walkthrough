# VulnOS: 2 Walkthrough

## Target IP: `192.168.224.21`

**CTF Type**: Linux Privilege Escalation  
**Difficulty Level**: Intermediate

### Table of Contents
1. [Introduction](#introduction)
2. [Reconnaissance](#reconnaissance)
   - [Network Discovery](#network-discovery)
   - [Port Scanning](#port-scanning)
3. [Enumeration](#enumeration)
4. [Exploitation](#exploitation)
   - [Initial Access](#initial-access)
5. [Privilege Escalation](#privilege-escalation)
6. [Post-Exploitation](#post-exploitation)
7. [Conclusion](#conclusion)

# VulnOS 2 Walkthrough

## Target Information
- **Target IP**: `192.168.224.21` 
- **CTF Type**: Linux Privilege Escalation
- **Difficulty Level**: Intermediate
- **Objective**: Gain root access to the VulnOS 2 machine.

---

### Introduction
This walkthrough provides a step-by-step guide to solving the VulnOS: 2 machine from VulnHub. The objective of this CTF challenge is to escalate privileges from a normal user to root on a Linux-based target. We will begin with network reconnaissance, followed by exploitation and privilege escalation techniques to capture the flag.

## Reconnaissance

### Network Discovery
To locate the target machine's IP address, we use `netdiscover` on our network.

1.Open a terminal in Kali Linux.(open as a root user or use the command 'sudo su -')
2.netdiscover -r 192.168.1.0/24

 After running the command, I identified the target machine's IP, which was 192.168.224.21

### Port Scanning
Next, we perform an Nmap scan to identify open ports and services running on the target.

1.nmap -sC -sV 192.168.224.21

The output of the Nmap scan reveals that the target machine has several open ports, including port
1.22/tcp   open  ssh     OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.6 (Ubuntu Linux; protocol 2.0)
2.80/tcp   open  http    httpd  Apache httpd 2.4.7 ((Ubuntu))
3.6667/tcp open  irc     ngircd
The results show that the machine has SSH and HTTP services running, which suggests potential attack vectors.

## Enumeration

### Enumerating HTTP Service
I navigate to the HTTP service running on port 80 by visiting `http://192.168.224.21` in a browser. The website reveals a simple page.

I was greeted with a static website with instructions to “pentest the company website” which seems to be hosted on the same server and to “Get root of the system and read the final flag”. A link to this website is also provided. On checking the source for this page, nothing valuable was found.
Clicking on the website link, we reach a website which looks like a company website .
I navigate through the 3 other menus, while also checking the source code  to find any hidden links or clues but nothing good
The ‘JABS’ page seems to be an online store to purchase their ‘finest AI models’ I could add these models into a cart, which is hidden.
The ‘Documentation’ page seemed empty, but has a hidden message (select the whole page or view source code to reveal it) stating that the section is hidden for ‘security reasons’ and in order to view the product documentations, go to /jabcd0cs/.

On navigating to the new url(http://192.168.224.21/jabcd0cs/) we reach the login page of an open source document management system (DMS) named OpenDocMan.
I try to login using the guest/guest credentials as per instructed in the previous documentation page . I was logged in and saw a list of documents. I check out each of the documents and found nothing  relevant.except that there exists another weird user registered in the DMS named min,web who is named as ‘laughing man’.

I navigated through all available linked within the webapp. I try uploading a file, checking it out, checking it back in and then deleting it. I try creating a new user account and login with the new account and do all of the above. I found a profile page section for the user, where i was allowed  to update the name, email, password, etc. One of the form fields include a check-box to mark if the user was an Admin or not. it was disabled though. I tried editing the html form element in developer mode to enable the check-box, checked it and tried to submit,but didn't work instead got redirected to an error page.I even tried using burpsuite ,replacing parameters but all were redirected to an error page.

## exploitation

I decided to search for a vulnerability/exploit based on OpenDocMan,version 1.2.7 and found https://www.exploit-db.com/exploits/32075 exploit at exploit-db.com . This detailed multiple vulnerabilities for the version including SQL injection vulnerability and another vulnerability, where it would be possible to gain administrative access to the application at the point of sign-up.

I tried the url "http://192.168.224.21/jabcd0cs/ajax_udf.php?q=1&add_value=odm_user%20UNION%20SELECT%201,version%28%29,3,4,5,6,7,8,9" (ull find this url in the details when u visit the above exploit)  within the entry on the browser and it worked.

Since SQL injection would work, I decided to use "sqlmap" to enumerate the database. To get a list of databases i ran the below command
-> sqlmap -u "http://192.168.20.21/jabcd0cs/ajax_udf.php?q=1&add_value=odm_user" --batch --dbs --level 3 --risk 3 --threads=5 

upon completion, i got a list of databases that was found and one of them is ‘jabcd0cs’ 

next to dump all the content inside the database "jabcd0cs" i reran sqlmap with the below command:
-> sqlmap -u "http://192.168.20.21/jabcd0cs/ajax_udf.php?q=1&add_value=odm_user" --batch --dbs --level 3 --risk 3 --threads=5 -D jabcd0cs --dump
where -D is used to specify the database name and --dump is used to dump all the content inside the db
The entire database content is dumped and displayed on the screen including the table "odm_users" which contains the list of users with username and password hashes (md5).
with this I came to know that the admin username is "webmin" and also got the hash of the webmin’s password (hash:b78aae356709f8c31118ea613980954b )
i used online tools to decrypt the hash some tools did not work but this one -> "https://www.md5online.org/md5-decrypt.html" worked and gave me the password or the decrypted result 
you can also use kali linux inbuilt tools like hashcat or any other
i finally found the password to be "webmin1980"

### initial-access

after all these stuff i remembered that ssh port was open when i had performed nmap scan
so i just thought of trying the credentials obtained (that is username= webmin and pass = webmin1980) to login to ssh ,use the below command to login to ssh

-> ssh webmin@192.168.224.21 (NOTE: change the ip to your target machines ip) when prompted for password use "webmin1980" as password.
was successfully logged in to the target machine as "webmin"

## Privilege-Escalation
After getting access i just typed the command "id" in the shell and came to know i was not a root user.
i tried listing out all the file using "ls" command but nothing interesting was found 
then to know about the OS and kernel i used the command "uname -a" and found the below Information

->Linux VulnOSv2 3.13.0-24-generic #47-Ubuntu SMP Fri May 2 23:31:42 UTC 2014 i686 athlon i686 GNU/Linux

then i just searched for "Linux 3.13.0-24-generic exploits" on google and found the "overlayfs" exploit  here -> "https://www.exploit-db.com/exploits/37292" which was to be used for privilege escalation
i copied the exploit (or u can even download it directly using "wget" command) and followed the command below
firstly copy the exploit then:-
-> cat > exploit.c 
then press enter and paste the copied exploit then press "ctrl+D" the file will be saved and you'll exit back to shell from cat 
then you need to provide executable permission to the file using chmod ,below is the command
-> chmod +x <filename> in my case the file name is exploit.c so  -> chmod +x exploit.c
then i compiled it using gcc,command below
-> gcc exploit.c -o shell (this command complies the file named shell.c and saves the compiled code or script to the file names shell)
next u need to run the compiled file 
->./<filename> so here it will be "./shell"
this will give you a root shell and you'll be able to do anything on the target machine or lets say you have escalated you privileges by gaining root access

## Post-Exploitation
then i used the command "id" and confirmed that im a root user
then i changed my directory to the root using the command "cd /root"
then listed the files in root directory using the command "ls"
and found the file named flag.txt which our target 
then used "cat flag.txt" and found the flag which had the following details
"You successfully compromised the company "JABC" and the server completely !!
Congratulations !!!
Hope you enjoyed it.

What do you think of A.I.?"

## conclusion
I am aware that there are other approaches for this CTF feel free to try them as well  
In this write-up, we have discussed how to perform a penetration test on a vulnerable machine.
This was a fun challenge and I was able to gain root access and obtain the flag.
I hope this write-up has been helpful in understanding the steps involved in a penetration test.
  

                                  "THANK YOU"


























