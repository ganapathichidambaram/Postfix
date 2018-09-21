=====================================================
Basics of Package Installation for Email Server
=====================================================

Nginx Installation
===========================

Installation
---------------------------

* Install the Nginx Web Server to run the postfix admin and other related web interface to run.

    .. code:: bash

     apt install nginx

* Enable the Nginx Service to run on startup on machine.

	.. code:: bash

	 systemctl enable nginx

* Start the Nginx Web Server.

	.. code:: bash

	 systemctl start nginx

* Remove Default Nginx Site and add our custom site directory configuration.

	.. code:: bash

	 rm -f /etc/nginx/site-enabled/default


Setup TLS Certificates
--------------------------------------------

A modern e-mail server can't be operated seriously without TLS certificates. We will use Let's Encrypt
certificates for this purpose, as they are free and yet accepted by all browsers, mail clients and
operating systems. If you already have valid certificates, you can use them instead.

* Install letsencrypt package on through ubuntu repository.

	.. code:: bash

	 apt install certbot
	 
* Create directory and allocate necessary permission for letsencrypt domain verification.

	.. code:: bash

	 mkdir -p /var/lib/letsencrypt/.well-known
	 chgrp www-data /var/lib/letsencrypt
	 chmod g+s /var/lib/letsencrypt

* Add mentioned below snippets for Letsencrypt configuration under **/etc/nginx/snippets/letsencrypt.conf**.

	.. code:: bash

	 location ^~ /.well-known/acme-challenge/ {
	 allow all;
	 root /var/lib/letsencrypt/;
	 default_type "text/plain";
	 try_files $uri =404;
	 }

* Generate DH Params

	.. code:: bash

	 openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048

* Create mentioned below snippets for SSL configuration under **/etc/nginx/snippets/ssl.conf**.

	.. code:: bash

	 ssl_dhparam /etc/ssl/certs/dhparam.pem;
	 ssl_session_timeout 1d;
	 ssl_session_cache shared:SSL:50m;
	 ssl_session_tickets off;
	 ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
	 ssl_ciphers 'ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS';
	 ssl_prefer_server_ciphers on;
	 ssl_stapling on;
	 ssl_stapling_verify on;
	 resolver 8.8.8.8 8.8.4.4 valid=300s;
	 resolver_timeout 30s;
	 add_header Strict-Transport-Security "max-age=15768000; includeSubdomains; preload";
	 add_header X-Frame-Options SAMEORIGIN;
	 add_header X-Content-Type-Options nosniff;

* Add mentioned below our custom site directory configuration into **/etc/nginx/site-enabled/postfix** as per our need.

	:: 

	 server {
		listen [::]:80 default_server;
		root /var/www/html;
		index index.php index.html index.htm index.nginx-debian.html;
		server_name mail.hoppr.in;
		location / {
                try_files $uri $uri/ =404;
		}
		location ~ \.php$ {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/run/php/php7.2-fpm.sock;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                include fastcgi_params;
		}
		location /rspamd/ {
			proxy_pass http://127.0.0.1:11334/;
			proxy_set_header Host $host;
			proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		}
		# redirect server error pages to the static page /50x.html
		error_page 500 502 503 504 /50x.html;
		location = /50x.html {
                root /var/www/html;
		}
		location ~ /\.ht {
                deny all;
		}
		include snippets/letsencrypt.conf;
	 }
	 server {
		listen 443 ssl http2;
		root /var/www/html;
		index index.php index.html index.htm index.nginx-debian.html;
		server_name mail.hoppr.in;
		location / {
                try_files $uri $uri/ =404;
		}
		location ~ \.php$ {
			include snippets/fastcgi-php.conf;
            fastcgi_pass unix:/run/php/php7.2-fpm.sock;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            include fastcgi_params;
		}
		location /rspamd/ {
		proxy_pass http://127.0.0.1:11334/;
		proxy_set_header Host $host;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		}
		# redirect server error pages to the static page /50x.html
		error_page 500 502 503 504 /50x.html;
		location = /50x.html {
			root /var/www/html;
		}
		location ~ /\.ht {
                deny all;
		}
		ssl_certificate /etc/letsencrypt/live/mail.hoppr.in/fullchain.pem;
		ssl_certificate_key /etc/letsencrypt/live/mail.hoppr.in/privkey.pem;
		ssl_trusted_certificate /etc/letsencrypt/live/mail.hoppr.in/chain.pem;
		include snippets/ssl.conf;
		include snippets/letsencrypt.conf;
	 }

* Restart nginx for effective configuration

	.. code:: bash
	
	 systemctl restart nginx

* Generate certificate using below command.

	 certbot certonly --standalone --rsa-key-size 4096 -d mail.hoppr.in -d imap.hoppr.in -d smtp.hoppr.in

* And letsencrypt certificate valid only for 90 days, so add cron jobs to auto renewal.

	.. code:: bash

	 certbot renew --pre-hook "systemctl stop nginx" --post-hook "systemctl start nginx" --renew-hook "systemctl reload nginx; systemctl reload dovecot; systemctl reload postfix"


PHP Packages Installation 
============================

* Below mentioned PHP packages required to run the php related tools which is used for postfix Email Server.

	.. code:: bash

	 apt install php-imap php-mbstring php7.2-imap php7.2-mbstring  php-fpm php-mysql

* Set Timezone as per our local TimeZone on php configuration(/etc/php/7.2/fpm/php.ini) under value of **date.timezone**.

	.. code:: bash

	 date.timezone = Asia/Calcutta

* And Restart php to take effective

	.. code:: bash
	
	 systemctl restart php7.2-fpm


MySQL Installation
===========================

The mail server's virtual users and passwords are stored in a MySQL database. Dovecot and Postfix require this data. Follow the steps below to
create the database tables for virtual users, domains and aliases.


	.. code:: bash
	
	 apt install mysql-server

* Set Password for root user of MySQL.

	.. code:: bash

	 mysql_secure_installation

	Answer Y at the following prompts to secure mysql.

		* Change the root password?.
		* Remove anonymous users?.
		* Disallow root login remotely?.
		* Remove test database and access to it?.
		* Reload privilege tables now?.