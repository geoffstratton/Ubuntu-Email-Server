# Build an Ubuntu Email Server

How to build an Ubuntu Email Server with Postfix, Dovecot, and MySQL.

### Update November 2020: Update November 2020: If you're on Ubuntu 20.04, these instructions still mostly work as written. There is one update of note, though, related to Dovecot and SSL. This is indicated below for Dovecot's 10-ssl.conf file.

### Update November 2016: If you're on Ubuntu 16.04, these instructions will mostly work as written. However, there are a [few minor changes](Ubuntu1604.md) that you might want to review before you get started.

This article details how to set up your own Ubuntu mail server using Postfix and Dovecot with virtual users and domains. I started this project on Ubuntu 12.04 LTS but the procedures apply to later versions as well. After this we'll add a webmail client (I'd suggest [Roundcube](https://github.com/geoffstratton/RoundCube-on-Ubuntu-16-with-Nginx-and-PHP-FPM), although you can use SquirrelMail) and an anti-spam solution ([SpamAssassin](https://github.com/geoffstratton/SpamAssassin-on-Ubuntu)). These articles are also available in my Github. 

This assumes you have MySQL running and understand how to configure any local firewalls (probably with ufw if you're running a firewall on your Ubuntu server). It also assumes you have your domains forwarding to your server and that your /etc/hostname file is set to mydomain<span></span>.com.

## The Ubuntu Mail Server Install

### Step 0: Install the necessary packages.
	
```shell
$ apt-get install postfix postfix-mysql dovecot-core dovecot-imapd dovecot-lmtpd dovecot-mysql
```

Include dovecot-pop3d if you want POP email; I won't bother here.

## The Ubuntu Mail Server Database

### Step 1: Create a database and tables to store your mail server configuration information.
	
```shell
$ mysql -uroot -p
```

```sql
mysql> CREATE DATABASE mailserver DEFAULT CHARACTER SET = 'utf8'; GRANT SELECT ON mailserver.* TO '(mail_username)'@'127.0.0.1' IDENTIFIED BY '(mail_password)'; FLUSH PRIVILEGES;`
mysql> USE mailserver;
```

Now, create tables for your virtual domains and users, and virtual aliases if you want forwarding. (For some further discussion of this, see my comments below.)

```sql
mysql> CREATE TABLE `virtual_domains` (
      `id` int(11) NOT NULL auto_increment,
      `name` varchar(50) NOT NULL,
      PRIMARY KEY (`id`)
    );
mysql> CREATE TABLE `virtual_users` (
      `id` int(11) NOT NULL auto_increment,
      `domain_id` int(11) NOT NULL,
      `password` varchar(106) NOT NULL,
      `email` varchar(100) NOT NULL,
      PRIMARY KEY (`id`),
      UNIQUE KEY `email` (`email`),
      FOREIGN KEY (domain_id) REFERENCES virtual_domains(id) ON DELETE CASCADE
    );
mysql> CREATE TABLE `virtual_aliases` (
      `id` int(11) NOT NULL auto_increment,
      `domain_id` int(11) NOT NULL,
      `source` varchar(100) NOT NULL,
      `destination` varchar(100) NOT NULL,
      PRIMARY KEY (`id`),
      FOREIGN KEY (domain_id) REFERENCES virtual_domains(id) ON DELETE CASCADE
    );
```

The virtual_users password field is set at 106 characters because of the formula used in the next step for creating user passwords: 86-character encrypted password + 4 separator characters + 16-character salt. The SQL for this is ENCRYPT('firstpassword', CONCAT('$6$', SUBSTRING(SHA(RAND()), -16))). This tells MySQL to:

1. Generate a random floating point value between 0 and 1.0
2. Calculate an SHA1 checksum for the random value, expressed as a string of 40 hexadecimal digits, e.g., 'a9993e364706816aba3e25717850c26c9cd0d89d'
3. Select a substring of (2), starting with the character in the 25th position and running to position 40 (16 total characters)
4. Concatenate '$6$' and your substring from (3), so you end up with a value like '$6$7850c26c9cd0d89d'
5. Encrypt 'firstpassword' into a binary string using your concatenated randomized value from (4) as the salt. Per the MySQL documentation, ENCRYPT() relies on the crypt() Unix system call, so results will vary by platform, but on current Linux systems this gives you a string like '$6$7850c26c9cd0d89d$encrypted-password'. On Linux systems, the '$6$' appended to the salt also tells crypt() to use SHA-512 encryption, giving you an 86-character encrypted string (see the crypt man page). So now you have '$6$' + 16-character salt + '$' + 86-character encrypted password = 106 characters.
6. Return the final result to the INSERT statement.

### Step 2: Add some email addresses and domains.

```sql
mysql> INSERT INTO `mailserver`.`virtual_domains`
      (`id` ,`name`)
      VALUES
      ('1', 'mydomain.com'),
      ('2', 'subdomain.mydomain.com');
mysql> INSERT INTO `mailserver`.`virtual_users`
      (`id`, `domain_id`, `password` , `email`)
      VALUES
      ('1', '1', ENCRYPT('firstpassword', CONCAT('$6$', SUBSTRING(SHA(RAND()), -16))), 'email1@mydomain.com'),
      ('2', '1', ENCRYPT('secondpassword', CONCAT('$6$', SUBSTRING(SHA(RAND()), -16))), 'email2@mydomain.com');
mysql> INSERT INTO `mailserver`.`virtual_aliases`
      (`id`, `domain_id`, `source`, `destination`)
      VALUES
      ('1', '1', 'alias@domain1.com', 'email1@mydomain.com');
```

## The Postfix Mail-Transfer Agent

### Step 3: Set up Postfix to handle incoming email.

Edit /etc/postfix/main.cf (my comments are included with the ## symbol):

```ini
# See /usr/share/postfix/main.cf.dist for a commented, more complete version
# Debian specific:  Specifying a file name will cause the first
# line of that file to be used as the name.  The Debian default
# is /etc/mailname.
#myorigin = /etc/mailname

smtpd_banner = $myhostname ESMTP $mail_name (Ubuntu)
biff = no

# appending .domain is the MUA's job.
append_dot_mydomain = no

# Uncomment the next line to generate "delayed mail" warnings
#delay_warning_time = 4h

readme_directory = no

# TLS parameters

## We're commenting out the default system self-signed cert and using the free SSL cert included with Dovecot. 
## We're also requiring TLS encryption for users to connect. If you bought an SSL certificate, substitute it for the 
## Dovecot entries in smtpd_tls_cert_file and smtpd_tls_key_file.

#smtpd_tls_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem
#smtpd_tls_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
#smtpd_use_tls = yes
#smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
#smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache

smtpd_tls_cert_file=/etc/ssl/certs/dovecot.pem
smtpd_tls_key_file=/etc/ssl/private/dovecot.key
smtpd_use_tls = yes
smtpd_tls_auth_only = yes

## Allow authenticated users to send email, and use Dovecot to authenticate them. 
## Tells Postfix to use Dovecot for authentication. 
smtpd_sasl_type = dovecot
## Path to the Postfix auth socket, relative to /var/spool/postfix/. 
smtpd_sasl_path = private/auth
## Tells Postfix to let people send email if they've authenticated to the server. 
## Otherwise they can only send if they're logged in (SSH).
smtpd_sasl_auth_enable = yes
## Tells Postfix who can send email: SASL-authenticated users connecting from a 
## network specified in 'mynetworks' below. Also prevents anybody from sending mail to users who aren't on the server.
smtpd_recipient_restrictions =
        permit_sasl_authenticated,
        permit_mynetworks,
        reject_unauth_destination

# See /usr/share/doc/postfix/TLS_README.gz in the postfix-doc package for
# information on enabling SSL in the smtp client.

myhostname = hostname.mydomain.com
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases
myorigin = /etc/mailname

## Comment out the mydestination value and put in 'localhost'. 
## This allows you to use the virtual domains in MySQL for mail delivery. 
## If there's overlap between the virtual domains and mydestination you'll see warnings in the mail log.
#mydestination = mydomain.com, hostname.mydomain.com, localhost.mydomain.com, localhost
mydestination = localhost

relayhost =

## Mynetworks is important. If it isn't set to localhost, anybody may be able to use your server to send spam (i.e., an open relay).
## In other words, there are two situations where a client doesn't have to authenticate to send email through your server: 
## 1) They send to a recipient who has an account on the server
## 2) They send from a client whose IP is listed in mynetworks in /etc/postfix/main.cf. 
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128

mailbox_size_limit = 0
recipient_delimiter = +
inet_interfaces = all

## Tells Postfix to use Dovecot's LMTP instead of its own LDA to save emails to the local mailboxes.
virtual_transport = lmtp:unix:private/dovecot-lmtp

## Tells Postfix you're using MySQL to store virtual domains, and gives the paths to the database connections. 
virtual_mailbox_domains = mysql:/etc/postfix/mysql-virtual-mailbox-domains.cf
virtual_mailbox_maps = mysql:/etc/postfix/mysql-virtual-mailbox-maps.cf
virtual_alias_maps = mysql:/etc/postfix/mysql-virtual-alias-maps.cf
```

Now, create the three files we specified in the last part of main<span></span>.cf.

/etc/postfix/mysql-virtual-mailbox-domains.cf:

```ini
user = mail_username
password = mail_password
hosts = 127.0.0.1
dbname = mailserver
query = SELECT 1 FROM virtual_domains WHERE name='%s'
```

The 'mail_username' and 'mail_password' here should be the actual credentials you specified for the mailserver database in step 1. Postfix has to log in to the mailserver database to run queries against it.

/etc/postfix/mysql-virtual-mailbox-maps.cf:

```ini	
user = mail_username
password = mail_password
hosts = 127.0.0.1
dbname = mailserver
query = SELECT 1 FROM virtual_users WHERE email='%s'
```

/etc/postfix/mysql-virtual-alias-maps.cf:

```ini	
user = mail_username
password = mail_password
hosts = 127.0.0.1
dbname = mailserver
query = SELECT destination FROM virtual_aliases WHERE source='%s'
```

Restart postfix and test your connections:
	
```shell
$ service postfix restart
$ postmap -q mydomain.com mysql:/etc/postfix/mysql-virtual-mailbox-domains.cf
$ postmap -q myemail@mydomain.com mysql:/etc/postfix/mysql-virtual-mailbox-maps.cf
$ postmap -q myalias@mydomain.com mysql:/etc/postfix/mysql-virtual-alias-maps.cf
```

If the first two postmap commands return anything but 1, and the third one returns anything but the destination email address, there's a problem -- most likely bad credentials or queries in the three mysql-virtual files.

Note: You should ensure that port TCP/25 is open on your firewall so your server can receive email from the wider internet. You might also want to open the submission port TCP/587 for users to connect securely with email clients. If so, modify your /etc/postfix/master.cf file by uncommenting these lines:
	
```ini
submission inet n       -       -       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_client_restrictions=permit_sasl_authenticated,reject
```

## The Dovecot Mail-Delivery Agent

### Step 4: Set up Dovecot.

Edit /etc/dovecot/dovecot.conf:
	
```ini
## Add pop3 here if you want that too.
# Enable installed protocols
!include_try /usr/share/dovecot/protocols.d/*.protocol
protocols = imap lmtp
```

Also verify that dovecot.conf is including all the other configuration files:
	
```ini
!include conf.d/*.conf`
```

Edit /etc/dovecot/conf.d/10-mail.conf:
	
```ini
## Tells Dovecot where to find mail in maildir format, in this case in /var/mail/vhosts/yourdomain.com/email@yourdomain.com.
mail_location = maildir:/var/mail/vhosts/%d/%n

## Tells Dovcot to write to the /var/mail folder.
mail_privileged_group = mail
```

Verify permissions, and create the virtual mail directory and user/group:
	
```shell
$ ls -ld /var/mail
drwxrwsr-x 2 root mail 4096 Apr  12 22:14 /var/mail

$ mkdir -p /var/mail/vhosts/example.com
$ groupadd -g 5000 vmail
$ useradd -g vmail -u 5000 vmail -d /var/mail
$ chown -R vmail:vmail /var/mail
```

Edit /etc/dovecot/conf.d/10-auth.conf:
	
```ini
## Uncomment this line
disable_plaintext_auth = yes
## Modify this line
auth_mechanisms = plain login
## Comment out this line:
#!include auth-system.conf.ext
## Uncomment the MySQL auth line
#!include auth-system.conf.ext
!include auth-sql.conf.ext
#!include auth-ldap.conf.ext
#!include auth-passwdfile.conf.ext
#!include auth-checkpassword.conf.ext
#!include auth-vpopmail.conf.ext
#!include auth-static.conf.ext
```

Create /etc/dovecot/conf.d/auth-sql.conf.ext to hold your authentication info:

```ini	
passdb {
  driver = sql
  args = /etc/dovecot/dovecot-sql.conf.ext
}
userdb {
  driver = static
  args = uid=vmail gid=vmail home=/var/mail/vhosts/%d/%n
}
```

Edit /etc/dovecot/dovecot-sql.conf.ext:
	
```ini
## Uncomment and specify the driver
driver = mysql
## Uncomment and set connection information
connect = host=127.0.0.1 dbname=mailserver user=mail_username password=mail_password
## Uncomment and set the encryption format for passwords
default_pass_scheme = SHA512-CRYPT
## Uncomment and set password query
password_query = SELECT email as user, password FROM virtual_users WHERE email='%u';
```

Set permissions on the /etc/dovecot directory so the vmail user can use it:
	
```shell
$ chown -R vmail:dovecot /etc/dovecot
$ chmod -R o-rwx /etc/dovecot
```

Edit /etc/dovecot/conf.d/10-master.conf:
	
```ini
## Disable unencrypted IMAP by setting the port to 0
service imap-login {
  inet_listener imap {
    port = 0
  }
...
}
## (Leave imaps-login alone)

## Create LMTP socket for Dovecot where we told Postfix to find it
service lmtp {
unix_listener /var/spool/postfix/private/dovecot-lmtp {
  mode = 0600
  user = postfix
  group = postfix
  }
  # Create inet listener only if you can't use the above UNIX socket
  #inet_listener lmtp {
    # Avoid making LMTP visible for the entire internet
    #address =
    #port =
  #}
}

## Create authorization socket where we told Postfix to find it 
service auth {
  # auth_socket_path points to this userdb socket by default. It's typically
  # used by dovecot-lda, doveadm, possibly imap process, etc. Its default
  # permissions make it readable only by root, but you may need to relax these
  # permissions. Users that have access to this socket are able to get a list
  # of all usernames and get results of everyone's userdb lookups.
  unix_listener /var/spool/postfix/private/auth {
    mode = 0666
    user = postfix
    group = postfix
  }

  unix_listener auth-userdb {
    mode = 0600
    user = vmail
    #group =
  }

  # Postfix smtp-auth
  #unix_listener /var/spool/postfix/private/auth {
  #  mode = 0666
  #}

  # Auth process is run as this user.
  user = dovecot
}

## Set the auth-worker user to vmail
service auth-worker {
  # Auth worker process is run as root by default, so that it can access
  # /etc/shadow. If this isn't necessary, the user should be changed to
  # $default_internal_user.
  user = vmail
}
```

Edit /etc/dovecot/conf.d/10-ssl.conf:
	
```ini
## If you have your own SSL cert and key, specify them here. We're using the free ones that come with Dovecot.
ssl_cert = </etc/ssl/certs/dovecot.pem
ssl_key = </etc/ssl/private/dovecot.key
## Force clients to use SSL
ssl = required
```

**Note for Ubuntu 20.04:** With this LTS release, the included OpenSSL version requires at least 2048-bit Diffie-Hellman parameters. Create a dh.pem file and then reference it from Dovecot's 10-ssl.conf file:

```shell
$ openssl dhparam -out /etc/dovecot/private/dh.pem 2048
```

```ini
ssl_cert = </etc/ssl/certs/dovecot.pem
ssl_key = </etc/ssl/private/dovecot.key
ssl_dh = </etc/dovecot/private/dh.pem
```

Restart Dovecot:

```shell
$ service dovecot restart
```

Note: Make sure that port TCP/993 is open on your server's firewall for IMAP. If you added POP, open port TCP/995.

## Ubuntu Mail Server Conclusion and Testing

### Step 5: Test everything.

In your preferred email program, set up a test account:

```plaintext
Username: myemail@mydomain.com, as you specified in step 2
Password: Password you added to the MySQL table in step 2 for this email address
IMAP server: yourdomain.com
SMTP server: yourdomain.com
Ports: 993 for secure IMAP, 995 for secure POP, 25 or 587 for SMTP
```

Try sending mail and see what happens. If it didn't work, check the log in /var/log/mail.log. You can also turn on verbose logging for Dovecot by adding a few lines to /etc/dovecot.conf:

```ini	
## Verbose logging
auth_debug_passwords=yes
mail_debug=yes
auth_verbose=yes
verbose_ssl=yes
auth_verbose_passwords=plain
```

Once things are working, you should probably comment these out again so they don't clog up your mail log.

Finally, to add users/domains/forwards to your mail server, just add them to the appropriate tables in MySQL like in step 2. That's all you need to do since authentication and incoming mail query the database to determine validity. For new accounts, the appropriate folders are created in /var/mail/ when each account first receives a message.

In our next installment, we'll set up an anti-spam system.
