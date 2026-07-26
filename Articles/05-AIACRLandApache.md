# AIA, CRL, and Apache

When we created our signing CA, we added a couple URLs for authority information access and certificate revocation list distribution points.  In this article, we will create a webserver, which I'll just call the PKI server, and populate it with our CA certificates and CRLs.  These are just HTTP endpoints, so any webserver will do, but for this tutorial, I've elected to use Apache.  We begin by installing and configuring it:

## Apache Install and Configuration

To configure the PKI server we'll need a separate server/VM/container from the one that hosts our CA.  You'll need to make sure that your network's DNS will resolve it to the `pki_url` that you specified in the `openssl` config file from part 2.  Once you have this up and running, go ahead and become root on the PKI server and install Apache webserver with the default package:
```
apt install apache2
```
You should immediately be able to point a browser to your PKI server and be greeted with the standard Ubuntu/Apache "It works!" page.

Next, you'll need to copy your root and signing CAs to the PKI server and put them in the `/var/www/html` directory.  This will make them accessible through the web server.  However, if you've been following along thus far, these files are in the `.pem` format which is the most common format since it's so convenient for things like email and copy and paste between windows etc.  However, the RFC for certificates specifies that certificates at AIA endpoints should be in the binary DER format, so we have to convert them with these commands run from the `/var/www/html` directory after setting your environment variables for your CA names.  
```
openssl x509 -outform DER -in $ROOTCA.pem -out $ROOTCA.crt
openssl x509 -outform DER -in $SIGNINGCA.pem -out $SIGNINGCA.crt
```
Make sure that the filenames match those in the authority information access extension of your signing and leaf certificates.  Now applications that want to access your AIA information can do so via the URL in the certificate.  

You'll also want to copy in that `$CA.crl` file that we created back in article 2 and put that in the `/var/www/html` directory as well, so that CRLs are available in case you need to revoke your signing CA.

### Apache Certificate Enrollment

Astute readers will have noticed that when we set the AIA URL, we did so using an `http://` URL not `https://`.  This might seem odd given that we now have the ability to sign for servers and such.  However, if someone doesn't have a copy of the root CA, they can't verify our AIA endpoint using that CA.  So, we're still stuck with the problem of how to distribute trust in the root CA.  Unfortunately, there's just not much we can do about that.  Depending on your environment, you might be able to push root CA trust using Windows GPO or some MDM solution for mobile devices, but in the end there is no standard way to do this.  Nevertheless, since we are going to use this PKI server for other things besides AIA and root CA distribution, we will find it useful to configure Apache to operate using TLS when it is available.  To that end, we will have to enroll our Apache server with our Smallstep CA.  To do this, we begin by logging into our PKI server and creating the necessary directory structure:
```
mkdir -p /etc/step/{certs,config}
```
Then copy your `defaults.json` from your CA server to your PKI server.  It should go in the `/etc/step/config` directory.  Also, put your root and issuing CAs in the `/etc/step/certs/root_ca.pem` and `/etc/step/certs/intermediate_ca.pem` paths as well.

We'll need to install the `step-cli` package:
```
curl -LO https://dl.smallstep.com/gh-release/cli/gh-release-header/v0.28.6/step-cli_0.28.6-1_amd64.deb
dpkg -i step-cli_0.28.6-1_amd64.deb
```
You'll want to set your `STEPPATH` environment variable to `/etc/step`.  Probably a good idea to put that in your `/etc/bash.bashrc` so that it's automatically done for you.
If you're reading this in the future it's likely the current version has changed and you'll want to update the commands accordingly.

Next we go to the `/etc/step/certs` directory and create a CSR:
```
step certificate create $(hostname).freyjapki.com ssl-cert.csr ssl-cert.key --csr --insecure --no-password
```
Next, we'll need a token from the CA.  To get that, we login to the CA server and run the following command, substituting values as appropriate:
```
step ca token pki.freyjapki.com --provisioner default 
```
You'll be prompted for the password; enter it, and you will see a long string of gibberish output to your terminal; this is your enrollment token.  Easiest thing to do at this point is just copy that to your clipboard and go back over to your PKI server in the `/etc/step/certs` directory and run this command:
```
step ca sign ssl-cert.csr ssl-cert.pem --token "<paste token text here>"
```
You should now have a certificate in `/etc/step/certs/ssl-cert.pem`.  You can inspect it:
```
step certificate inspect ssl-cert.pem
```
Congratulations, you have a certificate for your webserver.  

At the moment, the key file `ssl-cert.key` is only readable by root.  We aren't running Apache as root so we will need to do some permissions adjustments.  I find that the best way to deal with this is to create a group on the server called `ssl-cert` and add the user accounts for any services that might need access to the key to that group.  Then set the ownership of the `ssl-cert.key` file to `root:ssl-cert` and the permissions to `640`.  
```
groupadd ssl-cert
usermod -aG ssl-cert www-data
chown root:ssl-cert /etc/step/certs/*
chmod 640 /etc/step/certs/ssl-cert.key
```

### Apache TLS Configuration

