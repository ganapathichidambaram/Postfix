=====================================================
Postfix - Mail Transfer Agent
=====================================================

Postfix is a free and open-source mail transfer agent that routes and delivers electronic mail.

Install postfix
==============================

Now we can install the postfix packages.

	.. code:: bash

	 apt install postfix postfix-mysql sasl2-bin

You will have to answer two question about the type of mail and the name of your mail server. 
Make sure to replace the hostname and domain values with yours

* the type of mail configuration: Internet Site
* the system mail name: hostname.domain.com

Make sure that sasl run at the startup by editing its configuration file(/etc/default/saslauthd)

	.. code:: bash

	 # Should saslauthd run automatically on startup? (default: no)
	 START=yes

Now restart the service

	.. code:: bash

	 # systemctl restart saslauthd


As we are configuring a mail server with virtual users, we need an owner of all mailboxes so will create a system user 
which will be used by all virtual users to access email on the server. First, create the group owner and the folder 
which will store the mailboxes.

	.. code:: bash
		
	 # groupadd -g 5000 vmail && mkdir -p /var/mail/vmail

Now create the owner

	.. code:: bash
		
	 # useradd -u 5000 vmail -g vmail -s /usr/sbin/nologin -d /var/mail/vmail


Make sure to give the permission of the mail directory to the owner so that it can store the mails into the appropriate directories.

	.. code:: bash
		
	 # chown -R vmail:vmail /var/mail/vmail

If you don't do this, dovecot will not be able to create the required folders to store the emails.

Create the configuration files for the database
==================================================

Now create a folder which will contain some database files

	.. code:: bash

	 # mkdir -p /etc/postfix/sql

Postfix need 03 database files which will allow it to access the database that we created earlier:

Domains to contain the list of domain names hosted on the server. it will allow postfix to determine 
if our server is in charge of a domain (mytuto.com) when it receives an email (user@mytuto.com) on it. 
If it's the case, it will mean that the domain is in our database.

	.. code:: bash

	 # vim /etc/postfix/sql/mysql_virtual_domains_maps.cf
	 user = postfix
	 password = postfix-db-password
	 hosts = 127.0.0.1
	 dbname = postfix
	 query = SELECT domain FROM domain WHERE domain='%s' AND active = '1'

We will enable the configuration and add it automatically to the /etc/postfix/main.cf file and reload the 
postfix configuration to avoid having to do it manually. So the file will be updated everytime you use this 
command with new values.

	.. code:: bash

	 # postconf -e virtual_mailbox_domains=mysql:/etc/postfix/sql/mysql_virtual_domains_maps.cf

Now we can check the configuration. We will run a command that will execute the query contained in the file 
in order to search for a domain in our database. An element (the searched domain) must be returned or nothing
if the domain is not present.

	.. code:: bash

	 # postmap -q mytuto.com mysql:/etc/postfix/sql/mysql_virtual_domains_maps.cf
	 mytuto.com

As you can see, postfix is able to retrieve the domains stored in our database

Mailbox to store all the virtual email addresses. It will be used to verify also if the mailboxes exist

	.. code:: bash
	
	 # vim /etc/postfix/sql/mysql_virtual_mailbox_maps.cf
	 user = postfix
	 password = postfix-db-password
	 hosts = 127.0.0.1
	 dbname = postfix
	 query = SELECT maildir FROM mailbox WHERE username='%s' AND active = '1'

Now let's update the configuration file

	.. code:: bash

	 # postconf -e virtual_mailbox_maps=mysql:/etc/postfix/sql/mysql_virtual_mailbox_maps.cf

Run the command to test the query on the database

	.. code:: bash

	 # postmap -q alain@mytuto.com mysql:/etc/postfix/sql/mysql_virtual_mailbox_maps.cf
	 mytuto.com/alain/
	 
Alias to contain the different email aliases.

	.. code:: bash

	 # vim /etc/postfix/sql/mysql_virtual_alias_maps.cf
	 user = postfix
	 password = postfix-db-password
	 hosts = 127.0.0.1
	 dbname = postfix
	 query = SELECT goto FROM alias WHERE address='%s' AND active = '1'

Now add the configuration

	.. code:: bash

	 # postconf -e virtual_alias_maps=mysql:/etc/postfix/sql/mysql_virtual_alias_maps.cf

Now run the command to test the query. It is the destination user (alain@mytuto.com) that should be displayed 
and not the abuse address. It shows that postfix can do the matching.

	.. code:: bash

	 # postmap -q abuse@mytuto.com mysql:/etc/postfix/sql/mysql_virtual_alias_maps.cf
	 alain@mytuto.com

Make sure that those files are not readable by the normal users because the passwords are stored in clear.
In order for postfix to read those file, we can change the group owner to postfix

	.. code:: bash

	 # chgrp postfix /etc/postfix/sql/mysql_*.cf


Configure postfix
======================

Now we will manually edit the postfix main configuration file. So, make a copy before editing.

	.. code:: bash

	 # cp /etc/postfix/main.cf /etc/postfix/main.cf.bak

