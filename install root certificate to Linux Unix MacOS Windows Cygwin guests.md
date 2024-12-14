# [install root certificate to Linux Unix MacOS Windows Cygwin guests.md](install%20root%20certificate%20to%20Linux%20Unix%20MacOS%20Windows%20Cygwin%20guests.md)
(last update 2024-10-20)

**Notes:**

Linux requires a PEM formatted certificate that is named with a .crt file extension. Export in Base-64 format which Windows will name with a .cer extension. It's all the same. If you see `-----BEGIN CERTIFICATE-----` at the top of the cert then it is in PEM format. Just rename it.

We are working with standard .pem files. There is an extended **trusted** format that has -----BEGIN TRUSTED CERTIFICATE----- at the top, but that's not the one we're using here. We are only working with the basic certs that have -----BEGIN CERTIFICATE----- at the top.

One thing I'm not touching on here is certificate chains. Many public certificate services have intermediate certs, and that means you need a complete chain. The complete chain needs to have all the intermediate and main CA certs in the right order and saved to your root cert file. There are many ways to combine chains and convert formats but that's outside our simple scope here. The Linux, Unix, Mac, etc. cert instructions below assume you already have a valid chain if you need that.

Finally, we are installing root certificates for the OS to use. Most web browsers have their own set of root certificates and will not use the cert we're installing.

---

### Import and install the root cert - see instructions for your OS listed below
Once you have your cert you should test that it's formatted properly and has valid data. This command works on all OSes except Windows. Do this:   
`openssl x509 -noout -text -in your_cert_name.pem`

We'll start with MacOS, work through a few flavors (distros) of Linux, then a few flavors of Unix, Windows, and end with Cygwin (because we can). Mac is up first because it is Unix, but it works in its own Apple-ish way.

#### MacOS (OSX) 12, 13, and 14 (probably 15 too)
You will need sudo priveleges. That is the default for the first user account you create. If you can change system settings that require your password then you have sudo priveleges. These instructions are really intended for professional systems people. Adding a root cert has huge security implications, so be careful.

First we'll look at the full command and explain the options. Then we'll cover the actual steps to implement this change.

We're using the MacOS *security* command. It does a lot, and most if it is outside the scope of this document. Our focus is adding a trusted root certificate. Alternatively, this can be done using the Mac System Settings, but it's awkward. Our method goes right to the task at hand without a lot of clicking around.

You may not use an SSH session because there is a sudo verification pop-up window. You must be logged onto the desktop.

Here's the full command:  
`sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain your_cert_name.pem`
- **add-trusted-cert** this is the sub-function of the *security* command that we are using
- **-d** tells it to add the cert to admin store
- **-r trustRoot** tells the command how to handle the request
- **-k /Library/Keychains/System.keychain** tells it to add the root cert to the **System** keychain, very important location
- and finally the cert name

