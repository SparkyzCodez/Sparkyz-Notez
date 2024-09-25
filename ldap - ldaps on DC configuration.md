[ldaps on DC configuration.md](ldaps%20on%20DC%20configuration.md)  
(last update 2024-09-25)

How To:  
enable ldap signing and channel binding via GPO  
enable encrypted ldaps using TLS  
create a certificate template that will work for ldaps  
harden Windows server to not respond to unencrypted ldap requests  
request a ldaps certificate from a third (3rd) party  
place ldaps SSL/TLS certificates in the preferred location (it's not in the computer certificate personal folder)

Prereqs:  
These instructions assume you have an Enterprise CA from which you can get the root CA chain and a cert to use with TLS. We are not using self-signed certs for this. They are too weak for a production network. If you are doing self-signed then you may be able to get what you need from this document anyway.

Notes:  
First note - I'm learning this as a write it up. I'm sure I am misunderstanding a number of these concepts, but in all it is a solid procudure. Leave comments or open issues so we can make this better.

These procedures were developed on Windows 2019 servers and verified on Windows 2022 servers.

The unencrypted ldap default port port is 389. The encrypted ldaps default port is 636.

Windows doesn't seem to make anything easy. Yes, there are open source (Cygwin, Linux, Unix, etc.) and old school Windows (certutil.exe) tools that can generate and convert certificates, but I'm sticking with 100% Windows GUI methods in this writeup. I spend most of my time with Linux and Unix, and I need to learn more Windows-only methods.

These instructions assume you have a Windows AD CS Enterprise Certificate Authority infrastructure from which to request a cert. The first section is about requesting the cert based on a pre-existing template and getting that cert installed into the Personal certs. If you use other means to get a suitable cert then just put your cert into the Personal certs and jump to the second and subsequent sections.  

**Important: I couldn't get ldp.exe, which is included with Windows to work with ldaps, but the Softerra LDAP Browser 4.5 worked without problems. I don't want you to get stuck troubleshooting a problem that doesn't exist, so don't jump to conclusions if ldp.exe doesn't work. Try another method before you try to fix something that isn't broken.**  

ldap sends passwords in clear text - Only using currently logged in user in Softerra from a domain joined computer and an AD authenticated account avoids this. All other non-encrypted login methods have this problem. That's the problem we're addressing in this document.

user connection strings in Softerra work in two formats, the first format is probably the most compatible
	- ldap DN style: CN=Sparky,CN=Users,DC=labworld,DC=example,DC=com
	- Windows style: labworld.example.com/Users/Sparky

Just a reminder, before your clients can use ldaps they will need to trust your CA by installing a root certificate.

Final rant: Why must Windows require so many clicks and steps to do this stuff? Yours truly, a Linux guy. :-)  

---

## We are doing two broad tasks in this document
- First we enable ldaps. We'll use an easier method and then work through steps in a harder method that you may need if you have certificate conflicts.
- Then we harden the server for unencrypted and unsigned ldap connection attemps. You can' turn off clear-text ldap, but you can keep it from leaking out of your server.

---

### Enable ldaps

Note: I'm not sure we need to do anything with the root cert, so just check that you already have the root CA cert installed. I think a Windows Enterprise CA distributes the root certs to domain controllers automatically. A CA on a DC is a bad idea, especially if it's the root CA. Yeah, don't do that.

