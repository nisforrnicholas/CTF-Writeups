# RootMe

##### Difficulty: [ Easy ]

**Tags:** `Linux`,  `nmap`,  `Gobuster`,  `upload restrictions`,  `reverse shell`,  `python`

---

##### Written: 07/05/2021

##### IP Address: 10.10.128.245

---

First, I ran a basic NMAP scan on the machine (Top 1000 ports) using the command:

```
sudo nmap -sC -sV -vv -oN initial_scan.log 10.10.128.245 
```

From the nmap results, we can see that **2** ports are open: **22 (ssh)** and **80 (http)**:

<img style="float: left;" src="screenshots/screenshot1.png">

<Br>

Looking closer, we can see that the HTTP server is running on **Apache/2.4.29**

Visiting the address in our web browser, we are brought to the following webpage:

<img style="float: left;" src="screenshots/screenshot2.png">

After trying out some low-hanging fruit (eg /admin, /robots.txt, /login...), I decided to use **Gobuster** to enumerate possible directories that the web server is hosting. Gobuster is used as such:

```
gobuster dir -u http://10.10.128.245 -x "php,js" -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt 
```

The **-x** tag is used to indicate any extensions to add on to our attempts. In this case, we can added **.php** and **.js** extensions as they are very commonly used. I also used the wordlist provided by Dirbuster for the enumeration. After some time, I obtained the following results:

<img style="float: left;" src="screenshots/screenshot3.png">

As we can see, there is one very interesting directory called **/panel**. After accessing this directory in our web browser, we can see the following page:

<img style="float: left;" src="screenshots/screenshot4.png">

Looks like we have a place where we can upload files! The first possible step that I thought of was to upload a PHP file which sets up a reverse shell into our machine. 

There is already a PHP reverse shell template file available on Github by pentestmonkey (https://github.com/pentestmonkey/php-reverse-shell). 

After downloading the php reverse shell template, all we have to do is to change the relevant fields within the template, such as the IP address of our local machine. After changing the relevant fields, I then uploaded the reverse shell PHP file into the webserver. After doing so, we can see that it has been successfully uploaded by checking the **/uploads** directory:

<img style="float: left;" src="screenshots/screenshot5.png">

Uh oh. Looks like they do not allow for PHP code to be uploaded. We need to find a way to bypass this file upload restriction.

After doing some research, I came upon this website which had a lot of useful information on bypassing upload restrictions:

https://www.exploit-db.com/docs/english/45074-file-upload-restrictions-bypass.pdf

From here, one way I learnt how web servers handled invalid file extensions was by providing either a **whitelist** or **blacklist**.

*Blacklisting file extensions is when the server bans specific types of file extensions. Whitelisting file extensions is the opposite; specific types of file extensions are allowed.*

One way to bypass blacklists is to change the .php extension to something less common, yet will still allow the file to be run as a PHP file. One extension is the **.phtml** extension. I decided to try changing the file extension to .phtml and uploaded the file onto the server.

<br>

Fortunately, this was all we need to bypass the file restrictions as we can see that the file has been successfully uploaded onto the web server by checking the **/uploads** directory:

<img style="float: left;" src="screenshots/screenshot6.png">

 When we click on this file, the web server will execute it. However, before doing so, we need to make sure to set up a Netcat server that will be waiting for the incoming connection from the web server. This can be done with the following command:

```
nc -lvnp 1234
```

In this case, the port that we wait on is set to **1234**, which we had denoted earlier within the PHP reverse shell script.

<br>

Once the netcat server is set up, we can then execute the uploaded PHP file and gain access!

<img style="float: left;" src="screenshots/screenshot7.png">

With access gained, we now needed to find the **user.txt** file. After manually searching through some directories, I decided to speed up the process by using the ```find``` command. The command is as such:

```
find / -name user.txt
```

The results show that the **user.txt** file is actually located in the **/var/www** directory:

<img style="float: left;" src="screenshots/screenshot8.png">

With that, we have obtained the user flag!

---

Next, we need to find a way to escalate our privileges. There are a few ways to go about this. Normally, I would import the **linpeas** enumeration script to help speed up and automate the process for me. However, the prompts given by the room state to search for files with **SUID bit** set. Hence, I shall do a manual search using the **find** command:

```
find / -perm /u=s
```

From the results, we can see that strangely, the **python** program has its SUID bit set. This means that when we run python, we are actually running as root:

<img style="float: left;" src="screenshots/screenshot9.png">

<br>

To exploit this SUID-set Python binary, I then looked on **GTFOBins** to find out what I could do to break out of the restricted shell and create a new shell as root. After looking through GTFOBins, I came across this method:

<img style="float: left;" src="screenshots/screenshot10.png">

Since we already have an existing SUID binary for python, we can skip the first line and execute the second line in the console.

*After doing some research, I learnt what **os.execl()** does. It executes a new program which replaces the current process. The new executable is loaded into the current process and will have the **same process id as the caller.** In this case, since the caller is **root**, considering that python was run as root, we can then start up a new shell as root.* 

*The arguments to **os.execl()** indicate that we wish to replace the **'/bin/sh'** program with **'sh -p'**. The **-p** tag is used with **sh** to run in privileged mode.*

After running the command, we get a new shell as root!

<img style="float: left;" src="screenshots/screenshot11.png">

<br>

**We can now access the 'root' directory in the system and obtain the root flag.**

