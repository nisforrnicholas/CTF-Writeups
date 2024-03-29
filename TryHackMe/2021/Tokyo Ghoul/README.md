# Tokyo Ghoul

##### Difficulty: [ Medium ]

**Tags:** `Linux`,  `nmap`,  `ftp`,  `steghide`,  `morse code`,  `Gobuster`,  `LFI`,  `john`,  `ssh`,  `escape Python sandbox`

---

##### Written: 03/12/2021

##### IP Address: 10.10.3.234

---

### [ Use nmap to scan all ports ]

Let's start off with a full **Nmap** scan on the target machine. We'll run it with standard scripts (-sC) loaded and version enumeration (-sV) enabled. We'll also run the scan on all ports.

```
sudo nmap -sC -sV -vv -T4 -p- 10.10.3.234
```

---

### [ How many ports are open ? ]

The results of the nmap scan reveals that **3** ports are open on the target machine:

```
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
```

Looks like we have a FTP, SSH and HTTP server running.

---

### [ What is the OS used ? ]

Nmap actually reveals that the HTTP server is running on **Apache**, which uses the **Ubuntu** operating system.

<img style="float: left;" src="screenshots/screenshot1.png">

---

### [ Did you find the note that the others ghouls gave you? where did you find it ? ]

Let's visit the HTTP webpage first.

<img style="float: left;" src="screenshots/screenshot2.png">

At the bottom of the page, there is a link titled "Can you help him escape?". 

Clicking on the link brings us to the subdirectory - **/jasonroom.html**

<img style="float: left;" src="screenshots/screenshot3.png">

If we look at the source code of the page, we can find the hidden note that the other ghouls gave us.

<img style="float: left;" src="screenshots/screenshot4.png">

The note is found at **/jasonroom.html**.

---

### [ What is the key for Rize executable? ]

The note tells us to go to the **FTP** server and look **anonymous**. This is most probably a hint that **anonymous login** is enabled on the FTP server. Anonymous login allows us to log into the server without needing a password, which is great for hackers as they are able to access the files within the server without authenticating.

In fact, Nmap has already revealed that anonymous login is enabled:

<img style="float: left;" src="screenshots/screenshot5.png">

With that, let's go ahead and log into the FTP server.

<img style="float: left;" src="screenshots/screenshot6.png">

The server contains a few directories and files. After exploring all of them, I managed to download 3 files: 

| **Aogiri_tree.txt** (text file) | **rize_and_kaneki.jpg** (image) | **need_to_talk** (executable) |
| ------------------------------- | ------------------------------- | ----------------------------- |

<br>

* **Aogiri_tree.txt**

<img style="float: left;" src="screenshots/screenshot7.png">

Does not seem to contain anything of use right now.

<br>

* **rize_and_kaneki.jpg**

<img style="float: left;" src="screenshots/screenshot8.png">

Oh look, a sweet photo of a loving couple! I'm sure there's nothing messed up with them whatsoever :smirk:

I tried to check the image for any embedded data using **steghide**.

``` 
steghide extract -sf rize_and_kaneki.jpg
```

<img style="float: left;" src="screenshots/screenshot9.png">

Unfortunately, we need a passphrase.

