# LazyAdmin

#### IP Address: 10.10.26.122

First, I ran a nmap scan on the target machine so as to enumerate more information about the services running.

```bash
sudo nmap -sC -sV -vv -oN nmap_initial 10.10.26.122
```

 The results are as follows:

<img style="float: left;" src="screenshots/screenshot1.png">

As we can see, there are two services running on the target machine: **ssh** and **http**.

Let's visit the HTTP web server.

<img style="float: left;" src="screenshots/screenshot2.png">

Looks like a default Apache2 homepage. Looking at the source code, I could see nothing of interest. Time to use **Gobuster** to run a directory enumeration on the web server:

```bash
gobuster dir -u http://10.10.26.122/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,js
```

I made sure to provide common extensions to check for, such as **.php**.

Gobuster was able to quickly find a hidden directory called **/content**

<img style="float: left;" src="screenshots/screenshot3.png">

Visiting the sub-directory, I am faced with a webpage that seems to run off **SweetRice**, which is a website management system that I've never heard of. There are instructions on how to open the website and there also some links to SweetRice documentation pages.

<img style="float: left;" src="screenshots/screenshot4.png">

I proceeded to run Gobuster on this directory with the same settings as before. The results are as follows:

<img style="float: left;" src="screenshots/screenshot5.png">

As we can see, there are a lot of sub-directories that we can explore. I looked through all of them and the 2 most interesting directories are the **/as** and **/inc** directories.

Going to the **/as** directory, we can actually see a login page!

<img style="float: left;" src="screenshots/screenshot6.png">

Nice, this is probably a login page to an administrator dashboard. Now we just need to find a way to login. I did try some basic **SQL injection** payloads such as **' OR 1=1 --** , but it did not work. I next tried using **sqlmap** to help automate the process but it was unable to find the form located on this page. 

In the **/inc** directory, we can see that this contains many files required for the running of the webpage. One folder that caught my attention was the **mysql_backup/** folder.

<img style="float: left;" src="screenshots/screenshot7.png">

Sure enough, in the folder was a **sql backup file**! This could potentially be extremely useful as it can contain credentials that we can use to log into the dashboard. Furthermore, we can simply view the contents using any text editor.

<img style="float: left;" src="screenshots/screenshot8.png">

I downloaded the backup file and opened it in a text editor. Below shows the contents:

<img style="float: left;" src="screenshots/screenshot9.png">

There's a huge wall of text, but **line 79** reveals something very interesting:

<img style="float: left;" src="screenshots/screenshot10.png">

Awesome! Looks like we have a **username** (manager) and a **hashed password**.

The password seems to be hashed using **md5**. I'll use **John the Ripper** to crack the hash. The command used is

```bash
echo 42f749ade7f9e195bf475f37a44cafcb > hash.txt

john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

**Results:**

<img style="float: left;" src="screenshots/screenshot11.png">

Nice! We got the password. The credentials of the administrator is **manager:Password123**

<img style="float: left;" src="screenshots/screenshot12.png">

Annnnnnd we're in! 

My first thought is to try uploading a malicious file onto the webserver, considering that we have access to all of those directories that contains those files. I'll try using the **php reverse shell** script that I already have stored on my computer.

Looking for a point where I can upload files, I come across the **POST > CREATE** on the sidebar at the left of the dashboard. Clicking on it, I see a **Add File button**. This could be where I can upload my reverse shell script.

<img style="float: left;" src="screenshots/screenshot13.png">

I filled some dummy data into the fields in the form. When I clicked on the **Add File** button, I came to this screen:

<img style="float: left;" src="screenshots/screenshot14.png">

We have a field where we can upload our reverse shell script! We seem to be able to create a new directory as well. Maybe our uploaded file will be placed in the new directory? Let's test this out.

I'll upload the **php-reverse-shell.php** file and create a new directory called **testDirectory**.

After creating the post, there didn't seem to be any errors so now I need to find where this testDirectory folder is on the webserver. Looking back at the results of Gobuster from earlier, I noticed another interesting sub-directory called **attachments**. This could be where the uploaded files are.

<img style="float: left;" src="screenshots/screenshot15.png">

Nice! However, **there were no files inside the testDirectory folder**. My guess is that there was some **file upload restrictions** that have been enforced by the web server. It probably detected that the file I wanted to upload was a php file and blocked it. Looks like I'll need to find some way to bypass these restrictions.

One way that I've learnt to do so is by simply changing the extension to a less common extension, such as **.phtml**. This works against file restrictions which work based on **blacklists**, as the developer might have forgotten to blacklist these uncommon extensions.

I changed the extension to .phtml and tried uploading to the same directory.

<img style="float: left;" src="screenshots/screenshot16.png">

The reverse shell script was successfully uploaded. Now I'll listen for incoming connections using **netcat**:

```bash
nc -lvnp 1234
```

With my netcat listener active, I can then click on the uploaded file to force the web server to execute it. With that, I was able to gain access into the server:

<img style="float: left;" src="screenshots/screenshot17.png">

#### From there, I could traverse to the home directory of the user 'itguy' and find the user flag in the user.txt file

---

Before continuing on with my privilege escalation, I upgraded the simple shell to a fully interactive TTY shell. This can be done with the following command:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

Next, I needed to find a way to escalate my privileges. First, since we discovered a password for the administrator account on SweetRice earlier, I was hoping that the itguy user would reuse this password. Thus, I tried to use **Password123** to log into itguy's ssh account. However, it did not work.

I then decided to use a privilege escalation script called **linpeas** to help automate the process of finding attack vectors. I transferred linpeas  over to the target machine and ran it. Unfortunately, it was not able to detect any potential PE vectors. Looks like I will need to do some manual privesc instead.

Let's first see what binaries I can execute with sudo privileges. This can be done using the command:

```bash
sudo -l
```

**Results**

<img style="float: left;" src="screenshots/screenshot18.png">

Looks like we can use Perl to execute a file called **backup.pl** in itguy's home directory. The **NOPASSWD** also means that we will not be prompted for a password when running the program with sudo, which is great considering that we do not know the password of www-data.

Looking at the contents of backup.pl:

<img style="float: left;" src="screenshots/screenshot19.png">

Looks like it calls another binary called **copy.sh** which can be found in the **/etc** directory. Since www-data does not have write permissions for backup.pl, we cannot directly edit it. Let's take a closer look at **copy.sh** instead.

<img style="float: left;" src="screenshots/screenshot20.png">

Looks like the file tries to delete certain files using the rm command. The important thing to note about this binary is that it is actually writable by www-data!

<img style="float: left;" src="screenshots/screenshot21.png">

I replaced the bash script commands:

```bash
echo /bin/sh > copy.sh
```

Now, when we use perl to execute backup.pl, the copy.sh binary will be executed and **/bin/bash** will spawn a new shell. Since I'll be using **sudo**, the shell spawned should be as **root**. 

 <img style="float: left;" src="screenshots/screenshot22.png">





#### Now that I'm logged in as root, I can access the root flag from root.txt!

