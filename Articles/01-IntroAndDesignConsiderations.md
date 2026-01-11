# Part 1: Introduction and Design 

## Introduction

If you're reading this article, there's a good chance that your small network is a jumble of WiFi passwords, certificate errors, and SSH security warnings.  The good news is, you can fix all of that.  You just need to spend a few days writing config files for arcane utilities, stringing together commands, and tiptoeing around a minefield of security disasters just waiting to happen.  There's a host of considerations, key generation, storage, access control, and automation.  This article series isn't a shortcut or quick guide.  This series aims to provide a complete tutorial on how to min-max cost and security in a PKI system on a modern (2026) small network.  I'm not going to even attempt to justify this project as necessary for home or even most small business networks.  The article series is called "Overkill PKI" for a reason.  But that's not the point.  This is meant to be as much educational as practical, so let's dig right in.

## Objectives

We are trying to create a system to secure a small network using public key cryptography.  We'll do this with a three-layer PKI system, an offline root CA, an online signing CA, and various end-entity certificates.   While we will primarily use our internal PKI, in some later articles, we may use publicly trusted certificates from Let's Encrypt as well.  There are many ways to accomplish this, and many of them have subtle security issues that you need to watch out for.

## Requirements

To build our PKI system, we'll need two things, somewhere to store the keys and somewhere to host the software that signs certificates.  
First, let's address the problem of where to store our keys.

### Key Storage

#### Option 1: No extra cost, just a plain old computer

For a bare minimum system we can simply use a regular computer and store our keys on disk.  We can use permissions and good passwords to keep unauthorized users out, and with proper maintenance that can be okay.  However, these are still just files and if an intruder were to access our system, by any means, there would be nothing to stop them from copying them, taking them offline, and using them to produce an unlimited number of impersonation certificates with which they could run amok and be very difficult to detect.  Is it possible to do better?

#### Option 2: Spare no expense, networked Hardware Security Module

Yes!  We could store our keys in a networked Hardware Security Module (HSM).  These are amazing specialty devices with amazing specialty price tags.  They can generate keys and perform cryptographic operations at blinding speed, internally.  They utilize tamper resistant hardware and specially hardened operating system software that make it virtually impossible to extract keys even with physical possession of the HSM itself.  To be safe, we'll need two HSMs so we have high availability, an offline backup module to store our keys, so they can be recovered in case of disaster, management software, licensing, and enterprise support for all of the above.  There are only two companies that really deal in such systems, Thales, and Entrust.  They don't publish pricing (an infuriating habit among enterprise hardware providers) but a fair guess based on my experience, is all that will run a bit over $100,000.  So, that's not going to work for our small network educational project. 

#### Option 3: The YubiKey 

There is a third option, smartcards.  We'll use YubiKey brand smartcards which are intended for use as authentication tokens but we can repurpose them as mini-HSMs.  Sorta, there are some serious limitations.  Keys can be generated on a YubiKey and cannot be exported so even if an intruder were to gain physical or network access to the YubiKey they couldn't copy the key and take it with them.  However, the encryption and signing operations on a YubiKey can be painfully slow.  Where a network HSM might do 10,000 signatures in a second, a YubiKey might take several seconds to do one signature.

In our small environment this isn't much of a problem though.  At the most, we might sign dozens of certificates in a day, so a couple seconds each is okay.  Having a YubiKey store our keys provides a level of physical security that we could never achieve with regular files.  Additionally, the YubiKey is protected by a PIN and will lock itself out after a very limited number (by default 3) of incorrect PIN attempts.  So even if someone were to steal the YubiKey token itself, they would find it impossible to use.   

The other limitation that we will stumble over is the fact without the ability to export keys, it's also impossible to backup keys.  If a key is generated in a YubiKey, and that YubiKey is lost or suffers hardware failure, the key is not recoverable.  We can mitigate this, but it requires careful planning.  The good news though is that YubiKeys can be purchased from Amazon for around $50 so that's a manageable price even for a hobby project.    

##### Option 3a: YubiHSM2

There is a related option to the YubiKey, the YubiHSM2.  This is a sort of hybrid device faster than a regular YubiKey and possessing the ability to securely backup one YubiHSM2 onto another.  They also feature secure logging allowing accounting for all commands that they execute.  These are serious advantages, but they come at a nontrivial cost, $650 each.  Again, if you want to be safe with them you'll need at least two on hand and a third as an off-site backup.  $1950 is a lot better than $100k but still not really viable for a home or hobby project.  However, much of what we will learn in this article series will apply directly to the YubiHSM2 (or the Thales and Entrust devices for that mater) so if you are in a small to medium sized enterprise environment the YubiHSM2 will be worth considering and following this tutorial will teach you many concepts and tools that will directly apply to more expensive options.  