Now we will activate SASL to force authentication for sending emails and hand off authentication to Dovecot. Be sure to add lines below

	.. code:: bash

	 # vim /etc/postfix/main.cf
	 # --------------------------------------
	 myhostname = mail.hoppr.in
	 mydomain = hoppr.in
	 alias_maps = hash:/etc/aliases
	 alias_database = hash:/etc/aliases
	 myorigin = /etc/mailname
	 virtual_alias_domains = mail.hoppr.in
	 mydestination = $myhostname, mail.hoppr.in, ip-172-30-1-40, localhost.localdomain, localhost
	 mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128
	 # --------------------------------------
	 mailbox_size_limit = 0
	 recipient_delimiter = +
	 inet_interfaces = all
	 inet_protocols = all
	 # --------------------------------------
	 virtual_mailbox_domains = mysql:/etc/postfix/sql/mysql_virtual_domains_maps.cf
	 virtual_mailbox_maps = mysql:/etc/postfix/sql/mysql_virtual_mailbox_maps.cf
	 virtual_alias_maps = mysql:/etc/postfix/sql/mysql_virtual_alias_maps.cf
	 # --------------------------------------
	 ## Path to the Postfix auth socket
	 smtpd_sasl_path = private/auth
	 smtp_sasl_path = private/auth
	 # --------------------------------------
	 ## Tells Postfix to let people send email if they've authenticated to the server.
	 ## Otherwise they can only send if they're logged in (SSH)
	 smtpd_sasl_auth_enable = yes
	 smtpd_sasl_security_options = noanonymous
	 smtp_sasl_security_options = noanonymous
	 smtpd_sasl_local_domain = $myhostname
	 # --------------------------------------
	 # TLS parameters
	 smtpd_use_tls=yes
	 smtp_use_tls = yes
	 smtpd_tls_security_level = may
	 smtpd_tls_auth_only = yes
	 smtp_tls_security_level = may
	 smtpd_tls_cert_file=/etc/letsencrypt/live/mail.hoppr.in/fullchain.pem
	 smtpd_tls_key_file=/etc/letsencrypt/live/mail.hoppr.in/privkey.pem
	 smtp_tls_cert_file=/etc/letsencrypt/live/mail.hoppr.in/fullchain.pem
	 smtp_tls_key_file=/etc/letsencrypt/live/mail.hoppr.in/privkey.pem
	 smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
	 smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache
	 smtpd_sender_restrictions = permit_sasl_authenticated
	 smtpd_recipient_restrictions = check_recipient_access hash:/etc/postfix/custom_replies


	 # Allow authenticated users to send email, and use Dovecot to authenticate them. Tells Postfix to use Dovecot for authentication
	 virtual_transport = dovecot
	 dovecot_destination_recipient_limit = 1
	 smtpd_sasl_type = dovecot
	 smtp_sasl_type = dovecot

	 # DKIM
	 # --------------------------------------
	 milter_default_action = accept
	 milter_protocol = 2
	 smtpd_milters = inet:127.0.0.1:8891
	 non_smtpd_milters = inet:127.0.0.1:8891

Now let's edit the /etc/postfix/master.cf configuration file. It's the process configuration file. 
We will enable secure SMTP ports by adding or uncomment the lines below and make a copy before.

	.. code:: bash

	 # cp /etc/postfix/master.cf /etc/postfix/master.cf.bak

	.. code:: bash

	 # vim /etc/postfix/master.cf
	 submission inet n       -       y       -       -       smtpd
		-o syslog_name=postfix/submission
		-o smtpd_tls_security_level=encrypt
		-o smtpd_tls_ask_ccert=yes
		-o smtpd_sasl_auth_enable=yes
		-o smtpd_reject_unlisted_recipient=no
		-o smtpd_client_restrictions=permit_sasl_authenticated,reject
		-o smtpd_recipient_restrictions=permit_sasl_authenticated,reject
		-o milter_macro_daemon_name=ORIGINATING
	 smtps     inet  n       -       y       -       -       smtpd
		-o syslog_name=postfix/smtps
		-o smtpd_tls_wrappermode=yes
		-o smtpd_sasl_auth_enable=yes
		-o smtpd_client_restrictions=permit_sasl_authenticated,reject
		-o milter_macro_daemon_name=ORIGINATING
	 dovecot   unix  -       n       n       -       -       pipe
		flags=DRhu user=vmail:vmail argv=/usr/lib/dovecot/deliver -f ${sender} -d ${user}@${nexthop}

Now you can run the postconf -n command to check some errors.

	.. code:: bash

	 # postconf -n
	 alias_database = hash:/etc/aliases
	 alias_maps = hash:/etc/aliases
	 ...
	 ...

If you have no warning messages, it means that your files do not contain errors. Now you can restart the postfix service.

	.. code:: bash

	 # systemctl restart postfix
	 # systemctl status postfix
		* postfix.service - Postfix Mail Transport Agent
		Loaded: loaded (/lib/systemd/system/postfix.service; enabled; vendor preset: enabled)
		Active: active (exited) since Wed 2018-09-22 10:16:02 UTC; 27s ago
		Process: 12225 ExecStart=/bin/true (code=exited, status=0/SUCCESS)
		Main PID: 12225 (code=exited, status=0/SUCCESS)

