# Bounty Hunter

#### IP Address: 10.10.213.55

<br>

**Deploy the machine.**

Done.

<br>

**Find open ports on the machine.**

I ran a basic **nmap** scan with the **-sC**, **-sV** and **-vv** tags set. For initial results, I first ran the scan against the top 1000 ports. After the results came through, I ran the scan again, this time against ALL of the ports.

Initial Nmap scan results:

<img style="float: left;" src="screenshots/screenshot1.png">

It seems like the machine is running **ftp**, with anonymous-login enabled, **ssh** as well as a **website**! Navigating to the website, I saw this:

<img style="float: left;" src="screenshots/screenshot2.png">

<br>

**Who wrote the task list?** 

I decided to first work on the website. Looking at the source code revealed nothing. I then tried to run a **gobuster** directory brute-force attack on the site, using the dirbuster's medium wordlist. 

While gobuster was running, I tried the low-hanging fruit, like "/robots.txt" and "/login". Unfortunately, they did not work.

I then decided to try connecting to the ftp server. Since anonymous login is enabled, I should be able to log in with the username '**anonymous**'.

<img style="float: left;" src="screenshots/screenshot3.png">

And I'm in! :smiling_imp:

<br>

<img style="float: left;" src="screenshots/screenshot4.png">

We can see that the ftp server holds the '**task.txt**' in file in question. Let's download the file to our local machine using the command '**get**'. Looking at the file gives us the answer to this task:

 <img style="float: left;" src="screenshots/screenshot5.png">





**Lin** wrote the task list.

<br>

**What service can you bruteforce with the text file found?**

Looking at the other downloaded text file, it seems like a wordlist for passwords! Since I was unable to find a login page for the website *(Gobuster has revealed nothing as of now)*, the only other service that we can try brute-forcing with this wordlist is **ssh**.

<img style="float: left;" src="screenshots/screenshot6.png">

<br>

 **What is the users password?** 

I will be using **Hydra** to carry out the brute-forcing. Since we have not encountered any other possible usernames, the username that I will try will be '**lin**'.

<img style="float: left;" src="screenshots/screenshot7.png">

Lin's password: **RedDr4gonSynd1cat3**

<br>

**Obtain user.txt**

Now, we log into the ssh server as lin.

<img style="float: left;" src="screenshots/screenshot8.png">

With that, we can obtain the first flag!

<img style="float: left;" src="screenshots/screenshot9.png">

<br>

**Obtain Root.txt**

Let's do some basic privesc enumeration. First, I check the sudo privileges on lin's account.

<img style="float: left;" src="screenshots/screenshot10.png">

Interesting! We can run **/bin/tar** as root. This could be something that’s exploitable. Let's take a look at this file.

Using the '**file**' command, we can see that this '**tar**' file is an executable:

<img style="float: left;" src="screenshots/screenshot11.png">

We can then look at **gtfobins.com** to find out if there are ways we can exploit this file! 

<img style="float: left;" src="screenshots/screenshot12.png">

---

**EXTRA NOTE:**

Instead of just copy and pasting the command from GTFOBINS, I wanted to understand how it works. The command is as follows:

```
tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

The **-cf** tag is just the standard tag for creating an archive file from the input files. Hence, if we run **tar -cf archive.tar foo** for example, we are just creating an archive file called **archive.tar** from file **foo.** In our case, we are creating a file written to /dev/null, from a file /dev/null. In this case, since anything written to /dev/null is basically removed from the system, we are creating a non-existent archive from a non-existent file. The reason we have to do this is because in order for tar to run, it needs to have an output archive file! Hence, if we don’t want to actually create a new file, we can just write this output to /dev/null.

<br>

The real exploit comes from the **--checkpoint** tag. 

<img style="float: left;" src="screenshots/screenshot13.png">

Hence, while creating the non-existent file, it will run the checkpoint action. This is where we can inject our commands in! In this case, we simply inject **/bin/sh** to open up a new shell.

<br>

---

Bingo! Seems that if we run the above command, we can spawn an interactive system shell. Since we have sudo access, we can spawn a shell as root.

To test this command, we try running it normally. We can see a new shell is indeed spawned, and we are logged in as lin.

<img style="float: left;" src="screenshots/screenshot14.png">

Now running the command with '**sudo**', we are logged in as root!

<img style="float: left;" src="screenshots/screenshot15.png">

With that, we can access root's home directory and obtain the final flag.

<img style="float: left;" src="screenshots/screenshot15.png">

