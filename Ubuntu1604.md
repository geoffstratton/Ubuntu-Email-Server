# Build an Email Server on Ubuntu 16.04 

My [previous instructions](README.md) for setting up a Postfix/Dovecot/MySQL mail server on Ubuntu 12.04 work mostly as-is for Ubuntu 16.04, but there are a couple of gotchas. These workarounds are necessary due to some different locations of utilities on the filesystem and some changes to Ubuntu's internals. We'll go through these differences one by one.

## Mail Logging

With Ubuntu 14.04, the logging subsystem changed from syslogd to rsyslogd, which behaves a little differently. In particular you may notice that your mail logs are now being written to /var/log/syslog as opposed to /var/log/mail.log as in older versions of Ubuntu.

To set up custom mail logging, first edit /etc/rsyslog.d/50-default.conf:

```ini	
# Uncomment these lines, set their paths to the log files you want, and place them above the catchall *.*; directive:
mail.info                 /var/log/mail.log
mail.warn                 /var/log/mail.log
mail.err                  /var/log/mail-err.log
*.*;
```

Note that the "*.*" directive will continue logging your mail entries to syslog as well as /var/log/mail.log. If you don't want this, comment it out -- but if there are any other logs you want to keep, like auth.log or kern.log, you should ensure those are specified.

Next you have to create the files you just referenced and give them appropriate ownership so rsyslog can write to them:

```shell
root@ubuntu# touch /var/log/mail.log
root@ubuntu# touch /var/log/mail-err.log
root@ubuntu# chown syslog:adm /var/log/mail*.log
```

Finally, restart rsyslog:

```shell	
root@ubuntu# /etc/init.d/rsyslog restart
```

Now when you log in via your email client, or send an email message, you'll notice the activity being appended to /var/log/mail.log.

## The Dovecot Certificate

On Ubuntu 12.04, installing Dovecot using the Aptitude package manager gave you a free key and certificate that you could reference from Postfix. It seems that this freebie certificate is no longer included, so you'll need to provide a key-cert pair yourself. Assuming you don't want to buy a certificate, you can simply use Let's Encrypt, or create a self-signed certificate:

### Let's Encrypt

First, obviously you should have Certbot [set up on your server](https://www.geoffstratton.com/free-ssl-certificates-lets-encrypt-ubuntu-16).
 
Then, create some certs for your mail domains (this assumes you're running Nginx as your web server):

```
root@ubuntu:/# certbot certonly --nginx -d mail.domain1.com -d mail.domain2.com
```

Finally, you want to ensure that Postfix and Dovecot get restarted when the certs are auto-renewed. Edit /etc/letsencrypt/renewal-hooks/mail.domain1.com, comment out the Nginx line, and add your mail services:

```
# renew_hook = systemctl restart nginx
renew_hook = systemctl restart postfix dovecot
```

If you want to test your renewal hooks, you can do `certbot renew --dry-run`.

### Self-Signed

```shell	
root@ubuntu# openssl req -x509 -newkey rsa:4096 -keyout dovecot.key -out dovecot.pem -days 365 -nodes
```

This creates an OpenSSL key-cert pair that is valid for 365 days. The "-nodes" switch bypasses the password protection on the key.

Once this is complete, place your cert and key files in appropriate locations (/etc/ssl/certs/ and /etc/ssl/private/), and then make sure that these paths are reflected in your /etc/postfix/main.cf file:

```ini	
smtpd_tls_cert_file=/etc/ssl/certs/dovecot.pem
smtpd_tls_key_file=/etc/ssl/private/dovecot.key
smtpd_use_tls = yes
```

And that's it!