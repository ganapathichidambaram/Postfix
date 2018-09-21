=====================================================
About Postfix Email Server 
=====================================================

Postfix is a free email server originally developed as an alternative,simpler and more secure to sendmail.
This document will show you how to setup complete email server with postfix on Ubuntu 18.04 server.

Required Packages
==========================

* Postfix - Mail Transfer Agent (MTA)
* Dovecot - Local Delivery Agent(LDA) for incoming emails (IMAP & POP3)
* SASL - Simple Authentication and Secure Layer
* Postfixadmin - Web Interface to manage mailboxes,virtual domains and aliases.
* Nginx - Web Server to run Webmail Client & postfix admin
* MySQL - Database Storage for mail users and domains configurations.
* PHP - Web access for Postfix admin & Webmail Client

