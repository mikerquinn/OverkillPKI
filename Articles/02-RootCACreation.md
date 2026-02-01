# Part 2: Root CA Creation

Our PKI will be based on trust of a single offline root Certificate Authority (CA).  This CA Certificate will be spread around far and wide to be trusted by all the devices that interact with our PKI.  Its private key is the single most critical part of our PKI infrastructure, and compromise of that key would allow an attacker to create an unlimited number of certificates to do anything he wants with any system that trusts our root certificate.  This is the one key that rules them all, must be kept locked away, not to be used, except at the gravest end of need. 

## CA Creation Hardware

Your main concern is that the root CA private key be created and stored in a place where it can't be compromised but will be accessible for years in the future.  So, simply creating and storing the key on the main drive of your daily use workstation is the worst place you could put it.  If that workstation should ever become compromised due to phishing, hacking, or even physical theft, your entire network PKI will be compromised along with it.

To make your PKI reasonably safe, the simplest thing to do is to get a USB memory stick, create your root CA key on it, and then take the memory stick out and put it away.  Assuming that the workstation that you are using isn't currently compromised, that's probably safe enough.  

If you are more paranoid, you can create the key on an offline computer dedicated to the task, backup the key to removable media (optionally encrypted) and store it away.  Because the key was never physically attached to the internet, remotely compromising it is impossible.  Short of someone breaking in and stealing your key, you should be completely safe.

If you are even more paranoid than that, you can create the root CA on an offline machine, and import the key into multiple smartcards.  Then physically destroy the drive on which the root CA was created.  In this case, even if the smartcards are stolen, a thief would have to know their PIN codes to use the keys.  Since a typical smartcard locks itself out after a handful of incorrect PIN attempts, this renders the keys highly resistant to even a physical theft.

For our purposes, we will use the middle approach: creating the key on an offline machine, backing it up to multiple USB memory sticks, and storing them away.  This provides a good compromise of security and availability without spending hundreds of dollars on five or six Yubikeys that will do nothing but sit around in a safe for the next decade and quite possibly never get used again.

## CA Creation

Whichever combination of the above approaches you adopt, the process of creating the PKI is the same.  We will create a config file for OpenSSL, then apply that config file to create a key and Certificate Signing Request (CSR).  We will use that same key to sign the CSR, thus creating a self signed certificate.  That will be the root CA.  We can then take that certificate and distribute trust for it throughout our environment, while locking away the key using whatever scheme we selected.

### CA Config File

Much of OpenSSL's functionality can be accessed directly on the command line, however, more complex configurations generally benefit from the use of configuration files.  We will create one for our root CA.  The [full file](../configs/CAConfig.conf) is in the `configs` directory of OverkillPKI repository, but we will go through it section by section here.

For the rest of this article series I'll have my own domain and CA names used in config files and such.  You'll need to substitute your chosen names as appropriate.  Where practical, I try to use variables so it's easier to copy-paste commands and scripts for your own use.

##### We begin with the `[ default ]` section:

```
[ default ]
domain                  = freyjapki.com
rootca                  = FreyjaPKIRootCA 
signingca               = FreyjaPKISigningCA
dir                     = .              
pki_url                 = http://pki.$domain
aia_url                 = $pki_url/$rootca.cer
crl_url                 = $pki_url/$rootca.crl     
name_opt                = ca_default
```

This section defines variables that will be referred to throughout the rest of this config file.  You will want to customize these for your own use case.  We begin by defining the name of our domain, root CA, signing CA, and the directory in which we will create its database files.  We'll also define a URL where we will make information about our PKI available.  We'll deal with creating this website in a later article, but for now, just enter an address that you will be able to use to host a website later; this does not need to be publicly accessible on the internet.  We also define paths within that website where we will make our certificate available and where we will distribute our Certificate Revocation Lists (CRLs).  The last line simply defines how the CA will display certificates when signing them.

##### The next sections define the parameters we will use to create our key and CSR for the Root CA:

