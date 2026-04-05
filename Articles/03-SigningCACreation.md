# Part 3: Signing CA Creation

Now that we have a root CA, we need to create an issuing CA on our YubiKey to actually issue certificates to our devices and users.  The key will be created, stored, and used within the YubiKey itself.  This prevents any possibility that it can be exfiltrated short of physical compromise of the key itself.  We'll continue to use our `opensssl` config file from the last article and make use of the sections related to the signing CA creation.  
## YubiKey PKCS11 Setup

We begin by installing the required packages.  
```
sudo apt install yubikey-manager libengine-pkcs11-openssl ykcs11 gnutls-bin opensc-pkcs11
```
OpenSSL is the library that underpins the bulk of Linux's cryptography functionality.  For many years it has provided support for external cryptographic functions through modules known as engines.  The engine system is currently being phased out in favor of the new provider system that will be used going forward.  Unfortunately for us, the PKCS11 provider is not really stable and functional as of this writing (Early 2026).  As a result, we will use the old engine system.  Hopefully, in the future it will be possible to update this tutorial to use the more modern mechanism.

We have to configure `openssl` to use the PKCS11 engine.  To do this, we edit the system-wide `openssl.cnf` file.  By default this is located in `/etc/ssl/openssl.cnf`.  You'll need to add a couple of sections to the file.  It doesn't matter where these sections appear but it's probably easiest to append them to the end of the file.  The sections should look like this:
```
[engine_section]
pkcs11 = pkcs11_section

[pkcs11_section]
engine_id = pkcs11
dynamic_path = /usr/lib/x86_64-linux-gnu/engines-3/libpkcs11.so
MODULE_PATH = /usr/lib/x86_64-linux-gnu/libykcs11.so
init = 0
```
These sections define the name of the engine we are using; in this case, we are just calling it `pkcs11` and tell `openssl` where to look for the libraries that it will hand off the cryptographic operations to. In this case we direct `openssl` to use the `libpkcs11.so` which is the library that tells `openssl` how to talk PKCS11, and then setting the `MODULE_PATH` to the `libykcs11.so` which is the library that interfaces to the YubiKey.  Essentially, when we specify that `openssl` should use an engine, we have `openssl` do all of its normal operations except that when it comes time to sign, encrypt, or decrypt data it hands that operation off to the PKCS11 library to let the YubiKey hardware do it.

### Groups and Permissions For YubiKey Use

As of Ubuntu 24.04, access to the smartcard service is restricted to superusers by default.  When you run the command `ykman piv info`, you should get an error about the PC/SC service being unavailable.  We can change this by creating a polkit rule.  

You'll need to create a file `/etc/polkit-1/rules.d/99-pcscd.rules`.  This allows any users in the group `yubi` to use the smartcard service without having to be full-on superusers.
```
polkit.addRule(function(action, subject) {
    if (action.id == "org.debian.pcsc-lite.access_card" &&
        subject.isInGroup("yubi")) {
        return polkit.Result.YES;
    }
});
polkit.addRule(function(action, subject) {
    if (action.id == "org.debian.pcsc-lite.access_pcsc" &&
        subject.isInGroup("yubi")) {
        return polkit.Result.YES;
    }
});
```
We'll also need to create the `yubi` group and add our current user to it.
```
sudo groupadd yubi
sudo usermod -aG yubi $(whoami) 
```
Now when you run the command `ykman piv info` you should get information about the current state of the yubikey.

## YubiKey Key Creation

Next we'll need to generate a private key in the YubiKey to use for signing.  We'll start by resetting the YubiKey.  You can skip this if you already have a key setup or if the YubiKey is new and hasn't been used yet.
```
ykman piv reset
```
Next we will need to set a PIN and PIN unlock code for the YubiKey.  You must change these from their defaults, otherwise anyone who can access the key can use it to do anything they want.
```
ykman piv access change-pin
ykman piv access change-puk
```
After each command you will have to put in the current (default) and new PIN/PUK code.

Next, you will have to create a key on the YubiKey token and use that to generate a CSR.  We'll use a 2048-bit RSA key (which is the default) that will allow us to later use SCEP for device enrollments.

