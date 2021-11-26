# Blog

##### Written: 26/11/2021

##### IP address: 10.10.90.101

*The ordering of the prompts in this room was kinda weird, so I reordered them.*

---

### [ What CMS was Billy using? ]

Before doing anything, we first add **blog.thm** to our **/etc/hosts** file.

<img style="float: left;" src="screenshots/screenshot1.png">

<br>

Let's conduct an **Nmap** scan on the target machine.

```
sudo nmap -sC -sV -vv -p- -T4 blog.thm
```

**Results:**

<img style="float: left;" src="screenshots/screenshot2.png">

From the results, we can see that **4** ports are open:

* **Port 22**: SSH Server
* **Port 80**: HTTP server
* **Ports 139 & 445**: Samba Server

<br>

Let's visit the HTTP Web server:

<img style="float: left;" src="screenshots/screenshot3.png">

<img style="float: left;" src="screenshots/screenshot7.png">

From the bottom of the page, we can see that the CMS Billy is using is **WordPress**.

---

### [ What version of the above CMS was being used? ]

Let's use **WPScan** to find out the version of WordPress that Billy is running.

```
wpscan --url http://blog.thm
```

<img style="float: left;" src="screenshots/screenshot8.png">

WordPress version: **5.0**

---

### [ user.txt ]

 Let's a closer look at the posts on the website.

**'A Note From Mom' by Karen Wheeler:**

<img style="float: left;" src="screenshots/screenshot4.png">

<br>

**'Welcome!' by Billy Joel**

<img style="float: left;" src="screenshots/screenshot5.png">

<br>

There wasn't anything particularly interesting within the posts, but at least we know that there are two users who have authorship over the website.

Since this is a WordPress website, there will most probably be a login page at **/wp-admin**.

<img style="float: left;" src="screenshots/screenshot6.png">

Nice!

The good thing about WordPress for hackers is that it actually exposes if a user exists on the server through the login form. If we try to login with a non-existent user, it will display the error message: '**Invalid username**'

Meanwhile, if we log in with a user that exists, we will get a different error message instead: '**The password you entered for the username XXX is incorrect.**'

We can use this to find out the username for Billy Joel and Karen Wheeler. 

I tried some potential usernames, like billy, karen, billyjoel, karenwheeler... Unfortunately, none of those worked.

<br>

Let's try using **WPScan** to enumerate some usernames instead:

```
wpscan --url http://blog.thm -e u
```

<img style="float: left;" src="screenshots/screenshot9.png">

WPScan managed to find two usernames: **kwheel**, **bjoel**

<br>

Next, let's check if there are any vulnerabilities for this version of WordPress. We'll check in **Metasploit**.

<img style="float: left;" src="screenshots/screenshot10.png">

Let's use the first one: **WordPress Crop-image Shell Upload**

Now let's take a look at the options of the exploit:

<img style="float: left;" src="screenshots/screenshot11.png">

We see that we actually need a valid **username** and **password** in order for the exploit to work.

<br>

We can use **WPScan** to run a dictionary attack on the two users that we enumerated earlier.

I first ran the attack on the user: **bjoel**. However, it was taking an extremely long time and I started to suspect that bjoel's password was not in the Rockyou.txt wordlist.

Next I ran the attack on the user: **kwheel**. Fortunately, I managed to obtain her password after a few minutes!

```
wpscan --url http://blog.thm -U kwheel -P /usr/share/wordlists/rockyou.txt -t 50
```

<img style="float: left;" src="screenshots/screenshot12.png">

**kwheel's** password: **cutiepie1**

<br>

With a valid set of credentials, I updated the options for the exploit and ran it.

<img style="float: left;" src="screenshots/screenshot13.png">

<img style="float: left;" src="screenshots/screenshot14.png">

We're in!

<br>

Looking around the machine, I found the **home** directory of Billy.

<img style="float: left;" src="screenshots/screenshot16.png">

However, the user flag was not actually in the **user.txt** file:

<img style="float: left;" src="screenshots/screenshot17.png">

<br>

Let's try searching for the real user.txt file. We first have to open a shell using the command `shell` in Meterpreter. Then, we run:

```
find / -iname user.txt 2>/dev/null
```

<img style="float: left;" src="screenshots/screenshot23.png">

The real user.txt file is in **/media/usb**.

<img style="float: left;" src="screenshots/screenshot25.png">

However, it seems that we do not have the necessary permissions to enter the directory. Guess we have to escalate our privileges if we wish to obtain the flag.

<br>

My first thought was to look for files with the **SUID**-bit set.

```
find / -type f -perm /u=s 2>/dev/null
```

<img style="float: left;" src="screenshots/screenshot19.png">

Looking through the results, I noticed an interesting binary called **checker** located in **/usr/sbin**. Let's try running it!

<img style="float: left;" src="screenshots/screenshot20.png">

Hmm... It simply outputs '**Not an Admin**' before closing. What if we use `ltrace` to trace the library calls that are called by the binary?

```
ltrace checker
```

<img style="float: left;" src="screenshots/screenshot21.png">

Interesting, it seems that it checks for the environment variable '**admin**'. Since it is currently not set, the binary then puts the 'Not an Admin' string onto the console. Let's try to set the environment variable!

```
export admin=yes // you can replace 'yes' with anything
```

<img style="float: left;" src="screenshots/screenshot22.png">

Great, **checker** actually opened up a privileged shell! We are now root :smile:

<br>

With that, we will be able to obtain the user flag.

<img style="float: left;" src="screenshots/screenshot24.png">

---

### [ Where was user.txt found? ]

The **user.txt** file was found in **/media/usb**

---

### [ root.txt ]

**root.txt** can be found in the root home directory.

<img style="float: left;" src="screenshots/screenshot26.png">











### 



### 



### 