```
[ req ]
default_bits            = 2048                  
encrypt_key             = no                  
default_md              = sha256              
utf8                    = yes                
string_mask             = utf8only          
prompt                  = no               
distinguished_name      = ca_dn           
req_extensions          = ca_reqext      

[ ca_dn ]
commonName              = $rootca

[ ca_reqext ]
keyUsage                = critical,keyCertSign,cRLSign
basicConstraints        = critical,CA:true
subjectKeyIdentifier    = hash

```

In the `[ req ]` section we have defined that we will be using a 2048 bit key (the algorithm will automatically default to RSA).  That key will not be encrypted when it is stored on disk.  Our message digest (MD) is defined as sha256 which is currently (as of 2026) considered best practice.  We are going to deal with UTF-8 strings, which is the standard encoding pretty much everywhere and we won't prompt for information about the request when we run the `openssl req` command.  Lastly, we tell the `openssl req` command where to look for the Distinguished Name (DN) of our CA and where to look for the extensions.

The `[ ca_dn ]` section will define the DN for our CA.  If you want, you can get more involved, but a Common Name alone is enough to identify our CA in our small network environment.

Lastly, the `[ ca_reqext ]` section will define the extensions that we want on our root CA certificate.  These are deliberately minimal and broadly permissive.  We want our root certificate to be relatively unrestricted in what it can do, since we never know if, years in the future, we might have some unforeseen need.  Here, all we do is define the `keyUsage` to allow this certificate to sign other certificates and CRLs.  We designate that this certificate is, in fact, a CA, and we set the `subjectKeyIdentifier` as a hash.  This will automatically generate a subject key identifier when we create our root CA.  