And here are the steps:
- log onto your MacOS desktop with an account that has admin/sudo privileges
- copy the cert file (text format, Base-64) to a convenient place in your file system like Desktop, Downloads, Documents, etc.
- open a *Terminal.app* prompt (It's in Applications/Utilities)
- run the commmand as it appears above replacing the last parameter with your actual cert name, the file extension doesn't matter

#### Red Hat 7, 8, and 9 (CentOS, Rocky, Alma, etc.)
Read the file /etc/pki/ca-trust/source/README for pretty much these exact instructions.

- copy the cert file (text format, Base-64, **.pem** extension) to /etc/pki/ca-trust/source/anchors
- run the command  
`update-ca-trust`

#### Ubuntu 20, 22, and 24 / Debian 11 and 12
Debian and Ubuntu, and all their derivatives require the certificate file to have a **.crt** file extension.

- copy the cert file (text format, Base-64, **.crt** extension) to /usr/local/share/ca-certificates
- run the command  
`update-ca-certificates`

#### SUSE 15.x (all service packs)
- copy the cert file (text format, Base-64, **.pem** extension) to /etc/pki/trust/anchors
- run the command  
`update-ca-certificates`

The update command runs silently with no feedback showing success.

#### FreeBSD 13.x (should be good for 12 as well)
This one is a bit convoluted so pay attention to all the steps.

- copy the cert file (text format, Base-64, **.pem** extension) to /usr/share/certs/trusted
- set the owner and group to root:wheel
- set the file to read only for user, group, and other
- run `openssl x509 -noout -hash -in your_cert_name.pem` and get the hash
- change directory to /etc/ssl/certs
- create a symbolic link in this exact format which is the **your_cert_hash.0** like this:  
`ln -s ../../../usr/share/certs/trusted/your_cert_name.pem your_cert_hash.0`

The part of the link command that is ../../../ matches the links that I found on my servers. I don't know why they chose to make a relative link like that, but that's what I found so that's what I did.

My lab symlink command was:  
`ln -s ../../../usr/share/certs/trusted/labworld.example.com-rootCA.pem cf329f4b.0`

The result looked like this:  
`lrwxr-xr-x  1 root  wheel  69 Oct 18 15:18 cf329f4b.0@ -> ../../../usr/share/certs/trusted/labworld.example.com-rootCA.pem`

#### OpenBSD 7.5
This one is simple and crude, but it is destructive so backup up your cert.pem file before you do anything else.

If you look in the cert.pem file you will see many certs along with their details like signature type and some hash values. I couldn't find a simple way to extract only those details from my input cert. -textout gives much of the same information but omits the hashes and includes the moduli. Doesn't really matter. Those additional details are all optional. All you need is the cert info itself.

- obtain the cert file (text format, Base-64, **.pem** extension)
- append your CA cert to the end of the /etc/ssl/cert.pem file like this:  
`cat your_cert_name.pem >> /etc/ssl/cert.pem`

#### Solaris 11 (should work on 10 as well, see /etc/release to see your current specific release version info)
Solaris can be a little picky (about just about everything, but I digress) about extraneous "stuff" in the certificate file. Remove all leading and trailing spaces. One line return at the end of the file is fine, but everthing needs be removed except the data beginning with -----BEGIN CERTIFICATE----- and ending with -----END CERTIFICATE-----.

Everything check out OK? Good, now proceed with these steps:
- copy the cert file (text format, Base-64, .pem extension) to /etc/certs/CA
- the user is root and the group is sys  
`chown root:sys your_cert_name.pem`
- the permissions need to be rw for the user and ro for the group and world  
`chmod u=rw,go=r your_cert_name.pem `
- restart the ca-certificates service  
`svcadm restart /system/ca-certificates`  
- verify it restarted correctly  
`svcs -x ca-certificates`

#### Windows (most versions from Windows 7 and up, and Server 2012 and up)
This is another method that does not require a GUI or an MMC snap-in. The mmc method is really well documented but, again, takes too many clicks for my liking. This is the PowerShell method that should work on any current and many older releases of Windows.

If you want to use a GUI method just make sure you are working on the cert stores for the **Local Computer**

Here's the command line method:
- log onto Windows with an account that has local admin privileges
- copy the cert file (text format, Base-64) to a convenient place in your file system, the file extension doesn't matter
- open a PowerShell window with Run as Administrator privileges
- `Import-Certificate -CertStoreLocation 'Cert:\LocalMachine\Root' -FilePath your_cert_name.pem`

Verify your cert got to the right place like this:  
`Get-ChildItem Cert:\LocalMachine\Root\`

#### Cygwin (Yes Cygwin, exactly the same as Red Hat)
Read the file /etc/pki/ca-trust/source/README for pretty much these exact instructions.

- copy the cert file (text format, Base-64, **.pem** extension) to /etc/pki/ca-trust/source/anchors
- run the command  
`update-ca-trust`

update-ca-trust in Cygwin does not return any info to the screen when adding or removing a root cert and it takes a relatively long time to complete, but it is working.

---

ref:  
[Solaris 11 root cert installation](https://docs.oracle.com/cd/E53394_01/html/E54783/kmf-cacerts.html#OSCMEkmf-taskcert)
