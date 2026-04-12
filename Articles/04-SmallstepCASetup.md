# Smallstep CA Setup

In this article, we will build the Smallstep CA system and deploy it to provide a networked CA enabling remote enrollment and automatic renewal of our certificates.  

## Building step-ca

Smallstep provides pre-built binaries for their `step-ca` system.  Unfortunately, these binaries don't include PKCS11 support so we will have to build step-ca from source to use step-ca with our YubiKey based CA.  First we will need to install some packages:
```
sudo apt install libpcsclite-dev gcc make pkg-config golang
```
Then we will need to download the latest `step-ca` release.  You can get that from their GitHub repo [smallstep/certificates](https://github.com/smallstep/certificates/releases).  You'll want to download the source tarball, substitute the version number for whatever is current when you do this:
```
mkdir step
cd step
curl -LO https://github.com/smallstep/certificates/archive/refs/tags/v0.30.2.tar.gz
```
Put the tarball in a working directory and expand it:
```
tar xvzf v0.30.2.tar.gz
```
Then you'll need to get all the prerequisites for building `step-ca`.
```
make bootstrap
```
This will download a whole mess of packages from all over the place to do all the interesting things that `step-ca` can do.  It'll take a minute but when it's done you'll build `step-ca`:
```
make build GO_ENVS="CGO_ENABLED=1"
```
You should have a `bin` directory that wasn't there before and within that directory you'll have binary called `step-ca`.  You'll want to put this file someplace sensible where it can be used by various users so:
```
sudo cp bin/step-ca /usr/bin
```
Lastly, we'll need to allow the `step-ca` binary to bind to ports under 1000 even when it's run as a non-root user:
```
sudo setcap CAP_NET_BIND_SERVICE=+eip /usr/bin/step-ca
```

## Configuring step-ca

Now that we have the CA built we will also need the `step` CLI utility.  This is used for general certificate and PKI utility as well as a client for a `step-ca`.  You're welcome to build this from source as well if you like but the pre-packaged deb file will do just fine.  You can get it from the [smallstep/cli](https://github.com/smallstep/cli/releases) repo.  I'm assuming that you are on an AMD64 system but if you're on something else, get the appropriate version for your architecture. Download and install it, again check for the current version number in the future:
```
curl -LO https://dl.smallstep.com/gh-release/cli/gh-release-header/v0.30.2/step-cli_0.30.2-1_amd64.deb
sudo dpkg -i step-cli_0.30.2-1_amd64.deb
```
Eventually we'll want the `step-ca` to run as a systemd service.  To do that we'll need to make a user for the service called `step`.  We'll want to add this user to our `yubi` group that we created in the last article so that it can access our YubiKey:
```
sudo useradd step
sudo usermod -aG yubi step
```
For the next little bit it'll probably be easier to just run as root using a `sudo -s`.  We'll want to create a directory structure for `step-ca` to operate in.  
```
mkdir -p /etc/step/{certs,config,db,secrets,templates}
```
From here we'll copy our root and signing certificate into the /etc/step/certs folder so we have them named `root_ca.pem` and `intermediate_ca.pem`.  From here we need to create the CA configuration file `/etc/step/config/ca.json`:
```
{
   "root": "/etc/step/certs/root_ca.pem",
   "federatedRoots": null,
   "crt": "/etc/step/certs/intermediate_ca.pem",
   "key": "YubiKey:slot-id=9a",
   "kms": {
      "type": "YubiKey",
      "pin": "123456"
   },
   "address": ":443",
   "insecureAddress": "",
   "dnsNames": [
      "yubitest.freyjapki.com"
   ],
   "logger": {
      "format": "text"
   },
   "db": {
      "type": "badgerv2",
      "dataSource": "/etc/step/db",
      "badgerFileLoadingMode": ""
   }
}
```
You'll want to change the YubiKey PIN to whatever you set back when you setup your YubiKey.  We'll also need the fingerprint of your `root_ca.pem` which you could do using the method we used from Article 3 but now that we have `step` installed, we can use a much shorter command:
```
step certificate fingerprint /etc/step/certs/root_ca.pem
```
Copy the resulting fingerprint and substitute it into the following file saving it as `/etc/step/config/defaults.json`:
```
{
   "ca-url": "https://yubitest.freyjapki.com",
   "ca-config": "/etc/step/config/ca.json",
   "fingerprint": "59138e613920a1a24ad9826f3a4e9b29642962a1b86ab1f996c154ffe35eeac7",
   "root": "/etc/step/certs/root_ca.pem"
}
```
Next you can change the owner of the `/etc/step` directory to your step user:
```
chown -R step:step /etc/step
```
Change your user to step and create an environment variable to tell `step-ca` where to get its configuration:
```
sudo -su step
export STEPPATH=/etc/step
```
Now you should be able to run the `step-ca` command.  It should give some status messages as it's getting itself initialized and end with a line something like:
```
2025/06/14 21:13:11 Serving HTTPS on :443 ...
```

If so, that's great, you have a remote CA setup.  You can check that it's working properly on the network by going to another machine and running the command:
```
curl -ks https://yubitest.freyjapki.com/health
```
Note that you need the `-ks` flags to make `curl` ignore the fact that your other machine doesn't trust your self-signed root certificate yet. We're done being the step user now so type `exit` to return to your root user session.

### systemd Service Creation

Since we don't want our CA server to run in a terminal all the time, we need to make it run as part of a system service.  We do this by creating a systemd unit file and saving it in `/etc/systemd/system/step-ca.service`:
```
[Unit]
Description=step-ca
After=network-online.target pcscd.service
Wants=network-online.target pcscd.service
BindsTo=pcscd.service

[Service]
User=step
Group=step
ExecStart=/usr/bin/step-ca /etc/step/config/ca.json
Type=simple
Restart=on-failure
RestartSec=10
[Install]
WantedBy=multi-user.target
```
Then we will run:
```
systemctl daemon-reload
systemctl enable step-ca
systemctl start step-ca
systemctl status step-ca
```
That last command should produce a status report of `active (running)`.  You now have your CA server running as a systemd service that should start whenever the system does.

### Root CA Trust

Now is as good a time as any to add your root certificate to the system CA trust store.  This will make it trusted by various applications on your system just like any public CA.  We do this by copying the root CA into the `/usr/local/share/ca-certificates` directory but we have to give it a `.crt` extension then running the update command:
```
cp /etc/step/certs/root_ca.pem /usr/local/share/ca-certificates/root_ca.crt
update-ca-certificates
```
### Provisioner creation

At this point, you can run a server but you can't actually issue a certificate.  To do that you will need a provisioner.  Provisioners are `step-ca`'s method of accepting certificate signing requests and deciding whether to issue certificates for those requests.  There are a variety of different types of provisioners which have different strengths and weaknesses and are suitable for different uses.  We'll start by creating a default provisioner of the JWK type.  This uses a signed JWT (JSON Web Token) which is simply a set of data signed by a private key on the CA.  We submit our CSR along with the token, and the CA will sign and return our CSR as a certificate.  There are some subtleties to the implementation but if you think of it as a one time use password you won't go far wrong.  We can do all the setup with the following command:
```
step ca provisioner add default --type JWK --create
```
You'll be prompted to enter a password and the provisioner will be created.  To get the provisioner configuration to update you'll be prompted to reload `step-ca` with a `kill -1 PID` command.  That tends to create issues with the YubiKey connection.  Best to just restart the service, it takes 2 sec.
```
systemctl restart step-ca
```
You can check the currently active provisioners with the command:
```
step ca provisioner list
```

### Issue Your First Certificate

Finally, you have something useful.  You can issue a certificate to your own local machine.  
Let's make our lives easier by creating another variable for your domain name:
```
export DOMAIN=freyjapki.com
```
Next, we create a CSR:
```
step certificate create $(hostname).$DOMAIN $(hostname).csr $(hostname).key --insecure --no-password --csr 
```
This command creates a new CSR and private key file.  We specified no password to encrypt the private key since for unattended applications, that's almost always more trouble than it's worth but you can omit the `--insecure --no-password` options if you like.  

Next we create a token to authenticate to the CA:
```
TOKEN=$(step ca token $(hostname).$DOMAIN --provisioner default )
```
You will be prompted to enter the password you created to decrypt the provisioner key.  Enter it and press enter.  

You can inspect the contents of this token with the command:
```
echo $TOKEN|step crypto jwt inspect --insecure
```
You should see an output showing the data contained in the token.  Most of this isn't all that interesting.  Note that the token includes the subject that you specified on the token creation command, as well as an `exp` parameter.  This is a timestamp 5 minutes after the creation of the token.  This severely limits the window in which an attacker can use a token that has been left laying around in a system someplace.  Remember, for a token to be accepted, the names in the token and CSR must match exactly and it must be used only once, within 5 minutes of its creation.

Next we submit the CSR and token to the CA to get a signed certificate back:
```
step ca sign $(hostname).csr $(hostname).pem --token $TOKEN
```
You should now see a new certificate file which you can look at using:
```
openssl x509 -noout -text -in $(hostname).pem
```
You'll note that it expires in 24 hours so we'll want to renew that.  The good news is that, by default, as long as the certificate isn't revoked or expired, it can be automatically renewed without having to go get another token.  You do this with the following command:
```
step ca renew $(hostname).pem $(hostname).key 
```
You'll be asked to confirm that you want to overwrite the existing certificate.  Once that's done you can inspect the new certificate and note that it expires later than the original one. 

### Further step-ca Configuration

We're still missing a lot of little things.  Remember revocation and authority information access, we don't have that yet.  That's what we are doing next.  You'll need to create a file, `/etc/step/templates/default.tmpl` substituting your domain and CA name as needed:
```
{
   "subject": {{ toJson .Subject }},
   "sans": {{ toJson .SANs }},
{{- if typeIs "*rsa.PublicKey" .Insecure.CR.PublicKey }}
   "keyUsage": ["keyEncipherment", "digitalSignature"],
{{- else }}
   "keyUsage": ["digitalSignature"],
{{- end }}
   "extKeyUsage": ["serverAuth", "clientAuth"],
   "issuingCertificateURL": ["http://aia.freyjapki.com/FreyjaPKISigningCA.crt"],
   "crlDistributionPoints": ["http://aia.freyjapki.com/FreyjaPKISigningCA.crl"]
   
}
```
You will need to add this template file to your provisioner with:
```
step ca provisioner update default --x509-template /etc/step/templates/default.tmpl 
```
This is just the `step-ca` default template with two additional properties added for `issuingCertificateURL` and `crlDistributionPoints`.  After a quick restart of the `step-ca` service, all new certificates will have extensions added for Authority Information Access and CRL Distribution Points.  You can go back through the above process and test that it works.

Next we need to actually enable the CRL generation functionality on the CA.  To do that we will edit the `/etc/step/config/ca.json` file.  Add the following keys to the main body of the CA configuration, don't forget to substitute your domain names where needed:
```
   "insecureAddress": ":80",
   "crl": {
      "enabled": true,
      "cacheDuration": "864000s",
      "generateOnRevoke": true,
      "idpURL": "http://yubitest.freyjapki.com/1.0/crl"
   },
```
This tells `step-ca` to open up an insecure HTTP server on port 80, enable CRL generation, make the CRLs last for 10 days, generate a new CRL whenever a certificate is revoked, and to host CRLs on the `1.0/crl` path from that port.  We could set the CRL distribution point extension in our certificates to that URL and call it a day but we have a slight complication.  Since our private key is in the YubiKey if that were to suffer a hardware failure of any sort, we'd be unable to start our CA server and thus unable to host the CRLs.  Since we need an http server for AIA (and other stuff) anyway, I think it's better to copy the CRLs from the CA to another server and make them available there.  This has other advantages as well, including the ability to use proxies and such for high availability, and allowing you to more finely segment network access to the CA vs the AIA/CRL server.

#### Testing Revocation and CRLs

After restarting the CA server you should be able to test the CRL endpoint using `curl`:
```
curl http://yubitest.freyjapki.com/1.0/crl | openssl crl -noout -text
```
You should see a text output of a CRL which should indicate that no certificates have been revoked. To test out the revocation you can create a certificate called `test.pem` with key `test.key` using the above procedure then revoke it using the following command:
```
step ca revoke --cert test.pem --key test.key 
```
When you query the CRL endpoint again you should see the serial number of your test certificate as revoked.  If you attempt a renewal of the certificate you will get an error.

### Conclusion

Congratulations.  You have an online signing and renewing CA that can issue and revoke certificates for devices and users within your network.  We're not done yet, since there are still a number of extra pieces and automations to add to make the system complete.  In the next article we'll build out our authority information access and CRL server.  While we are at it, we'll learn how to deploy and automatically renew certificates to an Apache webserver. 