##### The next section will define the relative paths and parameters for our CA it's a big one so we'll split it up a bit:
```
[ ca ]
default_ca              = root_ca               

[ root_ca ]
certificate             = $dir/$rootca/$rootca.pem  
private_key             = $dir/$rootca/private/$rootca.key  
new_certs_dir           = $dir/$rootca/certs           
serial                  = $dir/$rootca/db/$rootca.crt.srl 
crlnumber               = $dir/$rootca/db/$rootca.crl.srl
database                = $dir/$rootca/db/$rootca.db 
```
This first part defines the paths that the `openssl ca` command will use.  We tell it where to look for the CA certificate (which we haven't created yet), as well as the private key whose path we defined at the top of the file.  We tell `openssl ca` where to store the new certificates that it creates, as well as where to find the serial numbers for the certificates and CRLs.  Lastly, we give a path to the `openssl ca` database file, where it will keep track of the status of our certificates.

The next section defines the default parameters for the certificates that we will be signing with our CA.  Here, a number of parameters have to be set.  Many of these seem pedantic and make more sense in the context of a large scale production CA.  For the most part we will tell `openssl` not to second guess our design choices.  
```
unique_subject          = no                   
default_days            = 3653                 
default_md              = sha256               
policy                  = any_pol              
email_in_dn             = no                   
preserve                = no                   
name_opt                = $name_opt            
cert_opt                = ca_default           
copy_extensions         = copy                 
x509_extensions         = signing_ca_ext       
default_crl_days        = 3652                 

[ any_pol ]
domainComponent         = optional
countryName             = optional
stateOrProvinceName     = optional
localityName            = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = optional
emailAddress            = optional
```
Be default, `openssl ca` will not sign certificates that have the same subject as an existing certificate.  We turn that behavior off.  We then define that our CA will last 10 years.  Our default MD will be sha256, as opposed to the obsolete default, sha1.  The policy option refers to our `[ any_pol ]` section and defines what names our CA will accept and sign for, we make everything optional.  We are assuming that if someone is messing around with our root CA and private key, they know what they are doing and are doing it for a reason.  No sense second guessing them.  We don't put the email address in the DN of the root certificate.  We will preserve whatever order the DN parameters are passed into the CA.  The next two lines `name_opt` and `cert_opt` control how `openssl ca` displays certificate information to us when signing certificates.  By default we will copy whatever extensions from the CSR into the resulting certificate since we are going to assume that anyone signing with the root CA will have the good sense to check requests before approving them.

The next line assigns the default extensions that `openssl ca` will add to any request.  This refers to the `signing_ca_ext` section that we haven't defined yet.  We can manually override this using the `-extensions` flag to `openssl ca`.  We will also define that when we sign a CRL, it will be good for 10 years.  Finally we define the extensions that we will apply for CRLs.  These are defined in a `crl_ext` section that we will define later.

##### The next section deals with the extensions for our root CA

```
[ root_ca_ext ]
keyUsage                = critical,keyCertSign,cRLSign
basicConstraints        = critical,CA:true
subjectKeyIdentifier    = hash
authorityKeyIdentifier  = keyid:always
```
These mostly mirror the `req_ext` from above, however, we add an extension for `authorityKeyIdentifier`(AKI) this will make our certificate copy its AKI from the `subjectKeyIdentifier` of its issuer.  When we go to create our root CA we'll pass the `-extensions` flag to `openssl ca` to use these extensions instead of the default `signing_ca_ext`.

From here, the rest of the file deals with extensions and configurations that only apply to our signing CA so we'll defer explanation of those sections until the next article.

### Creating the CA Directory Structure and Database Files

Got all that?  Good.  If you've made it this far, you're finally ready to start building the CA.  We begin by setting up the directories that `openssl ca` will use to keep track of the certificates and key that we will use to create our PKI. 

You should begin with the `RootCA.conf` file in the base directory that your CA will live in.  We begin by creating an environment variable for the name of our CA.  This should be the `root_ca` name value from the top of our config file.
```
export CA=FreyjaPKIRootCA
```
Next we need to create paths for the database, certificate archive, and private key.  

```
mkdir -p $CA/{certs,db,private,crl}
```
We'll need to create serial number files to define the starting point for the certificate and CRL serial numbers.  We'll also create the certificate database file.
```
openssl rand -hex 16 > $CA/db/$CA.crt.srl
echo 1001 > $CA/db/$CA.crl.srl
touch $CA/db/$CA.db
```
We'll also set permissions on the private folder so that other users can't see it
```
chmod 700 $CA/private
```

### Create the Private Key, CSR, and Root CA Certificate

Now we have all the pieces in place to create our root certificate.  We begin by generating our private key and CSR files.
```
openssl req -new -config CAConfig.conf -out $CA.csr -keyout $CA/private/$CA.key
```

If you set `encrypt_key=yes` in the `[req]` section, you'll be prompted to provide a password to encrypt the private key.  It is critical that this password is strong and that you are able to remember/retrieve it later.  If you lose this password, the private key can't be decrypted and is effectively lost.  If you don't want to worry about this you can change the `encrypt_key` option in the `[ req ]` section to no and the key won't be encrypted.  However, this means that anyone who accesses the drive where the key is stored can read it freely.  You will need to decide what level of security/risk is right for you.  

Next, we create the actual root certificate itself:
```
openssl ca -selfsign -config CAConfig.conf -in $CA.csr -out $CA/$CA.pem -extensions root_ca_ext
```

You will be prompted to enter the password if you set one in the last step.  Then `openssl` will display information about the certificate you are about to sign.  Answer `y` to sign the certificate and then answer `y` again to commit the change to the database.

A whole bunch of things just happened all at once.  Openssl used your private key to sign the CSR that you created.  It put the resulting certificate in the directory that you indicated in the config file.  It also used the serial number file that you previously created to affix a serial number to the certificate.  It also saved the certificate to the certificate archive folder.  Finally, it added the serial number and current status to the certificate database file that you created above.  

## CRL Creation

We have one more step.  We must create an initial CRL.  This will initially be blank, but we still need to have one since some strict applications require it or else they will reject your certificates.  To do this, we will invoke the ca command again:
```
openssl ca -gencrl -config CAConfig.conf -out $CA/crl/$CA.crl
```
This generates a CRL which will be good for the duration specified in the config file.  It's in the CRL directory of the ca directory tree.  We'll use this file in a future article.

## Conclusion

At this point, you have created a lot of configuration to produce the private key and root certificate.  From here, you could just use this CA to start issuing certificates to your users and devices, but we aren't going to do things that way.  As discussed in the first article, we will build another layer under this certificate, the signing CA.  That will use a hardware protected key and will be the online issuer that will be used to create our actual certificates.  While we are at it we'll learn all about PKCS11 and Smallstep.  See you on the next one.

Copyright 2026 Michael Quinn
