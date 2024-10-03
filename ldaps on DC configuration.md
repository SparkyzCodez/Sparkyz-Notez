# [ldaps on DC configuration.md](ldaps%20on%20DC%20configuration.md)  
(last update 2024-10-03)

**How To:**  
enable encrypted ldaps using TLS  
create a certificate template that will work for ldaps  
enable ldap signing and channel binding via GPO  
harden Windows server to not respond to unencrypted ldap requests  
method to request a ldaps certificate from a third (3rd) party CA  
include necessary SAN and OID information in SSL/TLS certificates
how to create a CSR (certificate signing request) in Windows
place ldaps SSL/TLS certificates in the preferred location (it's not in the computer certificate personal folder)

**Prereqs:**  
These instructions assume you have a Windows Enterprise CA (AD CS) from which you can get the root CA chain and a cert to use with TLS. We are not using self-signed certs for this because they are too weak for production networks. If you are doing self-signed then you may be able to get what you need from this document anyway.

My lab setup
- dc1.labworld.example.com is the domain controller (Windows 2019) and also DNS and DHCP services
- ca.labworld.example.com is the enterprise CA server (Windows 2022) with AD CS installed and configured
- ldapsconnect.labworld.example.com is the CNAME DNS record that will point to the ldaps server, in this case a domain controller
- network is 172.17.1.0/24 (I'm not sure if I'll mention it in my instructions below, but I'm putting it here just in case)

**Preamble:**  
You may not need to do anything to make ldaps work. Surprised? I was, but yes, you might already be done putting ldaps online. There are strings attached though. You may be done if you meet all three of these criteria:
- you have a Windows Enterprise CA infrastructure(AD CS enterprise configuration)
- autoenrollment is enabled or set to the default settings
- your Domain Controller certificate template is set to the default security permissions that allow it to autoenroll

Here's how it works. The Domain Controller certificate template will create a cert in the local computer Personal folder with the domain controller's FQDN as the subject, and it will also put the FQDN in the SAN (subject alternative name). The SAN is in the extended information section of the certificate. The template also creates a 2048 bit private key and is set for Server and Client Authentication. That's the basics of what you need for SSL/TLS and ldaps. It does all of these automatically because of autoenrollment.

Sounds pretty good right? It can be. If you are using defaults for everything then all you have to do is wait for the certificate to be created. Just look in the local computer personal certificate store. One minute it's empty, then poof, your domain controller cert that's suitable for SSL/TLS is there. That also means your CA trusted root certificate will get installed automatically too.

At first there's no DC cert:
![no DC cert](ldaps%20on%20DC%20configuration-images/no_dc_cert.jpg)

If you don't see the certs in a timely fashion you can probably speed up the process with a `gpupdate /force`.

Then there is a DC cert:
![DC cert](ldaps%20on%20DC%20configuration-images/dc_cert.jpg)

And ldaps just works without any other configurations:  
(This image is from Softerra ldap browser. It shows that I used SSL, and in the background shows some of the ldap data.)
![ldaps works without additional config](ldaps%20on%20DC%20configuration-images/ldaps_works_without_config.jpg)

That's really convenient, but that may not be the cert you really need. Here's the catch. You must use the domain controller's hostname for all your ldaps connections because that's all that's included in the autoenrolled certificate. I really don't like this. It's fine for a little lab but a real AD domain has several domain contollers. Those DCs will probably be located in different network segments and quite possibly different regions. Sometimes (perhaps too often) domain controllers need to be decommissioned. That means DNS and hostname changes, which means ldaps will be offered on a different domain controller. In addition, you may be using AD LDS and the ldaps connection point isn't on a domain controller at all. And with more complex networks you'll probably have split DNS configured, giving you even more places to configure manually.

One solution is to usa a unique hostname for your ldaps services that will work everywhere. When the underlying servers change, your ldaps clients won't know and they won't care. You only need to update DNS records for an easy transition. (Updating passwords, however, still requires a visit to each client. You do have Ansible or some other automation, right?)

Here's what I do. I use a CNAME (alias) DNS record to point to the server that's providing my ldaps. If you need to change your source ldap server for any reason you just tweak the CNAME to point to the new system. Use split DNS? No problem, just create a CNAME or an A record in each seperate zone. For this to work you must put the FQDN in the SAN of your ldaps SSL/TLS certificate.

Each server providing ldaps, wether that's a domain controller or an AD LDS server, gets its own certificate. In each certificate I put the host FQDN, the ldaps CNAME FQDN, and any other FQDNs that I might use for ldaps.

Wether you decided to use default certificates or you need more detailed certificates, there's still more work to do. That work is hardening your ldap server. We'll cover that farther down after we take care of all the certificate work.

**Notes:**  
First note - I'm learning this as I write it up. I'm sure there are gaps in my understanding. My rationale is that if I can explain it to other people then I probably have a good understanding. Probably. I tested and retested these steps several times. Leave comments or open issues so we can make this better.

These procedures were developed on Windows 2019 servers and verified on Windows 2022 servers.

The default unencrypted ldap port is 389. The default encrypted ldaps port is 636.

**There are two other ldap ports to consider** The unencrypted ldap port used for LDAP for Global Catalog is 3268. The encrypted ldaps port used for LDAP for Global Catalog is 3269. Why do we care? Because users can connect to these ports and traverse all elements of directory and get many of the same node properties. That's the bad news. The good news is that everything we do in the following steps automatically applies to the Global Catalog ports as well. You don't have to take any additional steps.

You do not need to install Windows AD Lightweight Directory Services (AD LDS) on your domain controllers because domain controllers already have ldap (and ldaps) services running. If you do run AD LDS, the process of applying a certificate is similar. Run mmc -> certificates -> **service account** -> pick your LDS instance. Additional AD LDS configuration is outside the scope of this document.

Windows doesn't seem to make anything easy. Yes, there are open source tools (openssl on Cygwin, Linux, Unix, etc.) and old school Windows (certutil.exe) that can generate and convert certificates, but I'm sticking with 100% Windows GUI methods in this writeup. I spend most of my time with Linux and Unix, and I need to learn more Windows-only methods.

If you use a third party CA to issue your certs then you will need three things in the CSR at a minimum. You will likely add much more information, but without these three items, your cert won't get the job done. You will need the FQDN of your ldaps in the extended information in the form of DNS entries. We call that the SAN (subject alternative name). There can be multiple entries, but the minimum is the fqdn you're using for your ldaps. It's also a good idea to put any other hostname or alias associated with that server. The other two things needed are OIDs (object identifier codes) for server authentication and client authentication. These are the the OIDs that correspond to those two usage cases:
- 1.3.6.1.5.5.7.3.1 for Server Auth
- 1.3.6.1.5.5.7.3.2 for Client Auth (this one may be optional, but it is very common in SSL/TLS certs)

These instructions assume you have a Windows AD CS Enterprise Certificate Authority infrastructure from which to request a cert. The first section is about requesting the cert based on a template and getting that cert installed into the Personal certs. If you use other certificate authorities or have chosen to use self-signed certs then just refer to the instructions below to get your certs installed to the correct locations.

If certficate auto-enrollment is disabled on your system, and you just want a simple cert that will work on your domain controller then just use the default Domain Controller template.
![default DC template](ldaps%20on%20DC%20configuration-images/default_dc_template.jpg)

ldap sends passwords in clear text by default. ldap with signing and channel binding is a bit stronger. Domain joined computers will automatically pass their encrypted credentials/authorization in the ldap bind request. Non domain joined systems will simply fail because ldap will reject the connection due to weak authentication. 

Signing and channel binding do not actually stop systems from **trying** to login with clear text credentials. The clear text can still be easily sniffed with Wireshark or some other packet capture. It's essential that any passwords being used before we harden the system are changed.

By default private keys are stored in a hidden area of the certificate store. They are not bound to most of the certificate file formats. If you need a certificate with an embedded private then you'll have to export the cert to the PFX file format. You'll have to set a password on the file as part of the export process. I'll put a reference to the process at the bottom of this document.

User connection strings in Softerra work in two formats, the first format is probably the most compatible with any ldap client.
- ldap DN style: CN=Sparky,CN=Users,DC=labworld,DC=example,DC=com
- Windows style: labworld.example.com/Users/Sparky

My testing says it's not possible to completely block unencrypted ldap. When I tried, I could not get new computers joined to the network and group policy updates would not work. I verified the traffic with Wireshark. The bind request (initial login) is encrypted by ldap signing if that's enabled. The subsequent data transfer contains some clear text data. I don't know if this is subject to a MITM attack or if the protocol with signing and channel binding would detect the tampering.

Just a reminder, before your clients can use ldaps they will need to trust your CA by installing a root certificate.

Never install AD CS certificate services on a domain contoller. It needs to be on a seperate server that is especially hardened. A proper certificate authority infrastructure is way beyond the scope of this document, but the main point is that you root CA should be configured, then subordinate downstream CAs configured, then you turn off your root CA and leave it off except when it's time to update a root certificate. If your root CA chain gets compromised then the bad guys will own your certificate based auth. You don't want that.

**Important: I couldn't get ldp.exe, which is included with Windows, to work with ldaps. The Softerra LDAP Browser 4.5, however, worked without problems. I don't want you to get stuck troubleshooting a problem that doesn't exist, so don't jump to conclusions if ldp.exe doesn't work. Try another method before you decide to fix something that isn't broken.**  

I run mmc directly from a command prompt for almost all of these steps. If you already know the [snap-in].msc name or how to otherwise launch the necessary console directly then by all means do that.

I'll put reference links that relate to detecting ongoing attempts to login with unencrypted ldap at the very bottom of this document.

Final rant - Why must Windows require so many clicks and steps to do this stuff? Yours truly, a Linux guy. :-)  

---

# We are doing two broad tasks in this document
- First we **Enable ldaps**. We'll work through the necessary steps if you have more complex certificate needs. I did have conflicts and cert SAN issues, so I did need to use this process.
- Then we **Enforce Server Side ldap Signing and Channel Binding with a GPO** to guard against unsigned (clear text authentication) ldap connection attemps. Strictly speaking, this is just barely hardening because we are leaving ldap listening and able to respond to requests, but it does prevent attacks from non domain joined systems that can't appropriately sign their authentication requests.

---

## Enable ldaps

### Verify that you have a root certificate for your CA installed, install it if necessary (two methods: the web portal hard way, and directly from the CA)
I'm not sure we need to do anything with the root cert on the ldap server, so just check that you already have the trusted root CA cert installed. I think a Windows Enterprise CA distributes the root certs to domain domain joined systems automatically, but I am not 100% certain.

**Do this on the server on which you are configuring ldaps.**
- run MMC -> add snap-in... -> Certficates -> **Computer Account** -> Local Computer -> Finish then OK
- expand Certificates (local) -> Trusted Root Certification Auth -> Certificates (be sure to try refresh before making a final determination)
- IF there is no root certificate then get one from the CA
	- Method 1 (the certificate web services way) - Get a CA root chain from your CA web portal (use whatever other method at your disposal to get a CA chain cert bundle)
		- `https://ca.labworld.example.com/CertSrv` of course replace te domain name
		- use domain prefix for your login name labworld\sparky
		- select Base64 (later, you can verify that it's Base64 by opening it in a text editor, it will be legible)
		- download the CA certificate chain
	- Method 2 (the direct way) - Get a CA root chain from your CA server
		- on the CA server: run MMC -> add snap-in Certficates -> Computer Account -> Local Computer -> Finish then OK
		- expand Certificates (local) -> Trusted Root Certification Auth -> Certificates
		- right click the root cert -> All Tasks -> Export...
		- choose Cryptographic Message Syntax Standard (.P7B), !! Include all certs in the path !! -> and save it to a file, get it to the ldaps server
	- right click -> All Tasks -> import... -> verify local computer -> choose your downloaded chain cert
	- install in Trusted Root Certification Authorities -> finish

It should look something like this when you are done:
![root cert is installed](ldaps%20on%20DC%20configuration-images/ca_root_cert.jpg)

### Configure an appropriate template to generate an ldaps certificate
I've read several methods to do this. Some use the kerberos template and others use the IPsec offline template for the source. I am using the **Domain Controller** template as my source.

**Do this on the enterprise CA server.**
- on your CA: run MMC -> add snap-in... -> Certficate Templates -> Finish then OK
- right click Domain Controller certificate (not the Domain Controller Authentication cert) -> Duplicate Template (the duplicated cert will open in a Properties window)
![duplicate template](ldaps%20on%20DC%20configuration-images/dupe_template.jpg)
	- General tab - rename to Domain Controller for LDAPS, set to one year validity -> click Apply (but don't close the window yet, we have more work to do)
	- Request Handling tab - **select Allow private key to be exported**
	![allow private key export](ldaps%20on%20DC%20configuration-images/exp_pri_key.jpg)
	- Cryptography tab - verify the Minimum key size is 2048, and Microsoft RSA SChannel Crypto Provider is selected
	- Extensions tab - we're just checking but not changing anything in this tab
		- Application Policies - verify both Client Authentication and Server Authentication are set
		![verify application policies](ldaps%20on%20DC%20configuration-images/app_pol.jpg)
		- leave the other sections unchanged
	- Subject Name tab - select Supply in the request (You will get a warning. Just accept the risk for now. Once the certs are generated you can switch it to require certificate manager approval.)
	![supply in request](ldaps%20on%20DC%20configuration-images/supply_in_req.jpg)
	- Leave all the other Properties tabs unchanged, the Domain Controller source template was already setup correctly for everything else
	- click Apply then OK

Notice that the template schema version has incremented, that we have a new and previously unused version number, and that server and client auth appear as intended purposes
![new template](ldaps%20on%20DC%20configuration-images/new_template.jpg)

Now do yourself a favor. Open the new template and double check everything. If you miss clicking apply on any of the tabs then the settings won't be saved. The private key must be exportable.

### Enable the Template
In this step we have to allow this template to be used. If you miss this step you'll never see our new template on other systems.
- on your CA: Server Manager -> Tools -> Certificat Authority (you may also launch this as a snap-in from an mmc window)
- right click on Certificate Templates -> New -> Certificate Template to Issue
- choose our new Domain Controller with LDAPS certificate

It should look like this when you're done:
![enabled template](ldaps%20on%20DC%20configuration-images/enabled_template.jpg)

### Generate a Suitable Certificate - The Easy Way One Step Method, request and install cert in Local Machine/Personal/Certificates
Here's the easy way to generate a cert if you're only using the hostname to your ldaps server. You may want to skip this next step and go down to the hard way. Sometimes Windows uses the wrong certificate, meaning not the one you want it to use. We'll cover the details below in The Hard (better) Way.


**This step is done on your ldaps server.**
- run MMC -> add snap-in... -> Certficates -> **Computer Account** -> Local Computer -> Finish then OK
- expand Certificates (local) -> Personal -> Certificates
- right click Certificates -> All Tasks -> Request New Certificate...
![request new cert](ldaps%20on%20DC%20configuration-images/request_new_cert.jpg)
- select Active Directory Enrollment Policy -> next  
**Don't click Enroll until the instructions say to do that**
- check the Domain Controller for LDAPS check box -> click **details** to the right -> click the Properties button (or click on the warning line: More information is required to...)
![cert properties button](ldaps%20on%20DC%20configuration-images/cert_properties_button.jpg)
	- General tab - pick a Friendly name so it's easier to find in the certificate consoles, just a good practice -> click Apply
	![general tab](ldaps%20on%20DC%20configuration-images/cert_prop_general.jpg)
	- Subject tab - this is really important information, some items are optional but other items are mandatory, check with your CA issuer
		- Subject Name: Full DN (use this format, not FQDN, something like `cn=dc1, dc=labworld, dc= example, dc=com`)
			- alternatively you can add each component seperately but you need to get the domain components (dc items) in the right order
			![DN format and correct order](ldaps%20on%20DC%20configuration-images/cert_prop_subject-DN.jpg)
		- Subject Name: Country - use your two letter ISO country code, don't use full words here
		- Subject Name: Locality (city), State
		- Alternative Name: DNS - your FQDN, for example dc1.labworld.example.com, ldapsconnect.labworld.example.com, etc.
			- **This is the SAN info that's necessary to make SSL/TLS certs work correctly.**
			![cert SAN info](ldaps%20on%20DC%20configuration-images/cert_prop_subject-SAN.jpg)
		- click Apply
	- Extensions tab
		- Key usage - verify Digital Signature and Key encipherment
		- Extended Key Usage - verify Server Authentication and Client Authentication
		- Custom extension definition - this is where you put any **additional** OIDs you need, this is not the right place for Server and Client auth because Windows already has check boxes for those two purposes in the Extended Key Usage section
	- Private Key tab
		- Cryptographich Service Provider - verify Microsoft RSA Schannel
		- Key options - verify Make key exportable is checked (this is a convenience for the Easy Way but mandatory for the Hard Way)
		![exportable private key](ldaps%20on%20DC%20configuration-images/cert_prop_prikey-exportable.jpg)
		- Key type - verify Exchange (not signature) is selected
		- Key permissions - do not set Use custom permissions
	- Certification Authority - verify your CA is selected
	- Signature - none
	- Click the OK button
- **Click Enroll** (now you may click the Enroll button)

At this point you probably have a new certificate that was placed in your Personal Certificates folder along side the Domain Controller cert. If that didn't happen then your automatic creation is probably blocked by your CA. Go to the CA, find the pending certificate request, and approve it.
![personal cert store with ldaps SSL/TLS cert](ldaps%20on%20DC%20configuration-images/cert_ldaps.jpg)

Take a look at the icon just to the left of the certificate name. Notice that there is a key in the upper left and a ribbon in the bottom right. The key indicates that the private key is attached to this certificate. Not all certs need an attached private key, so you won't always see this. It is, however, essential for our purpose.
![private key attached symbol](ldaps%20on%20DC%20configuration-images/cert_privatekey.jpg)

There's another way to verify the private key status. Check the properties of the cert. At the bottom of the general tab it should say: *You have a private key that corresponds to this certificate.*

Test all your FQDNs that you put in the certificate SAN. I do my testing with Softerra LDAP Browser. It's free and provides read only access to the directory. One thing you will notice is that you get a warning `No client certificate to authenticate to...`. The is not talking about the server side certificate; it's only telling you that you aren't using a certificate as well. It's OK, no problem.

The problem to look for is if ldaps is using the wrong certificate. If the the SAN names don't work then ldaps is using a different cert such as the autoenrolled domain certificate that is also in the personal store. You can try rebooting and retesting. If you're still using the wrong cert then delete the cert you just added and move on to hard way described just below.

If everything is working the way you like it then scroll on down to the hardening process.

---
This is the hard way. (But it is the better way.)

Objectives:
- generate a CSR (certificate signing request) file
- create a cert from the CSR in Windows AD CS and save it to file
- import cert to Service Account/Local Comuter/Active Directory Domain Services/NTDS/Personal/Certificates

It kind of begs the question. Why would you do this if there's an easier way? The answer is if you are having certificate conflicts. Windows has implicit rules about which certificate gets used. If there are certs with the same names and characteristics in the folder then it will pick the one with the latest expiration date.

There's a solution. For ldaps, the server looks first in the NTDS/Personal folder. We are going to put our cert there so Windows never has to guess (wrongly) about which cert to use for ldaps. This is actually the preferred location. The location we used above just happens to work, but it's not ideal. Look for the link at the bottom of this page, *Enable LDAP over SSL with a third-party certification authority*, for details.

Some of the steps below will look very similar to what we've already done. There are differences though, so pay attention to the details.

Ready for the hard way that may just be better for you? No? Me neither, but here we go.

### Generate a CSR (certificate signing request)
**This step is done on your ldaps server.**

We're creating a CSR so that a certificate is NOT automatically created in the local machine personal store. We just want the CSR file itself. You can also use the CSR file with external certificate authorities.
- run MMC -> add snap-in... -> Certficates -> **Computer Account** -> Local Computer -> Finish then OK
- expand Certificates (local) -> Personal -> Certificates
- right click Certificates -> All Tasks -> Advanced Operations -> Create Custom Request...
![request CSR](ldaps%20on%20DC%20configuration-images/request_CSR.jpg)
- select Active Directory Enrollment Policy -> next
- Template -> Domain Controller for LDAPS, PKCS#10 format
![ldaps template pkcs#10 format](ldaps%20on%20DC%20configuration-images/request_CSR-ldaps_template.jpg)
- Domain Controller for LDAPS will be checked and greyed out **-> click details to the right** -> click the Properties button
![CSR properties](ldaps%20on%20DC%20configuration-images/csr_properties_button.jpg)
	- General tab - pick a Friendly name so it's easier to find in the certificate consoles, just a good practice -> click apply
	![general tab](ldaps%20on%20DC%20configuration-images/cert_prop_general.jpg)
	- Subject tab - this is really important information, some items are optional but other items are mandatory, check with your CA issuer
		- Subject Name: Full DN (use this format, not FQDN) - 
			- alternatively you can add each component seperately but you need to get the domain components in the right order
			![DN format and correct order](ldaps%20on%20DC%20configuration-images/cert_prop_subject-DN.jpg)
		- Subject Name: Country - your two letter ISO country code
		- Subject Name: Locality (city), State
		- Alternative Name: DNS - your FQDN, for example dc1.labworld.example.com, ldapsconnect.labworld.example.com, etc.
			- **This is the SAN info that's necessary to make SSL/TLS certs work correctly.**
			![cert SAN info](ldaps%20on%20DC%20configuration-images/cert_prop_subject-SAN.jpg)
		- click Apply
	- Extensions tab
		- Key usage - verify Digital Signature and Key encipherment
		- Extended Key Usage - verify Server Authentication and Client Authentication
		- Custom extension definition - this is where you put any additional OIDs you need, don't guess, only do it if you are certain
	- Private Key tab
		- Cryptographich Service Provider - verify Microsoft RSA Schannel
		- Key options - **verify Make key exportable** is checked
		![exportable private key](ldaps%20on%20DC%20configuration-images/cert_prop_prikey-exportable.jpg)
		- Key type - verify Exchange (not signature) is selected
		- Key permissions - do not set Use custom permissions
	- choose a file name an location, Base64 format (compatible with 3rd party CAs)
	![save the CSR req file](ldaps%20on%20DC%20configuration-images/cert_ldaps_save_CSR.jpg)
	- Click the OK button

Now our CSR has been generated. You can use it with a third party CA or our in-house Enterprise CA.

### Generate the Certificate File
The CA should automatically create a certificate file when we follow these steps.

If the CA simply creates a certificate without offering to save the file then we don't have the private key in the same certificate. There are other steps to reconnect the private key to the public key but that is beyond the scope of these procedures. Check the notes at the bottom of this document for more info. If you followed all the steps then you'll just be prompted to save the cert file and we can move to the next step.

**Do this on the enterprise CA server.**
- Server Manager -> Tools -> Certificate Authority
- right click the root node of your CA -> All Tasks -> Submit new request...
![submit the CSR to the CA](ldaps%20on%20DC%20configuration-images/ca_submit_csr.jpg)
- choose the CSR file we created in the previous section
![choose the CSR file](ldaps%20on%20DC%20configuration-images/ca_choose_CSR.jpg)
- if accepted you will prompted you to save the newly created certificate file, save it in the .cer format.
![save the .cer file](ldaps%20on%20DC%20configuration-images/ca_save_cer_file.jpg)

If your certificate issuance fails then look in the Certificate Authority Failed Requests to see the error and to either fix the problem and redo the process, or force the system to issue your cert anyway.

If you get an error "The DNS name is unavailable and cannot be added to the Subject Alternate name..." then you forgot to change the Subject Name option in the template. It's not too late. Just change the template and then resubmit your CSR. Uh, ask me how I know.

### 
**This step is done on your ldaps server.**
- run MMC -> add snap-in... -> Certficates -> **Service Account** -> Local Computer -> Active Directory Domain Services -> Finish then OK
![certificate snap-in service account](ldaps%20on%20DC%20configuration-images/cert-service_account.jpg)
![active directory domain service](ldaps%20on%20DC%20configuration-images/cert-service_adds.jpg)
- expand Certificates (local) -> NTDS\Personal -> Certificates
- right click Certificates -> All Tasks -> Import...
![import option](ldaps%20on%20DC%20configuration-images/cert-service_adds-import.jpg)
- choose the certificate file we created in the previous step -> next -> place in the NTDS\Personal folder
	- If the cert is in PFX format with an embedded private key you'll also be asked for the decryption passwor, .cer files don't need this

Take a look at the icon just to the left of the certificate name. Notice that there is a key in the upper left and a ribbon in the bottom right. The key indicates that the private key is attached to this certificate. Not all certs need an attached private key, so you won't always see this. It is, however, essential for our purpose.
![private key attached symbol](ldaps%20on%20DC%20configuration-images/cert_privatekey.jpg)

There's another way to verify the private key status. Check the properties of the cert. At the bottom of the general tab it should say: *You have a private key that corresponds to this certificate.*

That should do it. Test your ldaps connection. Test all your FQDNs that you added to the cert. Consider deleting the stray certificate files you have on your file system now.

---

Congratulations. You got ldaps working. Now it's time to harden the server and prevent ldap in clear-text from being serviced.

## Enforce Server Side ldap Signing and Channel Binding with a GPO

**This can have really big impact on your systems. Test and verify if you like having a job.**

So far all we've done is enable error free ldaps. Very low risk stuff. When we start enforcing signing and channel binding old clients and non-domain joined clients will start failing. Have a plan. Use good communication and change control.

There are a number of ways to enforce signing. You configure it on both the server side and on the client side. Enforcing this on all possible clients, Windows, Linux, Unix, Mac, etc., is outside the scope of this document. Accomplishing this on Windows domain joined computers is relatively easy though. We can use a GPO and apply it to all domain controllers. You may need to widen this to additional servers if you install AD LDS. (It's also possible to edit the registry, but we're focusing on domain joined systems that respond to GPO settings.)

Domain joined systems using the currently logged in user account will automatically respect the signing policy. For more information look at the reference document *How to enable LDAP signing in Windows* mentioned in the reference links of this document. That's a good start, but most of the time we'll be dealing with non domain joined or non Windows systems. That's outside the scope of this document, but the process will start with a good inventory of all your clients and outside requesters.

We'll focus on our Windows ldap servers here.

### Configure the GPO for Server Side ldap Signing and Channel Binding
**This step is done on your ldaps server.**
- run mmc -> add snap-in... -> Group Policy Management Editor -> Browse for Group Policy Object: Domain Controllers your.domain (double click, drill down) -> Default Domain Controller Policy -> Finish -> OK
![gpo default domain controller policy object](ldaps%20on%20DC%20configuration-images/gpo_object.jpg)
- (drill into tree) Default Domain Controller Policy -> Computer Configuration -> Policies -> Windows Settings -> Security Settings -> Local Policies -> then select Security Options
- (signing) scroll to Domain controler: LDAP server signing requirements -> right-click and selct Properties
- (signing) check box Define this policy setting, set to Require signing, Apply/OK
- (channel binding) scroll to Domain controler: LDAP server channel binding token requirements -> right-click and selct Properties
- (channel binding) check box Define this policy setting, set to Always, Apply/OK
![gpo settings](ldaps%20on%20DC%20configuration-images/gpo_location_settings.jpg)
- do a gpupdate /force to get the policy enacted quickly -> open a PowerShell or cmd prompt and type `gpupdate /force`
- verify for yourself (and consider using Wireshark to see the negotiation in action)
	- from a non domain joined computer or plain text password browsing on ldap will both fail with Invalid Credentials or Strong Authentication Required.
	![clear text creds after GPO](ldaps%20on%20DC%20configuration-images/ldap-clear_text_creds.jpg)
	- from a domain joined computer ldap will still work as expected only if you use the currently logged in user using Windows authentication (kerberos) option in Softerra.
	![with signing and kerberos auth](ldaps%20on%20DC%20configuration-images/ldap-with_signing.jpg)
	- from a non domain joined computer browsing on ldaps (TLS) will work as expected (tested with Softerra browser).
	![non-joined ldaps success](ldaps%20on%20DC%20configuration-images/ldaps_nonjoined_success.jpg)
	- Similarly, ldap queries from Linux/Unix systems using ldapsearch will now fail, but ldaps queries will succeed.
	![ldaps linux success](ldaps%20on%20DC%20configuration-images/ldaps_linux_success.jpg)

### Not quite done yet
Our last step is to change all the ldap related passwords for any account you use specifically for inbound ldap. Up until this point in time your client systems have been sending passwords in clear text. That means any accounts we have in the AD that are used specifically for ldap could already be compromised. It's time to change those passwords.

You also need to re-configure all those non-domain joined systems to use ldaps, or at least used signed bind/authentication for ldap. I'll write up the necessary changes for Linux/Unix in a near-future document. That's what drove my desire to start this project in the first place.

---

ref:  
[AD CS with Web Enrollment configuration.md](../AD\%20CS/AD\%20CS\%20with\%20Web\%20Enrollment\%20configuration\%20.md) (my notes)  
[Obtain a certificate for use with Windows Servers and System Center Operations Manager](https://learn.microsoft.com/en-us/system-center/scom/obtain-certificate-windows-server-and-operations-manager?view=sc-om-2022&tabs=Enterp%2CEnter)  
[Enable LDAP over SSL with a third-party certification authority](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/enable-ldap-over-ssl-3rd-certification-authority)  
[How to enable LDAP signing in Windows](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/enable-ldap-signing-in-windows-server#how-to-set-the-client-ldap-signing-requirement-by-using-a-domain-group-policy-object)  
[Enable LDAP over SSL with a third-party certification authority](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/enable-ldap-over-ssl-3rd-certification-authority)  
[ldaps certificates on domain controllers](https://xdot509.blog/2020/12/21/ldaps-domain-controller-certificates/)  
[LDAP Global Catalog Info](https://docs.servicenow.com/bundle/xanadu-platform-security/page/integrate/ldap/reference/r_LDAPUsingGlobalCatalog.html)  
[Binding to the Global Catalog](https://learn.microsoft.com/en-us/windows/win32/ad/binding-to-the-global-catalog)  
[Export a PFX format certificate with an embedded private key](PFX\%20export\%20with\%20private\%20key\%20intact.md)  
[Configure firewall rules with group policy](https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/configure)

event logging links:  
[LDAP/LDAPS authentication Audit through win events](https://learn.microsoft.com/en-us/answers/questions/558208/ldap-ldaps-authentication-audit-through-win-events)  
[Whats using LDAPS, Check in event viewer](https://learn.microsoft.com/en-us/answers/questions/228923/whats-using-ldaps-check-in-event-viewer)
