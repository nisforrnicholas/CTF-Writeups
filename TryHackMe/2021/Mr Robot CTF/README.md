# Mr Robot CTF

##### Difficulty: [ Medium ]

**Tags:** `Linux`,  `nmap`,  `robots.txt`,  `WordPress`,  `wpscan`,  `reverse shell`,  `john`,  `nmap`

---

##### Written: 03/11/2021

##### IP Address: 10.10.7.125

---

### [ What is key 1? ]

Let's start off with an **Nmap** scan on the target machine. We run the scan with standard scripts loaded and version enumeration enabled. We also do a full scan on all ports.

```
sudo nmap -sC -sV -vv -T4 -p- 10.10.7.125
```

**Results:**

<img style="float: left;" src="screenshots/screenshot1.png">

From the results, we can see that there are **two** ports discovered on our target:

**Port 22**: SSH server (closed)

**Port 80**: HTTP Web Server

<br>

Let's first take a look at the HTTP web page:

<img style="float: left;" src="screenshots/screenshot2.png">

We have a pretty cool web page that allows us to input some preset commands: **prepare**, **fsociety**, **inform**, **question**, **wakeup**, **join**.

Inputting any of the commands will bring us to different pages which look cool as well, but ultimately are of no use to us.

**Eg: inputting 'inform'**

<img style="float: left;" src="screenshots/screenshot4.png">

<br>

Next, I did some manual enumeration of the site, starting off with some low-hanging fruit.

Firstly, I checked the source code for the main page:

<img style="float: left;" src="screenshots/screenshot5.png">

We have a hidden message: **YOU ARE NOT ALONE**. We also have a '**USER_IP**' variable pointing to '**208.185.115.6**'. Not sure if this will be useful in the future, let's keep note of it for now.

<br>

Next, let's see if we can access the **robots.txt** file on the web server.

<img style="float: left;" src="screenshots/screenshot6.png">

Nice, we found the first key! We can access it by visiting **http://10.10.7.125/key-1-of-3.txt**:

<img style="float: left;" src="screenshots/screenshot7.png">

---

### [ What is key 2? ]

Visiting http://10.10.7.125/fsocity.dic prompts us to download a file called **fsocity.dic**.

**Snippet of fsocity.dic contents:**

<img style="float: left;" src="screenshots/screenshot8.png">

Looks like we have a username / password wordlist! This will come in useful later, but for now let's continue to enumerate the web server.

<br>

Now, let's run a **Gobuster** scan to find any hidden directories on the web server. We make sure to check for common extensions such as .php and .txt. 

```
gobuster dir -u http://10.10.7.125/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html -t 100
```

**Results:**

<img style="float: left;" src="screenshots/screenshot9.png">

From the results, we can tell that the web server is using **WordPress**. There are also two notable directories: **/admin** and **/login**

When we visit **/admin**, we get a blank screen that seems to be constantly refreshing and loading. It does not seem that we can interact with this page for now.

Next, visiting **/login** brings us to the WordPress dashboard login page!

<img style="float: left;" src="screenshots/screenshot10.png">

I tried using common credentials to log in, such as **admin:admin**. Unfortunately, that did not work. This means that we will have to carry out a dictionary attack. To do that, we first need to figure out an existing username.

The good thing about WordPress is that it actually tells us if a username exists or not. For example, if we were to input a non-existing username, we would get the error message: **"Invalid username."** On the other hand, if we were to input an existing username, we would get the error message: **"The password you entered for the username XXX is incorrect."**  

With this in mind, I tried various usernames that were found within the website: **mrrobot**, **friend**, **fsociety**, **root** etc. Unfortunately, none of them worked.

<br>

Perhaps the **fsocity.dic** file contains the username? Let's use **WPScan** to try enumerating the username with that wordlist.

```
wpscan --url http://10.10.7.125 -e u --users-list fsocity.dic -t 50
```

**Results:**

<img style="float: left;" src="screenshots/screenshot11.png">

We found some usernames that we can use!  => **Elliot**, **elliot**, **ELLIOT**. 

I'll use **'elliot'** as the username for my dictionary attack. Normally, my go-to password wordlist for dictionary attacks is **rockyou.txt**. However, I wanted to check whether **fsocity.dic** contained the password first.

<br>

Once again, we can use WPScan to carry out the dictionary attack. However, before doing so, note that the wordlist contains 858160 lines! That's a lot of words and will definitely slow down our attack. Furthermore, many of the words are duplicated, so we can surely optimize the wordlist by removing the duplicates.