## Server Requirements
At a minimum, you'll need a computer that can act as a server which can have a USB device plugged into it.  You can make this a virtual machine using USB passthrough.  Ideally, you'd want to have a virtual machine host that can provide segmented access to multiple virtual machines/containers, but in a pinch, a single Raspberry Pi could do the job.  Nothing that we are doing is remotely computationally intensive, so no need to worry about loading it up with CPU and RAM resources.  

All of the machines in our environment will be running Linux.  The instructions will be given for Ubuntu Server 24.04LTS but they should be adaptable for other distributions.  For PKI software we will use a combination of tools.  

We'll build our root and signing CAs with OpenSSL.  The actual signing of leaf certificates for both TLS and SSH will be accomplished using Smallstep's step-ca utility and server.  A log of certificate issuance and status will be stored in an internal database, however, an external PostgreSQL database can be used if desired.  Our authority information access information and CRLs will be hosted on Apache Webserver.  

All of these are free, open source projects that are available online.  In fact, you almost certainly have OpenSSL installed already.

## Feature Requirements

It's not enough that we just create certificates.  We also have to manage them, renew them, and potentially revoke them.  Since we don't want to spend our days and nights renewing and worrying about certificate expiration, we want to automate as much of that process as possible.

### Online Signing and Renewal

We want the ability for our various devices to be able to communicate directly with the CA, authenticate themselves, and either enroll or renew their certificate as needed.  Smallstep's step-ca offers a variety of options for automated enrollment including ACME, SCEP, and various semi-custom options based on JWK and OIDC tokens.  We'll address these in a future article.

### Revocation

We need to be able to revoke any certificate we issue (except the root, which is impossible to revoke).  In an environment as small as the one we are designing for, CRLs are perfectly fine.  After all, how many certs are you really likely to be revoking?  So if we need to revoke the signing CA, we can simply pull out the root CA key and generate a CRL from it.  However, this becomes more tricky when discussing revocation of leaf certificates from the signing certificate stored in the YubiKey.  If the YubiKey fails, it will be unable to sign any further CRLs.  We will have to be very careful about our CRL lifetime and where we host the CRLs.  

## Authority Information Access and CRL Distribution Points

We will need to provide our applications with an accessible way of getting the root and signing certificates, as well as a location to download the CRLs.  This will be a simple Apache Webserver.  It will serve http requests since that is what is specified by the standards documents.  Also it will be setup with a publicly trusted certificate from Let's Encrypt.  This can be used to distribute our root CA in a secure way to most devices that already trust Let's Encrypt's root CA (essentially all modern browsers and OSes).  Basically using the built in trust for Let's Encrypt to bootstrap trust for our certificate authority.

## SSH PKI

We will also create a PKI system for SSH access.  We'll use this to facilitate remote access to devices in our network.  We will also explore ways to leverage it to publish CRLs to our CRL distribution point server. 

## Disaster Recovery

We'd be remiss if we built our network PKI system without considering attack vectors, recovery plans, and potential vulnerabilities.  

### Root CA Compromise

This is the disaster scenario.  There is no recovery.  If an attacker gains access to the root CA private key, the only response is to manually remove the trust for the root CA from all of our endpoints, servers, and services and rebuild the entire PKI from scratch.  If an intruder has penetrated this deeply into the system, there is virtually no limit to the damage that they could do, and remediation for such a disaster is well beyond the scope of this series.  However, this is a fairly remote possibility if the root CA keys are kept truly offline and never allowed to touch any network-connected system.

### Signing CA Compromise

You may wonder how someone could compromise the signing CA when the key is stored in hardware from which it can't be exported.  That's a fair point, they can't copy the key and take it with them.  However, a threat actor could gain access to the server (VM or host) that the YubiKey is plugged into.  In that case, they could potentially use the key right where it is to sign arbitrary certificates.  If this happens, it will be almost impossible to know what certificates have been signed.  As such, the only response is to use the offline root CA to revoke the signing CA, and issue a new one.  

### Leaf Certificate Compromise

This is, by far, the most likely form of compromise that you will have to deal with.  It's also the easiest to remedy.  Simply revoke the certificate and publish that revocation through the CRL distribution point.  This sort of revocation is a normal part of CA operations and will be covered in a future article.  However, revocation checking on the part of applications is imperfectly implemented at best.  One of the design principals we will use in our PKI system is to issue very short lived certificates and automate their renewal.  This way, if a key is compromised, there is a very limited time window in which it can be used to access our systems.  For this reason, we will automate absolutely everything we can with our PKI system.

## Applications

Once we've killed a couple of weekends building out all that PKI infrastructure, it'd be a shame if we didn't use it for something.  We'll explore how to use our PKI to issue user and device certificates to MacOS and iOS devices with SCEP and ACME profiles.  We will use these certificates to authenticate to a FreeRADIUS backed EAP-TLS based WiFi and wired network.  PKI is a big security hammer, you'd be surprised to find just how many nails there are out that you can hit with it.

## Conclusion

We've defined the objectives for this project, and there are a lot of them.  In our next article, we will dive right into creating the root certificate authority.  


Copyright 2026 Michael Quinn
