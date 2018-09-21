=====================================================
Delivery - Better Delivery of Email
=====================================================

To Avoid Spam need to ensure some configuration on mail server for smooth delivery process.

* DKIM Signing
* DMARC
* SPF

DKIM Signing
=======================

Install opendkim package for dkim signing.

	.. code:: bash

		apt install opendkim opendkim-tools postfix-policyd-spf-python postfix-pcre

Create folder for dkim key and allocate necessary permission to access.

	.. code:: bash

		mkdir -p /etc/opendkim/keys/mytuto.com
		chown -R opendkim:opendkim /etc/opendkim
		chmod go-rw /etc/opendkim/keys

Generate opendkim for signing email for better delivery.

	.. code:: bash

		opendkim-genkey -b 2048 -D /etc/opendkim/keys/mytuto.com -h rsa-sha256 -r -s dkim -d mytuto.com -v

And configure **/etc/opendkim.conf** to use the generated key for signing.

	.. code:: bash
		
		# selector '2007' (e.g. 2007._domainkey.example.com)
		Domain    mytuto.com
		KeyFile    /etc/opendkim/keys/mytuto.com/dkim.private
		Selector   dkim
		SOCKET    inet:8891@127.0.0.1
		Canonicalization        relaxed/simple
		Mode                    sv
		SubDomains              no
		AutoRestart         yes
		AutoRestartRate     10/1M
		Background          yes
		DNSTimeout          5
		SignatureAlgorithm  rsa-sha256


And add DNS TXT entry with content of dkim.txt which is generated from opendkim-genkey

	.. code:: bash

		#cat /etc/opendkim/keys/mytuto.com/dkim.txt
		dkim._domainkey IN      TXT     ( "v=DKIM1; h=rsa-sha256; k=rsa; s=email; "
          "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAtNchMEHZ4U+7sYE69ZapO+hCPgbqx87muMKwwcM/voqrgLhCv/OOnHhcawoCb6buCwVrb+GgU0hHS+UqcTsFS3BTeFuPis5fXdoXzqUgOj1q6k/wqlscYRQJq+M+j+cufR2i7e8O1DQ/KO8tCjkZenOhPYZ8LA6HaagMTQgyGBP8HqgAMsY2PEGchdfB2SezGrZ1ZogvoUeGaH"
          "2A9AmUGJQzU3SPAbBs53v6SG5ePrhTRf6spC47THccCJfE7za5smMjVzkO9jD85XyQvAR6q/jVtaM9HbLT6+ipcydmaMT/9+SOG5JvvDHPrnDEAPKf3oTKSEmCa1VRKJNWCi8EpQIDAQAB" )

Now integrate opendkim with postfix to use opendkim key by adding below line at end of file(**/etc/postfix/main.cf**)

	.. code:: bash

		# DKIM
		# --------------------------------------
		milter_default_action = accept
		milter_protocol = 2
		smtpd_milters = inet:127.0.0.1:8891
		non_smtpd_milters = inet:127.0.0.1:8891


SPF Record
===================

The value in an SPF DNS record will look something like the following examples.

Example 1 Allow mail from all hosts listed in the MX records for the domain: ::

	v=spf1 mx -all

Example 2 Allow mail from a specific host: ::

	v=spf1 a:mail.mytuto.com -all

* The **v=spf1** tag is required and has to be the first tag.

* The last tag, **-all**, indicates that mail from your domain should only come from servers identified in the SPF string.Anything coming from any other source is forging your domain. An alternative is **~all**, indicating the same thing but also indicating that mail servers should accept the message and flag it as forged instead of rejecting it outright. **-all** makes it harder for spammers to forge your domain successfully; it is the recommended setting. **~all** reduces the chances of email getting lost because an incorrect mail server was used to send mail. **~all** can be used if you don't want to take chances.

The tags between identify eligible servers from which email to your domain can originate.

* **mx** is a shorthand for all the hosts listed in MX records for your domain. If you've got a solitary mail server, mx is	probably the best option. If you've got a backup mail server (a second MX record), using mx won't cause any problems.Your backup mail server will be identified as an authorized source for email although it will probably never send any.

* The **a** tag lets you identify a specific host by name or IP address, letting you specify which hosts are authorized. You'd use a if you wanted to prevent the backup mail server from sending outgoing mail or if you wanted to identify hosts other than your own mail server that could send mail from your domain (e.g., putting your ISP's outgoing mail servers in the list so they'd be recognized when you had to send mail through them).

For now, we're going to stick with the mx version. It's simpler and correct for most basic configurations, including those
that handle multiple domains. To add the record, go to your DNS management interface and add a record of type TXT for your 
domain itself (i.e., a blank hostname) containing this string:

	::

		mytuto.com TXT v=spf1 mx -all

DMARC
=================

Add below entries on your DNS Server for DMARC record for some mail server's better email delivery.

	::

		_dmarc TXT  "v=DMARC1; p=none; adkim=r; aspf=r;"

The **none** indicates that the remove server should not drop the mails, even if they are not coming from the servers 
listed in the SPF record. Once you're sure everything is fine, change the **none** to **reject**.

