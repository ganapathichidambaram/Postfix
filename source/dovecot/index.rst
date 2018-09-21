=====================================================
Dovecot - IMAP & POP3 Handler
=====================================================

Dovecot is an open-source IMAP and POP3 server for Linux/UNIX-like systems, written primarily with security in mind.
Now that Postfix is installed and configured, we need to install postfix to manage the pop and imap protocols, 
which allow us to recover our emails.

Installation of Dovecot
=============================

Dovecot packages are presents in the Ubuntu 18.04 default repositories. We will install it with the mysql support.
We will install sieve which is useful because it will automatically put the mails into the corresponding folders.
It means that, for each domain, it will create a corresponding folder containing the corresponding folder of a 
virtual user to store its email files.

	.. code:: bash
	
	 apt install dovecot-imapd dovecot-mysql dovecot-managesieved

Configuration of Dovecot
====================================

Now go to the folder containing the configuration files.

	.. code:: bash

	 # cd /etc/dovecot/conf.d

* **10-auth.conf** file to modify the connection mechanisms by adding or uncommenting the lines. 
	Dovecot uses the system users by default but we use Mysql users

	.. code:: bash

	 auth_mechanisms = plain login
	 #!include auth-system.conf.ext
	 !include auth-sql.conf.ext

* **auth-sql.conf.ext** file for the sql configuration

	.. code:: bash

	 passdb {
		driver = sql
		args = /etc/dovecot/dovecot-sql.conf.ext
	 }
	 userdb {
		driver = static
		args = uid=vmail gid=vmail home=/var/mail/vmail/%d/%n
	 }

* **/etc/dovecot/dovecot-sql.conf.ext** to tell dovecot how to connect to the SQL database

	.. code:: bash

	 driver = mysql
	 connect = host=127.0.0.1 dbname=postfix user=postfix password=postfix-db-password
	 password_query = SELECT username,domain,password FROM mailbox WHERE username='%u';
	 default_pass_scheme = MD5-CRYPT

* **10-mail.conf** file to configure the mail location directory

	.. code:: bash

	 mail_location = maildir:/var/mail/vmail/%d/%n/Maildir
	 mail_privileged_group = mail

* **10-master.conf** file for the connection to the socket.

	.. code:: bash

		service auth {
			unix_listener auth-userdb {
				mode = 0600
				user = vmail
			}
			unix_listener /var/spool/postfix/private/auth {
				mode = 0660
				user = postfix
				group = postfix
			}
			user = dovecot
		}

* **15-lda.conf** file to indicate sieve in order to automatically organize mail into the corresponding folder

	.. code:: bash

		protocol lda {
		# Space separated list of plugins to load (default is global mail_plugins).
		mail_plugins = $mail_plugins sieve
		}

We should give permission if we want that the vmail user can launch dovecot

	.. code:: bash

		# chgrp vmail /etc/dovecot/dovecot.conf

Now you can restart the dovecot service

	.. code:: bash

		# systemctl restart dovecot


Integrate dovecot to postfix
===================================

Now that we have configured dovecot, we should indicate postfix to work with dovecot. 
Edit the master postfix configuration file(**/etc/postfix/master.cf**) and add the lines below at the end of the file

	.. code:: bash

		dovecot   unix  -       n       n       -       -       pipe
		flags=DRhu user=vmail:vmail argv=/usr/lib/dovecot/deliver -f ${sender} -d ${user}@${nexthop}

Now edit the main postfix configuration file (**/etc/postfix/main.cf**)

	.. code:: bash

		# Allow authenticated users to send email, and use Dovecot to authenticate them. Tells Postfix to use Dovecot for authentication
		virtual_transport = dovecot
		dovecot_destination_recipient_limit = 1
		smtpd_sasl_type = dovecot
		smtp_sasl_type = dovecot

Then restart postfix

	.. code:: bash

		# systemctl restart postfix