We can use the `sort` program with the `-u` option to do so.

```
sort -u fsocity.dic > fsocity_sorted.dic
```

**Before sorting:**

<img style="float: left;" src="screenshots/screenshot12.png">

**After sorting:**

<img style="float: left;" src="screenshots/screenshot13.png">

We then use WPScan like so:

```
wpscan --url http://10.10.7.125 -U elliot -P /usr/share/wordlists/rockyou.txt -o wpscan_password_result
```

**Results:**

<img style="float: left;" src="screenshots/screenshot14.png">

Great! We have elliot's password: **ER28-0652**

<br>

With that, let's log into the WordPress dashboard.

<img style="float: left;" src="screenshots/screenshot15.png">

Looks like we have version **4.3.1** of WordPress running with the **Twenty Fifteen theme**. From past experiences, I know that we can exploit the **Theme Editor** to upload a PHP reverse shell. This will then allow us to gain an initial foothold into our target machine.

The Theme Editor can be accessed via **Appearance** > **Editor**.

We'll be using the PHP Reverse Shell script by pentestmonkey (https://github.com/pentestmonkey/php-reverse-shell).

We copy the entire script and replace one of the .php files in the Theme Editor. For our case, I'll replace **archive.php**.

**archive.php before replacing:**

<img style="float: left;" src="screenshots/screenshot16.png">

**archive.php after replacing:**

<img style="float: left;" src="screenshots/screenshot18.png">

After saving the file, we then run a netcat listener that is ready to catch our reverse shell connection.

Finally, we navigate to: http://10.10.7.125/wp-content/themes/twentyfifteen/archive.php

<img style="float: left;" src="screenshots/screenshot17.png">

With that, we managed to open up a shell into our target machine! :smile:

<br>

Searching around, I found the second flag in the **home** directory of the user '**robot**'. 

<img style="float: left;" src="screenshots/screenshot19.png">

Unfortunately, we do not have the permissions to open the file. Looks like we need to find a way to log in as the user 'robot'.

In the same directory, there is a **password.raw-md5** file:

<img style="float: left;" src="screenshots/screenshot20.png">

This seems to be the md5-hashed password for 'robot'! We can use **John the Ripper** with the rockyou.txt wordlist to crack the password.

```
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=Raw-MD5
```

**Results:**

<img style="float: left;" src="screenshots/screenshot21.png">

And we have robot's password: **abcdefghijklmnopqrstuvwxyz**

<br>

When trying to log in as root in our reverse shell, I realized that I couldn't as I was in a **non-interactive** shell. Hence, programs that require user input, such as `su`, will not work. 

I then tried to ssh directly into the machine, but could not as the SSH port was closed.

<img style="float: left;" src="screenshots/screenshot22.png">

One way that we can upgrade our shell to a fully-interactive shell is using **Python**. The command is:

```
python -c 'import pty; pty.spawn("/bin/bash")'
```

<img style="float: left;" src="screenshots/screenshot23.png">

With our shell upgraded, we can su into robot's account and access the second flag.

<img style="float: left;" src="screenshots/screenshot24.png">

---

### [ What is key 3? ]

We now have to find a way to escalate our privileges.

I first did some manual enumeration by searching around the various directories on the machine, such as /opt and /tmp. However, I was unable to find anything of interest. Next, I checked for robot's sudo privileges, but turns out that they had none.

<img style="float: left;" src="screenshots/screenshot26.png">

Next, let's look for SUID-bit enabled files on the system.

```
find / -type f -perm /u=s 2>/dev/null
```

<img style="float: left;" src="screenshots/screenshot25.png">

Immediately I noticed an interesting file with the SUID bit set: **nmap**!

*(Instead of using `find`, we could have also automated the process by using a privesc automation script such as linpeas)*

<br>

Doing some research online, I found an article online on how we can exploit an SUID-bit enabled nmap: https://null-byte.wonderhowto.com/how-to/use-misconfigured-suid-bit-escalate-privileges-get-root-0173929/

To do so, we first run nmap in interactive mode.

```
./nmap --interactive
```

Then, we break out of nmap by spawning a shell.

```
!sh
```

<img style="float: left;" src="screenshots/screenshot27.png">

Since the program is run as root, the shell that we spawn will have root privileges. We have managed to escalate our privileges! 

<img style="float: left;" src="screenshots/screenshot28.png">

With that, we can obtain the third flag which is located in the root home directory.
