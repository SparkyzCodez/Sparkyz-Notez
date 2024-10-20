# [export root cert from Windows CAs.md](export%20root%20cert%20from%20Windows%20CAs.md)
(last update 2024-10-20)

**Notes:**

The export instructions work for Windows 2019 and Windows 2022.

---

First we need to get the CA root cert from our Windows CA.
### Export Root Cert from Windows CA - Two Methods

We'll start with the easy way. If you can't launch the Certificate Authority tool then you'll need to use the mmc method.
#### CA method
- log onto a CA server
- Server Manager -> Tools -> Certification Authority
- right click your domain CA -> Properties
- General tab -> choose your cert -> click View Cerificate
- Details tab -> Copy to file...
- choose Base-64 encoded X.509 (.CER) format -> name it whatever you like but I suggest  
`\<domain name>-CA.pem`. Note that it is .crt, but if you miss that you can just rename it later.

If you can't run the Certification Authority tool or you just like using mmc do this
#### mmc method
- on a CA server run mmc (you may need to launch it from a PowerShell or cmd command line)
- Add/Remove Snap-ins -> Certificates -> Computer Account -> Local Account -> OK
- Certificates -> Trusted Root Certification Authorities -> Certificates
- find the root cert
	- it will be in the name format \<domain name>-CA
	- not in the name format CA-\<domain name>
	- check the details, the correct cert will have been created (probably) with the Certificate Template Name: CA
- right click -> All Tasks -> Export... -> the export wizard launches
- choose Base-64 encoded X.509 (.CER) format -> name it whatever you like but I suggest  
`\<domain name>-CA.pem`. Note that it is .crt, but if you miss that you can just rename it later.

You may use the Web Cert portal if it's available to you too. Whatever you like. The important point is that you end up with a root certifcate in PEM/Base64 format.

---