I then used **stegseek** (https://github.com/RickdeJager/stegseek) to carry out a dictionary attack using the rockyou.txt wordlist.

```
stegseek rize_and_kaneki.jpg /usr/share/wordlists/rockyou.txt
```

<img style="float: left;" src="screenshots/screenshot10.png">

No luck there either. Let's move on for now.

<br>

* **need_to_talk**

Running this executable prompts us for another passphrase.

<img style="float: left;" src="screenshots/screenshot11.png">

The passphrase can be found with a simple use of the `strings` command.

```
strings need_to_talk
```

<img style="float: left;" src="screenshots/screenshot12.png">

Let's try inputting '**kamishiro**' into the executable.

<img style="float: left;" src="screenshots/screenshot13.png">

It seems that we have received another passphrase: **You_found_1t**

---

### [ Use a tool to get the other note from Rize . ]

Now we can try to input the newfound passphrase when extracting data from **rize_and_kaneki.jpg** using **steghide**:

<img style="float: left;" src="screenshots/screenshot14.png">

We successfully extracted the embed data into a file called **yougotme.txt**.

<img style="float: left;" src="screenshots/screenshot15.png">

---

### [ What the message mean did you understand it ? what it says? ]

Seems like the message has been encoded in **morse code**.

We can throw the code into **cyberchef** (https://gchq.github.io/CyberChef/) and decode it:

<img style="float: left;" src="screenshots/screenshot16.png">

<br>

The decoded message reveals a bunch of **hexadecimal characters**. Let's go ahead and convert them into ASCII:

<img style="float: left;" src="screenshots/screenshot17.png">

<br>

And we get a base64-encoded message. Let's decode it:

<img style="float: left;" src="screenshots/screenshot18.png">

The hidden message is revealed: **d1r3c70ry_center**

---

### [ Can you see the weakness in the dark ? no ? just search ]

**d1r3c70ry_center** is most probably a subdirectory on the HTTP web server. Let's try visiting it:

<img style="float: left;" src="screenshots/screenshot19.png">



We come to a page that tells us to scan it. Let's try running a **Gobuster** scan to enumerate any hidden subdirectories.

```
gobuster dir -u http://10.10.3.234/d1r3c70ry_center/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt,js -t 50
```

<img style="float: left;" src="screenshots/screenshot20.png">

After some time, Gobuster managed to discover the **/claim** directory. Let's visit it.

<img style="float: left;" src="screenshots/screenshot21.png">

Clicking on either the 'YES' or 'NO' buttons will lead us to the same empty page:

<img style="float: left;" src="screenshots/screenshot22.png">

What immediately caught my attention was the **view** parameter in the url. Could this be susceptible to a **local file inclusion** attack? Let's try to read the **/etc/passwd** file on the machine. We can do so with basic directory traversal:

```
http://10.10.3.234/d1r3c70ry_center/claim/index.php?view=../../../../../../../etc/passwd
```

<img style="float: left;" src="screenshots/screenshot23.png">

Hmmm looks like there is some sort of filtering / checking going on. 

From here, I tried many different payloads *(most from: https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/File%20Inclusion/README.md)* before I found one the worked!

```
http://10.10.3.234/d1r3c70ry_center/claim/index.php?view=%2F%2E%2E%2F%2E%2E%2F%2E%2E%2F%2E%2E%2F%2E%2E%2Fetc%2Fpasswd
```

For the LFI attack to work, we have to encode both the '**/**' and the '**.**' characters.

<img style="float: left;" src="screenshots/screenshot24.png">

---

### [ What did you find something ? crack it ]

The **/etc/passwd** file reveals a user called **kamishiro**. Fortunately for us, his hashed password is also available in the file.

We can use **John the Ripper** to crack the password. We'll first copy the entire last-line into a text file called 'hash'. Then we'll run the following command:

```
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

<img style="float: left;" src="screenshots/screenshot25.png">

With that, we've managed to obtain kamishiro's password: **password123**

---

### [ what is rize username ? ]

Rize's username is **kamishiro**.

---

### [ what is rize password ? ]

Rize's password is **password123**

---

### [ user.txt ]

Let's now log into kamishiro's SSH account.

<img style="float: left;" src="screenshots/screenshot26.png">

<br>

We can then grab the user flag located in the home directory of kamishiro:

<img style="float: left;" src="screenshots/screenshot27.png">

---

### [ root.txt ]

Now we need to find a way to escalate our privileges.

kamishiro's home directory contains an interesting python script called **jail.py** that is owned by **root**. 

<img style="float: left;" src="screenshots/screenshot30.png">

Let's take a look at kamishiro's sudo privileges. We can do so with  `sudo -l`:

<img style="float: left;" src="screenshots/screenshot31.png">

Nice! Looks like kamishiro can run the jail.py script as root. This is most certainly our escalation vector :smile:

Let's now analyze the script.

**Contents of jail.py:**

```python
#! /usr/bin/python3
#-*- coding:utf-8 -*-
def main():
    print("Hi! Welcome to my world kaneki")
    print("========================================================================")
    print("What ? You gonna stand like a chicken ? fight me Kaneki")
    text = input('>>> ')
    for keyword in ['eval', 'exec', 'import', 'open', 'os', 'read', 'system', 'write']:
        if keyword in text:
            print("Do you think i will let you do this ??????")
            return;
    else:
        exec(text)
        print('No Kaneki you are so dead')
if __name__ == "__main__":
    main()
```

jail.py simply prompts the user to input python code, which will then be executed by the script using the exec() function. What's important to note is that the script is placed in a sandbox,  as users are prevented from inputting certain python code like `os` and `import`. 

I found it quite difficult to escape the sandbox as I was unable to import any libraries. This prevented me from using libraries like `subprocess`, which would allow me to run system commands.

After doing some research, I came across this site which provides a few methods on breaking out of the sandbox: 

https://www.reelix.za.net/2021/04/the-craziest-python-sandbox-escape.html

<br>

The website advises to use the following payload:

```
getattr(getattr(__builtins__,'__tropmi__'[::-1])('so'[::-1]),'metsys'[::-1])('whoami');
```

---

**What is getattr()?**

From here: https://stackoverflow.com/questions/4075190/what-is-getattr-exactly-and-how-do-i-use-it

*Objects in Python can have attributes -- data attributes and functions to work with those (methods). For example you have an object `person`, that has several attributes: `name`, `gender`, etc. You access these attributes (be it methods or data objects) usually by writing: `person.name`, `person.gender`, `person.the_method()`, etc.*

*But what if you don't know the attribute's name at the time you write the program? For example you have the attribute's name stored in a variable called `attr_name`.*

*if*

```py
attr_name = 'gender'
```

*then, instead of writing*

```py
gender = person.gender
```

*you can write*

```
gender = getattr(person, attr_name)
```

*One good thing about using getattr() is that it provides error handling as well. For eg, if the 'age' attribute does not exist, we can provide a third argument to getattr(), which will then be returned*

```
>>> getattr(person, 'age', 0)
0
```

---

**What is \__builtins__?**

All of Python's built-in functions: https://docs.python.org/3/library/functions.html

*The builtins module is automatically loaded every time Python interpreter starts. All built-in data type classes such as numbers, string, list etc are defined in this module. The BaseException class, as well as all built-in exceptions, are also defined in it. Further, all built-in functions are also defined in the built-ins module.*

*We can call these built-in functions using the builtins module. For eg:*

```
>>> len('hello')
5
```

*is the same as*

```
>>> import builtins
>>> builtins.len('hello')
5
```

---

Looks like the payload does some sneaky string reversal to bypass the string matching done by the script.  After all, **'\__tropmi__'[::-1]** is essentially the same as **'\__import__'**. We then use the getattr() function to import the `os` library, which is also string-reversed. From there, getattr() is used once again to call the `system()` method in the `os` library, which allows us to run any system commands.

Let's try it out:

<img style="float: left;" src="screenshots/screenshot28.png">

Great! We have achieved code execution outside of the Python sandbox. Now we just alter our payload as such:

```
getattr(getattr(__builtins__,'__tropmi__'[::-1])('so'[::-1]),'metsys'[::-1])('cat /root/root.txt');
```

<img style="float: left;" src="screenshots/screenshot29.png">

With that, we are able to directly read out the root flag from the home directory of the root user.