You'll want the name of your signing CA handy for the next steps.  For the rest of this article, all the commands will assume you have `$SIGNINGCA` set appropriately, don't forget to set it as you bounce back and forth between systems.
```
export SIGNINGCA=FreyjaPKISigningCA
```
```
ykman piv keys generate 9a 9a.pub --pin-policy always
```
This tells the YubiKey to create a new key in slot 9a and save the resultant public key in a file, `9a.pub`.  The next step is a little convoluted.  We tell the YubiKey to create a self-signed certificate with the same name we are going to use for our signing certificate.  We won't actually use this certificate for anything but many of the subsequent commands will work better with a certificate in the slot than they will with just the public key.
```
ykman piv certificates generate -s "CN=$SIGNINGCA" 9a 9a.pub
```
You can confirm that the certificate has been created with the command `ykman piv info`.  This should list the certificates on the YubiKey.

## Creating the CSR

### Locating the private key

Next we need to create a CSR using our newly generated key on the YubiKey.  To do this, first, we have to tell `openssl` where our key is.  Unfortunately, `openssl` can't address keys by their YubiKey slot names.  Instead we have to get the pkcs11 URI for our private key.  This is a bit of a process.  To do this we will use the p11 tool we installed earlier.  Execute the command:
```
p11tool --list-tokens
```
This should list one or more pkcs11 tokens; depending on your system, there may be several.  You're looking for the one labeled with the name of your certificate.  It should look something like this:
```
Token 1:
   URL: pkcs11:model=PKCS%2315%20emulated;manufacturer=piv_II;serial=909b27839ec26c65;token=FreyjaPKISigningCA
   Label: FreyjaPKISigningCA
   Type: Hardware token
   Flags: RNG, Requires login
   Manufacturer: piv_II
   Model: PKCS#15 emulated
   Serial: 909b27839ec26c65
   Module: opensc-pkcs11.so
```
You'll want to copy the pkcs11 token URI including the leading `pkcs11:` all the way to the end with your signing CA name.  Make your life easier in the next step by setting that to a shell variable:
```
export TOKEN="<paste pkcs11 token URI here>"
```
Then you will need to get the private key URI from the YubiKey with the following command:
```
p11tool --list-keys --login $TOKEN
```
You'll be prompted to enter your PIN and then you'll get something like this:
```
Token 'FreyjaPKISigningCA' with URL 'pkcs11:model=PKCS%2315%20emulated;manufacturer=piv_II;serial=909b27839ec26c65;token=FreyjaPKISigningCA' requires user PIN
Enter PIN: 
Object 0:
   URL: pkcs11:model=PKCS%2315%20emulated;manufacturer=piv_II;serial=909b27839ec26c65;token=FreyjaPKISigningCA;id=%01;object=PIV%20AUTH%20key;type=private
   Type: Private key (RSA-2048)
   Label: PIV AUTH key
   Flags: CKA_WRAP/UNWRAP; CKA_PRIVATE; CKA_NEVER_EXTRACTABLE; CKA_SENSITIVE; 
   ID: 01
```
The thing you're looking for is the `id=%01` in the pkcs11 URL.  I've always seen slot 9a have id 01 but as far as I know that's not a guarantee so do check before continuing.  This is the identifier that you will pass to `openssl` to get it to use the key on the YubiKey to sign the CSR.

### CSR config file

Next, copy in the same `openssl` config file that you used to create the root and let's take a look at it.  Start with the section `[ signing_req ]`:
``` 
[ signing_req ]
default_bits            = 2048                  
encrypt_key             = yes                  
default_md              = sha256              
utf8                    = yes                
string_mask             = utf8only          
prompt                  = no               
distinguished_name      = signing_ca_dn           
req_extensions          = signing_ca_ext      

[ signing_ca_dn ]
commonName              = $signingca 
```
This should look familiar from the request file from the root CA.  It refers to the subsequent sections, `signing_ca_dn` and `signing_ca_ext`.  The next section is where we do some new things:
```
[ signing_ca_ext ]
keyUsage                = critical,keyCertSign,cRLSign
basicConstraints        = critical,CA:true,pathlen:0
authorityInfoAccess     = @aia_info
crlDistributionPoints   = @crl_info
nameConstraints         = critical,@name_constraints
```
This section is similar to the `root_ca_ext` section above but with a few extra features.  We add a basic constraints parameter specifying `pathlen:0`.  This means that a valid certificate chain can contain no additional CAs after this one.  Thus, if someone were to try to issue another CA with the signing CA and then issue end-entity certificates with that, the certificates would be invalid.  

We also set the authority information access and CRL distribution points extensions.  These will tell people where to get the root CA associated with the signing CA as well as where to get CRLs associated with them.  Note that these are http URLs, not https.