Now that we have a certificate, we need to configure Apache to actually use it.  With the standard Apache packages provided by Ubuntu, SSL is not enabled by default in Apache.  We will need to enable the SSL module and restart Apache with the commands:
```
a2enmod ssl
systemctl restart apache2
```
We will also to enable the baseline configuration for a website using the SSL module so run the following command:
```
a2ensite default-ssl
```
Next, we will need to edit the file `/etc/apache2/sites-enabled/default-ssl.conf`.  You are looking for two lines; they should begin with `SSLCertificateFile` and `SSLCertificateKeyFile`.  Edit these two lines to point to your certificate and key like this:
```
SSLCertificateFile      /etc/step/certs/ssl-cert.pem
SSLCertificateKeyFile   /etc/step/certs/ssl-cert.key
```
Then you can reload the Apache configuration using:
```
systemctl reload apache2
systemctl status apache2
```
You should see a healthy service running.  You'll be able to point your browser at your PKI server using either `http` or `https` and get the Ubuntu "it works!" page.


### Automatic renewal

One little caveat, the certificate you created will expire (by default in 24 hours).  So you're going to want to automate the renewals of that cert.  Fortunately, `step` has the tooling to do that with.  Run the following command to renew your certificate on demand from the `/etc/step/certs/` directory:
```
step ca renew ssl-cert.pem ssl-cert.key --force
```
This will manually renew the certificate one time and overwrite the existing certificate.  Since we don't want to login to the webserver every day and run that command, we'll need to setup a renewal job with systemd.  Create the following file and save it as `/etc/systemd/system/step-renew.service`:
```
[Unit]
Description=Step TLS Renewer 
After=network.target
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=1
User=root
Environment="STEPPATH=/etc/step"
ExecStart=/usr/bin/step ca renew /etc/step/certs/ssl-cert.pem /etc/step/certs/ssl-cert.key --exec /etc/step/renew.sh --daemon

[Install]
WantedBy=multi-user.target
```
This is similar to the systemd unit file that we used to run the `step-ca` with, but this time it's running the `step renew` command with some additional flags.  We added the `--exec /etc/step/renew.sh` flag, which will run a script after each time the certificate renews.  We also added the `--daemon` flag, this causes `step` to run continuously monitoring the time till expiration for the certificate and when it has 1/3 of its life left, renew it and run the `--exec` script.  We are running this service as root since it will need to be able to run various `systemctl` commands in the post renew script that mostly require root to operate.  Since this service is not listening on any ports or accepting any external input at all (other than from the CA) the security risks of this are minimal.  However, if you wanted, you could probably run it as a lesser user and grant it just enough permissions using a `sudoers` file.  Personally, I think that's more trouble than it's worth.  

### Automatic rebinding

Simply putting a new certificate on the server is usually not enough, we also need to tell our various applications to use it.  Usually this involves a reload and/or restart command applied to the various services on the machine that use TLS.  In this case, since we are just running Apache webserver, we will save the following script in `/etc/step/renew.sh`:
```
#!/bin/bash
systemctl reload apache2
```
Give it executable permissions, and run:
```
systemctl daemon-reload
systemctl enable step-renew
systemctl start step-renew
systemctl status step-renew
```
The status should show the service running and tell you how long till the next renewal.

### CRL Updates

We set a CRL distribution point on this server but we need to actually make the CRLs available, we can do this with a simple script that pulls the current CRL from the API endpoint, verifies it, and copies it to the `/var/www/html` directory to publish it to the world.  Create the following script with executable permissions in a convenient place; I like to use `/usr/local/bin/crlupdater.sh`:
```
#!/bin/bash
USAGE="$0 -s <signing CA path> -c <CA Host address> -o <output CRL path>"
while getopts "s:c:o:" opt; do
   case $opt in
      s)
         SIGNINGCA=$OPTARG
         ;;
      c)
         CAHOST=$OPTARG
         ;;
      o)
         CRLPATH=$OPTARG
         ;;
      ?)
         echo $USAGE
         exit 1
         ;;
   esac
done

CRL=$(curl -s "http://$CAHOST/1.0/crl"|openssl crl -outform PEM)
if openssl crl -verify -CAfile $SIGNINGCA -noout <<<$CRL; then
   openssl crl -outform DER -out $CRLPATH <<<$CRL
   exit 0
fi
exit 1
```
This script takes three options, for the path to the signing CA, the address of the CA server, and the path to output the verified CRL to.  We have to verify the CRL since it comes from an `http` URL not `https` when we get a CRL we don't know that it's legitimate.  Fortunately, the CRL is signed by the CA that issued it and we have a copy of that CA on our server already.  The script verifies the CRL is legitimate and copies it to the specified path.  

Next, we need to have this script run on a regular schedule, the simplest way to do that is a cron job.  Save the following file in `/etc/cron.d/crlupdater`:
```
* * * * * root /usr/local/bin/crlupdater.sh -c stepca.freyjapki.com -s /var/www/html/FreyjaPKISigningCA.pem -o /var/www/html/FreyjaPKISigningCA.crl > /dev/null
```
This will run the updater script every minute to update the published CRL.  That's definitely more often than necessary but it's not like CRLs are big files or verifying them is computationally intensive so no harm done.  

### Conclusion

You're done.  You now have a fully featured online issuing CA, AIA, and CRL endpoints.  From here you can apply what you've learned to go around your network and enroll all your various services with the CA, setup automatic renewal, and secure all your network services.  Future articles will focus on specific applications, provisioners, and use cases, but you can already do a lot with what you have.  




