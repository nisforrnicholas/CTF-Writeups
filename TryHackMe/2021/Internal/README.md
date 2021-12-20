# Internal

##### Difficulty: [ Hard ]

**Tags:** `Linux`,  `nmap`,  `Gobuster`,  `WordPress`,  `wpscan`,  `reverse shell`,  `ssh`,  `docker`,  `ssh-tunnel`,  `Jenkins`,  `hydra` 

---

##### Written: 25/10/2021

##### IP address: 10.10.43.10

---

### [ Basic Setup ]

Before working on the box, the room requested that we modify our hosts file to reflect **internal.thm**. With that done, we can proceed.

<img style="float: left;" src="screenshots/screenshot1.png">

---

### [ User.txt Flag ]

Let's start off with a basic **Nmap** scan on our target. We'll be running the scan with basic scripts loaded and version enumeration enabled. Since we do not care about how 'noisy' our scan is, we can crank up the aggressiveness using the `-T4` option.

```
sudo nmap -sC -sV -vv -T4 10.10.43.10
```

**Results:**

<img style="float: left;" src="screenshots/screenshot2.png">

As we can see, there are only **two** ports open on the machine:

**Port 22** - SSH server

**Port 80** - HTTP Web server

*Note: I later conducted a full nmap scan on **all ports**. The results were the same.*

<br>

We'll leave the SSH server alone for now. Let's go ahead and take a look at that HTTP web server:

<img style="float: left;" src="screenshots/screenshot3.png">

Looks like we have a basic Apache2 default webpage. We can try using **Gobuster** to enumerate any directories that exist on this web server. I'll be using Dirbuster's medium directory wordlist. I'll also make sure to check for files such as .php and .txt files.

``` 
gobuster dir -u http://internal.thm/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,css,txt -t 100 
```

**Results:**

<img style="float: left;" src="screenshots/screenshot4.png">

From the results, we can see a couple of directories on the web server. The directories that really interest me are **/blog** and **/phpmyadmin**. From the **/wordpress** directory, we can also tell that the WordPress service is being used.

<br>

Let's go ahead and navigate to the interesting directories:

**/blog**

<img style="float: left;" src="screenshots/screenshot5.png">

As suspected, we have a WordPress website! We'll run another **Gobuster** scan on **internal.thm/blog/** so as to enumerate any hidden directories. In the meantime, we can take a look at **/phpmyadmin**.

<br>

**/phpmyadmin**

<img style="float: left;" src="screenshots/screenshot6.png">

Interesting! We have a login page that could potentially be susceptible to credential brute-forcing. However, before we go down this rabbit hole, let us first take a look at our Gobuster scan for the /blog directory. *(Note: In the end, the /phpadmin directory was not needed to solve the room.)*

<img style="float: left;" src="screenshots/screenshot7.png">

After looking through all of the directories, the **/wp-login.php** stands out the most as it actually brings us to a WordPress login page:

<img style="float: left;" src="screenshots/screenshot8.png">