#### Verify that you have a root certificate for your CA installed, install it if necessary (two methods: the web portal hard way and directly from the CA)
Do this on the server on which you are configuring ldaps.
- run MMC -> add snap-in Certficates -> Computer Account -> Local Computer -> Finish then OK
- expand Certificates (local) -> Trusted Root Certification Auth -> Certificates (be sure to try refresh before making a final determination)
- IF there is no root certificate then get one from the CA
	- Method 1 (the hard way) - Get a CA root chain from your CA web portal (use whatever other method at your disposal to get a CA chain cert bundle)
		- `https://ca.labworld.example.com/CertSrv` of course relace te domain name
		- use domin prefix for your login name labworld\sparky
		- select Base64 (you can verify that it's Base64 by opening it in a text editor, it will be legible)
		- download the CA certificat chain
	- Method 2 (the easy way) - Get a CA root chain from your CA server
		- on the CA server: run MMC -> add snap-in Certficates -> Computer Account -> Local Computer -> Finish then OK
		- expand Certificates (local) -> Trusted Root Certification Auth -> Certificates
		- right click the root cert -> All Tasks -> Export...
		- choose Cryptographic Message Syntax Standard (.P7B), !! Include all certs in the path !! -> and save it to a file, get it to the ldaps server
- right click -> All Tasks -> import... -> verify local computer -> choose your downloaded chain cert
- install in Trusted Root Certification Authorities -> finish

#### Configure an appropriate template to generate an ldaps certificate
I've read several methods to do this. Some use the kerberos template and others use the IPsec offline template for the source. I am using the **Domain Controller** template as my source. Basically, we'll be making a Domain Controller suitable cert that will include the private key. We'll cover this in more depth a little later. SSL/TLS certs require a both a public and private key in the certificate. Ready? Let's to do this:
- on your CA: run MMC -> add snap-in Certficate Templates -> Finish then OK
- right click Domain Controller certificate (not the Domain Controller Authentication cert) -> Duplicate Template (the duplicated cert will open in a Properties window)
	- General tab - rename to Domain Controller for LDAPS -> click Apply (but don't close the window yet, we have more work to do)
	- Request Handling tab - **select Allow private key to be exported**
	- Cryptography tab - verify the Minimum key size is 2048, and Microsoft RSA SChannel Crypto Provider is selected
	- Extensions tab - we're just checking but not changing anything in this tab
		- Application Policies - verify both Client Authentication and Server Authentication are set
		- leave the other sections unchanged
	- Subject Name tab - select Supply in the request (you will get a warning, just accept the risk for now, once the certs are generated you can switch it to require certificate manager approval, I'll put a reminder near the end of these instructions)
	- Leave all the other Properties tabs unchanged, the Domain Controller source template was already setup correctly for everything else
	- click Apply then OK

Notice that the template schema version has incremented and that we have a new and previously unused version number

Now do yourself a favor. Open the new template and double check everything. If you miss clicking apply on any of the tabs then the settings won't be saved. The private key must be exportable.

#### Enable the Template
In this step we have to allow this template to be used. If you miss this step you'll never see our new template on other systems.
- on your CA: Server Manager -> Tools -> Certificat Authority (you may also launch this as a snap-in from an mmc window)
- right click on Certificate Templates -> New -> Certificate Template to Issue
- choose our new Domain Controller with LDAPS certificate

#### Generate a Suitable Certificate - One Step Method, request and install cert in Local Machine/Personal/Certificates
This step is done on your ldaps server.
- on your ldaps/DC: run MMC -> add snap-in Certficates -> Computer Account -> Local Computer -> Finish then OK
- expand Certificates (local) -> Personal -> Certificates
- right click Certificates -> All Tasks -> Request New Certificate...
- select Active Directory Enrollment Policy -> next  
**Don't click Enroll until the instructions say to do that**
- check the Domain Controller for LDAPS check box **-> click details to the right** -> click the Properties button
	- General tab - pick a Friendly name so it's easier to find in the certificate consoles, just a good practice -> click apply
	- Subject tab - this is really important information, some is optional, but the SAN (DNS name) information is essential
		- Subject Name: Full DN (use this format, not FQDN) - 
			- alternatively you can add each component seperately but you need to get the domain components in the right order
		- Subject Name: Country - your two letter ISO country code
		- Subject Name: Locality (city), State
		- Alternative Name: DNS - your FQDN, for example dc1.labworld.example.com, also add static IP in case anyone uses that
		- click Apply
	- Extensions tab
		- Key usage - verify Digital Signature and Key encipherment
		- Extended Key Usage - verify Server Authentication and Client Authentication
		- Custom extension definition - this is where you put any additional OIDs you need, don't guess, only do it if you are certain
	- Private Key tab
		- Cryptographich Service Provider - verify Microsoft RSA Schannel
		- Key options - **verify Make key exportable** is checked
		- Key type - verify Exchange (not signature) is selected
		- Key permissions - do not set Use custom permissions
	- Certification Authority - verify your CA is selected
	- Signature - none
	- Click the OK button
- **Click Enroll** (now you may click the Enroll button)

At this point you probably have a certificate that was automatically created and placed in your Personal Certificates folder. If that didn't happen then you automatic creation is probably blocked by your CA. Go to the CA, find the pending certificate request, and approve it.

Once the certificate is in the right place your ldaps should be working. Test it with Softerra LDAP Browser. What you will notice is that you get a warning `No client certificate to authenticate to...`. 

---
So now that we got that working let's explore a more complicated method you may need. If you already have ldaps running and you're happy with the results you can just skip down to the hardening process.

Objectives:
- generate a CSR (like you might use in a web based front end to your CA or with an external CA)
- create a cert from the CSR in Windows AD CS
- convert the cert to PFX format
- import cert to Service Account/Local Comuter/Active Directory Domain Services/NTDS/Personal/Certificates

It kind of begs the question? Why would you do this if there's an easier way? The answer is if you are having certificate conflicts. Windows has arcane rules about which certificate gets used if there are certs with the same names and characteristics. The usual deciding factor is if the cert renews automatically and has a farther expiration date.

There's a solution. For ldaps, the server looks first in the NTDS/Personal folder. We are going to put our cert there so Windows never has to guess (wrongly) about which cert to use for ldaps. This is actually the preferred location. The location we used above just happens to work, but it's not the ideal. Look that link at the bottom of this page, *Enable LDAP over SSL with a third-party certification authority*, for details.

Some of the steps below will look very similar to what we've already done. There are differences, so pay attention to the details.

Ready for the hard way that may just be better? No? Me neither, but here we go.

#### Generate a CSR (certificate signing request)
This step is done on your ldaps server.
- on your ldaps/DC: run MMC -> add snap-in Certficates -> Computer Account -> Local Computer -> Finish then OK
- expand Certificates (local) -> Personal -> Certificates
- right click Certificates -> All Tasks -> Advanced Operations -> Create Custom Request...
- select Active Directory Enrollment Policy -> next
- Template -> Domain Controller for LDAPS, PKCS#10 format
- Domain Controller for LDAPS will be checked and greyed out **-> click details to the right** -> click the Properties button
	- General tab - pick a Friendly name so it's easier to find in the certificate consoles, just a good practice -> click apply
	- Subject tab - this is really important information, some is optional, but the SAN (DNS name) information is essential
		- Subject Name: Full DN (use this format, not FQDN) - 
			- alternatively you can add each component seperately but you need to get the domain components in the right order
		- Subject Name: Country - your two letter ISO country code
		- Subject Name: Locality (city), State
		- Alternative Name: DNS - your FQDN, for example dc1.labworld.example.com, also add static IP in case anyone uses that
		- click Apply
	- Extensions tab
		- Key usage - verify Digital Signature and Key encipherment
		- Extended Key Usage - verify Server Authentication and Client Authentication
		- Custom extension definition - this is where you put any additional OIDs you need, don't guess, only do it if you are certain
	- Private Key tab
		- Cryptographich Service Provider - verify Microsoft RSA Schannel
		- Key options - **verify Make key exportable** is checked
		- Key type - verify Exchange (not signature) is selected
		- Key permissions - do not set Use custom permissions
	- choose a file name an location, Base64 format (compatible with 3rd party CAs)
	- Click the OK button

Now our CSR has been generated. You can use it with a third party CA or our in-house Enterprise CA.

#### Generate the Certificate File
We really need the CA to create a certificate file here. If it simply creates a certificate without saving the file then we won't have the private key in the same certificate. There are other steps to reconnect the private key to the public key but that is beyond the scope of procedures. Check the notes at the top of this document for more info. If you followed all the steps then you'll just be prompted to save the cert file and we can move to the next step.
This step is done on your AD CS Enterprise Certificate Authority Server.
- Server Manager -> Tools -> Certificate Authority
- right click the root node of your CA -> All Tasks -> Submit new request...
- choose the CSR file we created in the previous section
- if accepted you will prompted you to save the newly created certificate file

Now is a good time to re-edit your template to require certificate manager approval.

If your certificate issuance fails then look in the Certificate Authority Failed Requests to see the error and to force the system to issue your cert anyway.

If you get an error "The DNS name is unavailable and cannot be added to the Subject Alternate name..." then you forgot to change the Subject Name option in the template. It's not too late. Just change the template and then resubmit your CSR. Uh, ask me how I know.

#### Convert to a PFX Certificate File With The Private Key
For this step we'll import the certificate file into the personal store, then turn around and export it, with its private key, to PFX format. Then we delete the cert from the personal store. I don't like having to do this but I can't find a way to directly export the cert from the CA with the private key intact.

This step is done on your ldaps server because that's the server we used to generate the CSR.
- on your ldaps/DC: run MMC -> add snap-in Certficates -> Computer Account -> Local Computer -> Finish then OK
- expand Certificates (local) -> Personal -> Certificates
- right click Certificates -> All Tasks -> Import...
- choose the certificate we created in the previous step
- (refresh if necessary) right click the cert we just imported -> All Tasks -> Export...
- choose Yes, export the private key -> next  
This step is essential. If you don't have the option to export the private key then it didn't get associated with the certificate. You have two choices at this point. You can repair the certificate or redo CSR and initial certificate creation process. The process to repair is to get the serial number of the cert, open a command window, and exectue  
`certutil -repairstore my "PLACE_SERIAL_NUMBER_HERE"`.
- note that your only export option is now PFX format -> next
- set a password and use AES256 encryption -> next
- save the file
- delete the certificate from your personal store to avoid any side effects and confusion for Windows choosing which cert to use

#### 
This step is done on your ldaps server.
- on your ldaps/DC: run MMC -> add snap-in Certficates -> Service Account -> Local Computer -> Active Directory Domain Services -> Finish then OK
- expand Certificates (local) -> NTDS\Personal -> Certificates
- right click Certificates -> All Tasks -> Import...
- choose the PFX file we created in the previous step -> next -> enter the password to decrypt the cert -> place in the NTDS\Personal folder

That should do it. Test your ldaps connection. Consider deleting the stray certificate files you have on your file system now.

---
Congratulations. You got ldaps working. Now it's time to harden the server and prevent ldap in clear-text from being serviced.

### Harden the Server with Signing, Channl Binding, and Firewall Blocking ldap

**Note - this can have really big impact on your systems, test and verify if you like having a job**

There are a number of ways to enforce signing. You can do it on the server side and you can do it on the client side. It's important to do both. Accomplishing this on domain joined computers is relatively easy. We can use a GPO or even make registry settings.

But that's not really the problem! The problem is that any non domain joined system can attempt to make the connection with SASL (simple authentication), and by doing so they will always expose the credentials in the protocol bind request. We can't stop them from trying so we can't stop the credentials from traversing networks. And every time those systems try to connect the server will respond and tell them if their credentials are correct. That's just the way ldap works. No matter what, the server is going to tell you if your credentials are correct even after you have configured server side signing and channel binding.

OK, let's get start with hardening. Our first step is to change all the ldap related passwords for any account you use specifically for ldap. Just do it. We'll wait. ... ... ... Done? Good! That's a great start. At this point no non domain joined systems should be able to make a successful ldap request because their credentials are invalid. I did mention that this can have really big system impacts right?

There are many ways to configure clients. The domain joined systems using the currently logged in user account will automatically respect the signing policy. Look at the reference document *How to enable LDAP signing in Windows* mentioned at the top of this document for Windows configurations. That's a good start, but most of the time we'll be dealing with non domain joined or non Windows systems. That's outside the scope of this document, but the process will start with a good inventory of all your clients and outside requesters.

We'll focus on our Windows ldap servers here.

#### Enforce server side ldap signing and channel binding with a GPO
- run mmc -> add snap-in Group Policy Management Editor -> Browse for Group Policy Object: Domain Controllers your.domain (double click, drill down) -> Default Domain Controller Policy -> Finish -> OK
- (drill into tree) Default Domain Controller Policy -> Computer Configuration -> Policies -> Windows Settings -> Security Settings -> Local Policies -> then select Security Options
- (signing) scroll to Domain controler: LDAP server signing requirements -> right-click and selct Properties
- (signing) check box Define this policy setting, set to Require signing, Apply/OK
- (channel binding) scroll to Domain controler: LDAP server channel binding token requirements -> right-click and selct Properties
- (channel binding) check box Define this policy setting, set to Always, Apply/OK
- do a gpupdate /force to get the policy enacted quickly -> open a PowerShell or cmd prompt and type `gpupdate /force`
- verify for yourself (and consider using Wireshark to see the negotiation in action)
	- from a non domain joined computer browsing on ldap should fail with Invalid Credentials or Strong Authentication Required.
	- from a domain joined computer ldap will still work as expected only if you use the currently logged in user using Windows authentication (kerberos) option in Softerra. Other methods that try to send passwords will fail.
	- from a non domain joined computer browsing on ldaps (TLS) will work as expected (tested with Softerra browser).
	- Similarly, ldap queries from Linux/Unix systems using ldapsearch will now fail.

---

ref:  
[AD CS with Web Enrollment configuration.md](../AD\%20CS/AD\%20CS\%20with\%20Web\%20Enrollment\%20configuration\%20.md) (my notes)  
[Obtain a certificate for use with Windows Servers and System Center Operations Manager](https://learn.microsoft.com/en-us/system-center/scom/obtain-certificate-windows-server-and-operations-manager?view=sc-om-2022&tabs=Enterp%2CEnter)  
[Enable LDAP over SSL with a third-party certification authority](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/enable-ldap-over-ssl-3rd-certification-authority)
https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/enable-ldap-signing-in-windows-server
[How to enable LDAP signing in Windows](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/enable-ldap-signing-in-windows-server#how-to-set-the-client-ldap-signing-requirement-by-using-a-domain-group-policy-object)
[Enable LDAP over SSL with a third-party certification authority](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/enable-ldap-over-ssl-3rd-certification-authority)
[ldaps certificates on domain controllers](https://xdot509.blog/2020/12/21/ldaps-domain-controller-certificates/)