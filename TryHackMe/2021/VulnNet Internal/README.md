# VulnNet: Internal

##### Difficulty: [ Easy ]

**Tags:** `Linux`,  `nmap`,  `enum4linux-ng`,  `smb`,  `redis`,  `NFS`,  `rsync`,  `authorized_keys`,  `ssh tunnel`,  `TeamCity`,  `grep`,  `reverse shell`

---

##### Written: 24/11/2021

##### IP address: 10.10.102.151

---

### [ What is the services flag? (services.txt) ]

Let's first conduct an **Nmap** scan on the target machine.

```
sudo nmap -sC -sV -vv -p- -T4 10.10.102.151
```

**Results:**

<img style="float: left;" src="screenshots/screenshot1.png">

We can see that there are many open ports on our target machine, running services like SSH and Samba.

<br>

Let's first take a look at enumerating the **Samba** server running on ports **139** and **445**. I'll use **enum4linux-ng** (https://github.com/cddmp/enum4linux-ng) to run a scan on the server.

*enum4linux-ng is an alternative to enum4linux*

```
/enum4linux-ng/enum4linux-ng.py 10.10.102.151
```

**Results:**

<img style="float: left;" src="screenshots/screenshot2.png">

From the results, we can see that there is a share called '**shares**' that we can access anonymously. This means that we do not need to supply a password in order to access the files within. 

*Alternatively, we could also check the shares that we can access by using `smbclient -L 10.10.102.151 `* 

Let's connect to the share and take a look.

```
smbclient //10.10.102.151/shares
```

<img style="float: left;" src="screenshots/screenshot3.png">

There are two directories in the share: **temp**, **data**

**/temp** contains the **services.txt** file! 

<img style="float: left;" src="screenshots/screenshot6.png">

After downloading the file onto our local machine using the `get` command, we get the first flag:

<img style="float: left;" src="screenshots/screenshot4.png">

---

### [ What is the internal flag? ("internal flag") ]

Let's now take a look at **/data**: 

<img style="float: left;" src="screenshots/screenshot5.png">

There are two files: **data.txt**, **business-req.txt**

**Contents of data.txt:**

<img style="float: left;" src="screenshots/screenshot7.png">

**Contents of business-req.txt:**

<img style="float: left;" src="screenshots/screenshot8.png">

Interesting stuff, but nothing we can really use right now.

<br>

Let's move on to the **Redis** Database server running on port **6379**. We can connect to the DB using `redis-cli`:

```
redis-cli -h 10.10.102.151 -p 6379
```

<img style="float: left;" src="screenshots/screenshot9.png">

After successfully connecting, I tried to dump out all of the keys within the database:

```
// in redis-cli
keys *
```

<img style="float: left;" src="screenshots/screenshot10.png">

Unfortunately, we are unable to as we need to authenticate ourselves first. Hitting this dead-end, we can try enumerating the other services first.

<br>

Now we will try to enumerate **NFS** running on port **2049**. Perhaps there are directories on our target that we can mount onto our local machine?

To check, we can use `showmount`:

```
showmount -e 10.10.102.151
```

<img style="float: left;" src="screenshots/screenshot11.png">

Looks like there is a directory **/opt/conf** that we can mount. We then run the following commands to mount the directory:

```
mkdir /mnt/conf
mount -t nfs 10.10.102.151:/opt/conf /mnt/conf -o nolock
```

*The `-o nolock` option is required for mounting old-style (RH 5.2 or older) NFS servers, as well as other NFS servers that don't support lockd.*

With the directory mounted, we can go ahead and explore it:

<img style="float: left;" src="screenshots/screenshot12.png">

There are a few directories, but what really interested me was the **/redis** directory.

<img style="float: left;" src="screenshots/screenshot13.png">

In it, we find the **redis.conf** configuration file.

<img style="float: left;" src="screenshots/screenshot14.png">

Fortunately for us, the configuration file actually contains the **master password** for the Redis DB: **B65Hx562F@ggAZ@F**

<br>

With that, we can connect back to the Redis server and authenticate ourselves using the `auth` command

<img style="float: left;" src="screenshots/screenshot15.png">

Nice! Using `keys *`, we can see there a few keys in the database:

<img style="float: left;" src="screenshots/screenshot16.png">

We can then obtain the  **internal flag** value using the `get` command:

<img style="float: left;" src="screenshots/screenshot17.png">

---

### [ What is the user flag? (user.txt) ]

Amongst the keys, the '**authlist**' seems the most interesting. Using `type`, we can find out what type of file it is:

<img style="float: left;" src="screenshots/screenshot18.png">

Since this is a list, we have to use the `lrange` command to read it, instead of `get`. (More info on this: https://stackoverflow.com/questions/37953019/wrongtype-operation-against-a-key-holding-the-wrong-kind-of-value-php) 

<img style="float: left;" src="screenshots/screenshot19.png">

We have a **base64-encoded** string that has been repeated a few times.

```
echo 'QXV0aG9yaXphdGlvbiBmb3IgcnN5bmM6Ly9yc3luYy1jb25uZWN0QDEyNy4wLjAuMSB3aXRoIHBhc3N3b3JkIEhjZzNIUDY3QFRXQEJjNzJ2Cg==' | base64 -d
```

<img style="float: left;" src="screenshots/screenshot20.png">

Looks like we have a password for **rsync**.

<br>

I have never heard of rsync before this room, so I did some research.

---

*In general, rsync is a utility for copying and synchronizing files and directories remotely as well as locally in Linux/Unix systems. It can be also be used via SSH and is more optimized than`scp`. More info: https://www.tecmint.com/rsync-local-remote-file-synchronization-commands/*

---

I also found out how we can enumerate this service from: https://book.hacktricks.xyz/pentesting/873-pentesting-rsync

Firstly, we need to enumerate the rsync modules (or shares) that is available.

```
rsync --list-only rsync://10.10.102.151
```

<img style="float: left;" src="screenshots/screenshot21.png">

We have a **files** module.

Next, we transfer all of the files within this **files** module to our local machine. We will also be prompted to input the password that we obtained earlier.

```
rsync -av rsync://rsync-connect@10.10.102.151/files ./rsyn_shared 
```

*-a ensures files are transferred in archive mode, which ensures that symbolic links, devices, attributes, permissions, ownerships, etc. are preserved in the transfer. -v is just for verbosity.*

<img style="float: left;" src="screenshots/screenshot22.png">

<br>

Looking at the files that were transferred, I realized that it was actually the entire home directory of the **sys-internal** user on the target machine!

<img style="float: left;" src="screenshots/screenshot23.png">

We have the **user.txt** file as well, giving us the user flag.

<img style="float: left;" src="screenshots/screenshot24.png">

---

### [ What is the root flag? (root.txt) ]

In the home directory of the user, we can see the **.ssh** directory.

<img style="float: left;" src="screenshots/screenshot25.png">

While it currently does not contain any keys that we can use, one way we can gain a foothold into the machine is by using rsync to upload our own public key into an **'authorized_keys'** file within that directory! From there, our public key will be authorized on the target machine, and we will be able to use our own private key to log into the user account via SSH.

Detailed steps on how to set this up can be found here: https://kb.iu.edu/d/aews 

```
// generate ssh key-pair
ssh-keygen -t rsa

// transfer our public key into an 'authorized_keys' file in the /.ssh folder of the 'sys-internal' user
rsync -av ~/.ssh/id_rsa.pub rsync://rsync-connect@10.10.102.151/files/sys-internal/.ssh/authorized_keys
```

Once this is done, we can log into the **sys-internal** user's account using our private key.

```
ssh sys-internal@10.10.102.151
```

<img style="float: left;" src="screenshots/screenshot26.png">

And we're in!

<br>

Now we need to find a way to escalate our privileges.

I did some manual enumeration, doing things such as searching for files with SUID-bit set and exploring the various directories within the machine. However, there was nothing I could really use.

I did notice an interesting directory located in the **/** directory though:

<img style="float: left;" src="screenshots/screenshot27.png">

From my research, I found out that **TeamCity** is a build management and continuous integration server from JetBrains. The fact that this directory exists could indicate that the machine is running a TeamCity server (internally, which is why our nmap scan did not pick it up).

The default port for TeamCity servers is **8111**. To access it, we can use **SSH port forwarding**.

```
ssh -L 9999:localhost:8111 sys-internal@10.10.102.151
```

This will forward our requests to **localhost:9999** on our local machine, over to port **8111** of the target machine using the sys-internal account.

<br>

Let's now try and visit the TeamCity web page by visiting `http://localhost:9999`

<img style="float: left;" src="screenshots/screenshot28.png">

We have a login page!

I tried common credentials like admin:admin, but those did not work.

Since the web page prompted us to log in as a **Super user**, I clicked on the link, which brought me to this page:

<img style="float: left;" src="screenshots/screenshot29.png">

Interesting! If we have an authentication token, we would be able to log in without needing a username or a password. My first thought was to try and find a working token within the TeamCity directory on the target machine.

There were many files within the TeamCity directory, so let's use `grep` to search for the string **'token'** in all of these files.

```
grep -ir 'token' 2>/dev/null
```

*-i ignores case distinctions and -r is for recursive search*

I ran the command in all of the sub-directories within /TeamCity.

<img style="float: left;" src="screenshots/screenshot30.png">

Eventually, I found tokens within the **/log** sub-directory! The token that allowed me to log in was: **6483908632758184854**

<img style="float: left;" src="screenshots/screenshot31.png">

<br>

Normally with these sort of build management servers, we want to find a point where we can input commands for the server to execute when building. If we are able to achieve remote code execution, we can do things such as opening up a **reverse shell**.

First, let's create a new project.

Next, under **'Build Steps'** on the left sidebar, we can add a new build step and choose **Python** as the runner type.

In the options, under **'Command'**, we choose **'Custom script'** so that we can provide our own code.

<img style="float: left;" src="screenshots/screenshot32.png">

The reverse shell payload we'll use is from: https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md#python

<br>

After saving the configuration, we just need to set up a netcat listener on our local machine and then click '**Run**' on the top-right.

<img style="float: left;" src="screenshots/screenshot33.png">

With that, we've successfully gained access into the machine as root :smile:

<img style="float: left;" src="screenshots/screenshot34.png">

We can find the root flag in the home directory of root.