We can actually use a nifty tool called **WPScan** (https://github.com/wpscanteam/wpscan), which is a vulnerability scanner built specifically for WordPress websites. One of the features of WPScan is the ability to run a dictionary attack on the login page, which will hopefully enumerate a set of credentials that we can use to log into the WordPress dashboard.

Before we do that, we first need to guess what username the WordPress Administrator could be using. Fortunately for us, the error message when logging in with incorrect sets of credentials actually differs depending on whether that account exists or not.

**Username -> test**

<img style="float: left;" src="screenshots/screenshot9.png">

<br>

**username -> admin**

<img style="float: left;" src="screenshots/screenshot10.png">

Hence, from the different error messages, we can deduce that **admin** is a valid username and there is currently an account associated with it. This is most likely the administrator's account username.

---

*Alternatively, if we look at the **/readme.html** directory, we also find out that the username: **admin** is the default username for the WordPress dashboard.*

<img style="float: left;" src="screenshots/screenshot11.png">

<br>

---

Now that we have a username, let's go ahead and run WPScan. We will be using the Rockyou password wordlist.

```
wpscan --url http://internal.thm/blog/ -U admin -P /usr/share/wordlists/rockyou.txt
```

**Results:**

<img style="float: left;" src="screenshots/screenshot12.png">

Nice! We have managed to obtain the password to the administrator account: **my2boys**.

<br>

With that, let's log into the WordPress dashboard.

<img style="float: left;" src="screenshots/screenshot13.png">

<br>

The first thing we can do is to look at the **Posts** that the admin has uploaded previously.

<img style="float: left;" src="screenshots/screenshot14.png">

<img style="float: left;" src="screenshots/screenshot15.png">

Looks like there is a private post created which actually contains a set of credentials - **william:arnold147**

I tried to use these credentials to log into the WordPress dashboard, the myphpadmin login page as well as the SSH server. Unfortunately, they all did not work. Hitting a dead end, I decided to note these credentials down and move on.

<br>

From WPScan's results earlier on, I know that the current WordPress service is running off version **5.4.2**. Unfortunately, I was unable to find any exploit that could be used for our particular scenario in ExploitDB.

I then did some further research online on how we can exploit the WordPress service, eventually coming across this website:

https://pentaroot.com/exploit-wordpress-backdoor-theme-pages/

<br>

From the WordPress dashboard and WPScan, we know that the website is running a theme called **Twenty Seventeen**. The use of such themes is a potential attack vector as we can actually leverage the **Theme Editor** to upload some malicious code onto the web server! In our case, we will be uploading a **PHP reverse shell**.

We first navigate to **Appearance** > **Theme Editor**.

<img style="float: left;" src="screenshots/screenshot16.png">

Next, we have to find a theme file which we can edit. One file that is commonly used is the **archive.php** file. 

<img style="float: left;" src="screenshots/screenshot17.png">

All we have to do now is to delete all of the currently existing code and replace it with our own reverse shell code. We shall use the PHP Reverse Shell code provided by **pentestmonkey** (https://github.com/pentestmonkey/php-reverse-shell).

**Replaced code:**

<img style="float: left;" src="screenshots/screenshot18.png">

With the code replaced, we click on **Update File** to save it.

Lastly, with a netcat listener up and running, we can then open the reverse shell by navigating to:

http://internal.thm/blog/wp-content/themes/twentyseventeen/archive.php

<br>

<img style="float: left;" src="screenshots/screenshot19.png">

And we're in! :smiling_imp:

<br>

<img style="float: left;" src="screenshots/screenshot20.png">

There is a user called **aubreanna**, but we are unable to access her home directory. Looks like we'll need to do some manual enumeration.

After searching through some of the common directories within the machine, I eventually found an interesting file in **/opt**:

<img style="float: left;" src="screenshots/screenshot21.png">

Great! We now have aubreanna's password - **bubb13guM!@#123**

<br>

This should be her credentials to her account on the machine. Let's try to SSH into her account.

<img style="float: left;" src="screenshots/screenshot22.png">

With that, we are able to access aubreanna's home directory and obtain the **user flag**.

---

### [ Root.txt Flag ]

Apart from the user flag, there is also a text file called **jenkins.txt**.

<img style="float: left;" src="screenshots/screenshot23.png">

It seems as if there is a Jenkins service running on a Docker container hosted by our target. We know this as the welcome message when we log into SSH actually reveals the IP address for docker0, which is 172.17.0.1.

<img style="float: left;" src="screenshots/screenshot34.png">

Of course, we would not be able to directly access the container from our local machine, as the IP address **172.17.0.2** is an internal IP address. Instead, we can use an **SSH reverse tunnel** to grant us access through the target machine.

On our local machine, we run the following command:

```
ssh -L 9999:172.17.0.2:8080 aubreanna@10.10.43.10
```

This will set up a local SSH Reverse Tunnel, where all traffic to port 9999 of our local machine will be forwarded to port 8080 of 172.17.0.2, which can be reached via our target machine using aubreanna's account.

<br>

Once the tunnel is established, we then navigate to **localhost:9999** in our web browser.

<img style="float: left;" src="screenshots/screenshot24.png">

And we have a Jenkins login page! We will use **Hydra** to attempt to brute-force a working set of credentials. Let's try using **admin** as the username again.

Before running Hydra, we first need to find out some key information in the request made when we try to log into Jenkins. We can use the in-built network inspector in our web browsers to do so *(ctrl-shift-i on Firefox and Chrome)*.

<img style="float: left;" src="screenshots/screenshot25.png">

From the request headers, we know that the request is a **POST** request. This means that we will use **http-post-form** as the method in Hydra. We also see that the request is directed towards **/j_acegi_security_check**.

<img style="float: left;" src="screenshots/screenshot26.png">

Next, from the form data of the request, we see that four parameters are sent: **j_username**, **j_password**, **from** and **Submit**.

<img style="float: left;" src="screenshots/screenshot27.png">

Lastly, we see that the error message upon providing incorrect credentials is: **Invalid username or password**

<br>

 With these information, we can craft out our Hydra command:

```
hydra -s 9999 -l admin -P /usr/share/wordlists/rockyou.txt 127.0.0.1 http-post-form "/j_acegi_security_check:j_username=^USER^&j_password=^PASS^&from=%2F&Submit=Sign+in:Invalid username or password" -V
```

**Results:**

<img style="float: left;" src="screenshots/screenshot28.png">

Nice! We managed to obtain the password to the admin account - **spongebob**. Now we can log in.

<br>

There are numerous ways to exploit Jenkins, many of them resulting in remote code execution. An easy way is to use the **Script Console** tool. This tool can be accessed via **Manage Jenkins** > **Script Console** from the left-hand sidebar.

<img style="float: left;" src="screenshots/screenshot29.png">

With this tool, we can run Groovy code on the server. This means that we can set up a reverse shell yet again, granting us access into the Docker container running the Jenkins service.

We shall use the following payload: https://gist.github.com/frohoff/fed1ffaab9b9beeb1c76

<img style="float: left;" src="screenshots/screenshot30.png">

With our netcat listener up and running, we run the code and gain access into the container running the Jenkins server.

<img style="float: left;" src="screenshots/screenshot31.png">

<br>

Next, we can conduct some manual enumeration, starting off by checking some of the common directories where users can store files and other useful information. Since I managed to find user credentials in the **/opt** directory earlier on, I figured that would be the first place I would look.

<img style="float: left;" src="screenshots/screenshot32.png">

Luckily for us, it seems as if there is a **note.txt** file located in the /opt directory! :sweat_smile: The file actually contains the credentials to access the **root** account on our target machine.

<img style="float: left;" src="screenshots/screenshot33.png">

With that, we are able to log into the machine as root and obtain the **root flag**.