The last line of the `signing_ca_ext` section points to the `name_constraints` section which has a number of rules in it:
```
[ name_constraints ]
permitted;DNS.0=$domain
permitted;URI.0=
permitted;email.0=$domain
excluded;IP.0=0.0.0.0/0.0.0.0
excluded;IP.1=0:0:0:0:0:0:0:0/0:0:0:0:0:0:0:0
```
This section is important.  Name constraints are an underutilized and poorly documented extension to the X509 certificate specification.  There are a number of rules for how name constraints work.  The details are best learned by reading [RFC5280 Section 4.2.1.10](https://datatracker.ietf.org/doc/html/rfc5280#section-4.2.1.10).  

In this case, we are defining that the CA can only sign for DNS names in our private domain, email addresses in our domain, and nothing else.  We specifically exclude all IP addresses and we make the set of allowed URIs empty.  

For a long time, name constraints were seldom used because most browsers and applications didn't support them.  That is no longer the case.  All major browsers and operating systems now support name constraints and they can be an important tool to limit the damage that can be done in the event of a CA compromise. In this case, someone with access to the CA could issue certificates for things in our own network, and that is bad, but they couldn't issue for other things like banking or external mail servers.  That's a worthwhile improvement in security.

Finally, we define the extensions and publishing information for our Certificate Revocation List and Authority Information Access:
```
[ crl_ext ]
authorityKeyIdentifier  = keyid:always
authorityInfoAccess     = @aia_info

[ aia_info ]
caIssuers;URI.0         = $aia_url

[ crl_info ]
URI.0                   = $crl_url
```

### Generating the CSR

Finally, we are ready to create the certificate signing request.  Putting all of the above together we execute the following command:
```
openssl req -new -keyform engine -engine pkcs11 -config CAConfig.conf -section signing_req -out $SIGNINGCA.csr -key "pkcs11:id=%01"
```
You should be prompted for the PIN and upon entering it, a CSR file should be generated.

## Signing the CSR

The resulting CSR file now needs to be transferred to your root CA.  Note that this is the only file you need to transport.  You don't need to, and should not, transfer the private key file.  Copy that CSR file into the root directory of your root CA directory structure.  Then you'll sign the CSR using the `openssl ca` command:
```
openssl ca -config CAConfig.conf -in $SIGNINGCA.csr -out $SIGNINGCA.pem -notext
```
You should get some information about the certificate and a request to confirm that what you see is what you want to sign and a confirmation to commit to the certificate database.  Approve both and you will have a new file `FreyjaPKISigningCA.pem`.  You can inspect this certificate with the following command:
```
openssl x509 -noout -text -in $SIGNINGCA.pem
```
You should see something like the following:
```
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            4f:01:d3:86:ad:28:ad:7a:10:61:02:88:06:3a:a5:22
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = US, O = Freyjapki, OU = Freyjapki Root CA, CN = Freyjapki Root CA
        Validity
            Not Before: Jun 14 14:21:05 2025 GMT
            Not After : Jun 15 14:21:05 2035 GMT
        Subject: C = US, O = Freyjapki, OU = Freyjapki Signing CA, CN = Freyjapki Signing CA
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:c3:c6:3a:78:f7:e5:44:9f:9a:b2:ba:d5:95:cf:
                    39:12:6c:b5:11:3c:d2:23:c0:22:9f:89:a2:4a:ea:
                    45:7a:97:af:5d:5f:e5:28:75:67:2c:5d:14:47:56:
                    7a:88:64:9d:42:f7:fe:1c:21:83:7e:9b:98:c7:ec:
                    aa:0d:0b:da:97:43:e5:f6:a9:b4:24:85:82:4e:a8:
                    83:59:c4:1c:c1:d6:24:e5:50:54:4d:f9:f4:82:a6:
                    ac:75:2f:5f:a5:38:1a:1e:2a:dd:00:43:9e:e1:8e:
                    35:1c:d8:99:01:d3:fc:14:0b:1a:0d:7d:66:c7:70:
                    61:01:7e:c8:5d:a6:5a:c4:16:83:28:a3:e4:03:49:
                    ac:e5:b1:7a:64:42:14:91:32:57:16:71:54:b7:1b:
                    eb:23:20:63:d2:76:3e:37:59:9b:55:fd:f8:e5:1f:
                    6b:dc:ea:f2:a7:48:de:db:56:c6:37:42:07:4e:c9:
                    5c:45:19:8b:9e:85:c8:d9:57:15:bf:83:75:a9:bf:
                    fc:8b:f3:9d:df:76:8a:b7:e4:6d:9c:86:b9:d2:d3:
                    06:10:70:79:bf:30:4b:a5:fa:36:cf:7b:86:77:3d:
                    e7:21:c5:51:51:57:31:e7:ff:07:b1:c1:4a:29:2c:
                    20:c1:78:66:d5:81:9d:7b:9f:dd:68:80:7d:cc:a9:
                    22:25
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Certificate Sign, CRL Sign
            X509v3 Basic Constraints: critical
                CA:TRUE, pathlen:0
            Authority Information Access: 
                CA Issuers - URI:http://aia.freyjapki.com/FreyjaPKIRootCA.cer
            X509v3 CRL Distribution Points: 
                Full Name:
                  URI:http://aia.freyjapki.com/FreyjaPKIRootCA.crl
            X509v3 Name Constraints: critical
                Permitted:
                  DNS:freyjagsd.com
                  URI:
                  email:freyjagsd.com
                Excluded:
                  IP:0.0.0.0/0.0.0.0
                  IP:0:0:0:0:0:0:0/0:0:0:0:0:0:0:0
            X509v3 Subject Key Identifier: 
                08:65:AE:1F:7D:7C:A1:7F:AF:8F:73:CE:97:D4:0A:1E:CD:49:31:7F
            X509v3 Authority Key Identifier: 
                56:07:04:45:9C:3A:57:42:FD:32:60:2C:B1:74:A2:A9:19:5E:31:3A
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        91:cf:8f:11:db:6f:c2:03:ce:2e:98:8c:78:5b:e2:b5:09:04:
        40:94:b0:7e:cf:fd:6e:3f:60:6c:04:49:b4:d8:db:26:4f:11:
        c1:48:9a:dd:19:e9:d8:a9:15:52:30:5a:f8:53:39:48:92:3f:
        cf:47:2d:3f:78:19:54:20:38:f2:8d:dc:3d:80:a3:45:69:6b:
        66:6c:0a:f0:a0:ab:21:2a:bb:42:8b:c2:4b:2c:22:e8:53:43:
        08:a7:4f:79:3e:b4:f4:05:66:fd:b1:ed:75:03:70:eb:3d:26:
        fd:2d:4d:d8:22:49:25:4d:3b:be:04:26:0d:99:ac:c0:6e:1b:
        e5:89:eb:05:60:91:42:8f:f8:86:1e:34:25:68:a3:98:30:d9:
        8e:12:ff:9d:7a:d0:ce:77:73:84:38:77:af:fe:69:09:f8:07:
        f8:3d:a6:7b:d7:37:37:db:05:8d:c8:ec:f6:70:e0:52:ae:c5:
        aa:91:97:49:d6:01:ec:23:5f:1a:76:a5:45:b9:0b:cb:ee:50:
        d3:38:54:35:de:c8:08:ed:1c:83:95:03:d6:c4:e4:f6:e9:61:
        50:0b:69:58:67:ce:65:ea:fc:c8:db:e5:6d:fe:ea:04:c6:80:
        d6:c3:f3:60:0f:95:a4:aa:9f:79:a0:26:0a:85:a5:d5:bf:e6:
        04:1f:fa:e2
```
Copy that file back to where you have your signing CA set up.  

## YubiKey Certificate Import

Now that you have your signing certificate, you need to import it to the YubiKey, replacing the previous self-signed certificate, so that it can be used for signing.  This is quite easily done using `ykman`:
```
ykman piv certificates import 9a $SIGNINGCA.pem
```
To confirm that the import was successful, run:
```
ykman piv info
```
Check the fingerprint against the fingerprint of your certificate using `openssl` by running:
```
openssl x509 -in $SIGNINGCA.pem -outform DER | openssl dgst -sha256
```
This converts your text-based PEM file into a binary DER file and hashes it using sha256.  This hash should match the fingerprint produced by `ykman` above.  

Congratulations, you have a two-layer PKI system suitable for signing leaf certificates.  From here you could set-up another CA directory structure and go about signing certificates for your clients and servers throughout your network using similar config files and `openssl ca` commands to the above.  However, that will be quite tedious.  Instead we will use another open-source system, Smallstep, which will be the subject of the next article.
