---
title: "MySQL 5.0 and SSL"
date: 2014-04-20
draft: false
summary: "I needed to get an old version of MySQL server 5.0 running with SSL on Debian Etch. It was not as simple as you might have thought."
tags: ["MySQL", "Security"]
---

I needed to get an old version of MySQL server running with SSL. Thankfully, that support has been there for a long time, although on my previous try I found it rather frustrating and gave it over for some other job that needed doing.

If securing client connections to a database server is a non-negotiable requirement, I would suggest that MySQL is perhaps a poor-fit and other options, such as PostgreSQL -- according to common web-consensus and my interactions with developers would suggest -- should be first considered. While MySQL can do SSL connections, it does so in a rather poor way that leaves much to be desired.

UPDATED 2014-04-28 for MySQL 5.0 (on ancient Debian Etch).

Here is the fast guide to getting SSL on MySQL server. I'm doing this on a Debian 7 ("Wheezy") server. To complete things, I'll test connectivity from a 5.1 client as well as a reasonably up-to-date MySQL Workbench 5.2 CE, plus a Python 2.6 client; just to see what sort of pain awaits.

UPDATE: 2014-07-31 -- MySQL Workbench CE 6.1.7 is much better at connecting over SSL compared to CE 5.2 (at least, with the same config, 5.2 CE refused to establish a connection over SSL, where CE 6.1.7 had no such issues. 6.1.7 still doesn't expose an interface to require certificate validation though.
I'm not going to cover using Client SSL Certificates in this guide, although that could be useful.

(There is a brief summary at the bottom of this post)

We'll be doing the following:

1. Create your certificate request to get signed by a CA.
1. Install the key and signed certificate, making sure to set ownership and permissions carefully.
1. Add the configuration to /etc/mysql/my.cnf
1. Restart MySQL server
1. Use MySQL client to check 'show variables'.
1. Enforce users to use SSL

## Create your certificate request to get signed by a CA

```bashsession
$ openssl req -new -newkey rsa:2048 -nodes
Generating a 2048 bit RSA private key
........................................................+++
....................................................................................+++
writing new private key to 'privkey.pem'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:NZ
State or Province Name (full name) [Some-State]:Otago
Locality Name (eg, city) []:Dunedin
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Your company here
Organizational Unit Name (eg, section) []:
Common Name (eg, YOUR name) []:server.example.com
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
-----BEGIN CERTIFICATE REQUEST-----
MIIDFzCCAf8CAQAwgdExCzAJBgNVBAYTAk5aMQ4wDAYDVQQIEwVPdGFnbzEQMA4G
A1                                                            8x
Jj                                                            QD
Ex    This is a certificate request. It is basically a        Q0
NC    certificate that has yet to be signed by a CA.          EB
BQ                                                            vi
Ef    You could inspect this using a command such as          1r
mb                                                            Vv
O6      openssl req -text < this_request                      Sa
JS                                                            Rg
P5    Certificates contain a number of things, including      b3
DQ    the CN (its name), and any alternate names.             9F
W/    But this method doesn't give you the option to          fL
vn    include alternate names (but there are scripts)         8e
iF                                                            zZ
yhS2cZCTg/twJ+okkUfy92oZoypWhkJfQzPZM6HGwI2RNrcHda0kFQipulji0A0o
gBzTeOx+TKDzIy4EHjh7lhHTLs9FK1jk/A6G
-----END CERTIFICATE REQUEST-----
```

You would need to send this to whoever purchases or obtains your SSL certificates. When you do, you'll need to ask for the returned certificate to be an X.509 certificate in an PEM encoding (or just say 'suitable for Apache on Linux'). Eventually, you will get a signed certificate and CA bundle back. Some formats will deliver these combined, another thing to note is that the CA bundle may be a concatenation of PEM certificates.

## Install the key and signed certificate, making sure to set ownership and permissions carefully

**Important**: Unlike Apache; which starts with root privileges, opens its keys and does other privileged actions like binding to low-numbered ports; *then* drops privileges; MySQL seems to open its private keys *after* it has dropped its privileges. On a Debian server, the lower-privileged user is called 'mysql', and  by changing to the 'mysql' user and group. We don't want that user to be able to write to the private key or certificate, but it does need to read it. Thus, we need to change the ownership of the private key file to be user root, group mysql, and mode 0640.

Where you put the keying material doesn't often matter, but it can make a difference in general if advanced permissions systems things like SELinux are in effect, and it makes it easier when working as a team. Another benefit it that it makes it easier to update the keying material when replacing the certificate, which might also be shared with something like Apache.

On a Debian system, the private key should ideally go in /etc/ssl/private/, while on a Red Hat system, the private key should ideally go in /etc/pki/tls/private/. Similarly the certificate (including CA certificates) should go in /etc/ssl/certs/ or /etc/pki/tls/certs/ respectively. The filename doesn't have a strong common convention; I think its more useful to have a consistency within your team with regard to this.

I tend to put the private key into a file bearing the name (CN) in the certificate, with a '.key' extension. Others may call it 'something.pem'.

That said, on Debian systems, /etc/ssl/certs/ contains a lot of CA certificates, which makes me reluctant to use it (it would be harder for a colleague to find see the certificate). So on a Debian server, I'll put all of them in /etc/mysql/

So I would put the private key into a file /etc/mysql/SERVER.key with permissions root:mysql 0640.

## Key format

UPDATE 2014-04-28 If you're dealing with MySQL 5.0 on Debian Etch (ancient at this time of writing), there is another issue that can occur. This may be applicable for users of OpenSSL circa version 0.9.8 or such. If I ever meet the guy that wrote this [comment on a similar post for Ubuntu](http://askubuntu.com/a/439274), then I owe him a beer. In short, if your key file looks like this:

```plain
-----BEGIN RSA PRIVATE KEY-----
...
-----END RSA PRIVATE KEY-----
```

Then this is in PKCS#8 format, and we apparently need it in PKCS#1 format (even though I created the key-pair on the same machine). The most obvious difference is that the `RSA PRIVATE KEY` turns into `RSA KEY`. Here's how you do that; note that this version of OpenSSL will write out PKCS#1.

```bash
cd /etc/mysql/
mv SERVER.key{,.pkcs8}
openssl rsa -in SERVER.key.pkcs8 -out SERVER.key
```

I would then put the certificate into a file /etc/mysql/SERVER.crt with permissions root:root 0644

I would also put the CA bundle into a file /etc/mysql/SERVER.ca-bundle with the same permissions (this extension is not one I have seen others use)

## Add the configuration to /etc/mysql/my.cnf

```conf
ssl-cert=/etc/mysql/SERVER.crt
ssl-ca=/etc/mysql/SERVER.key
ssl-key=/etc/mysql/SERVER.key 
#mysql 5.0: don't specify ssl-cipher   (or at least, know that this may not work) 
```

## Restart MySQL and Test

```bash
service mysql stop
service mysql start
```

Use MySQL client to check `show variables`

```plain
mysql> show variables like '%ssl%';
+---------------+----------------------------+
| Variable_name | Value                      |
+---------------+----------------------------+
| have_openssl  | YES                        |
| have_ssl      | YES                        |
| ssl_ca        | /etc/mysql/SERVER.ca-bundle|
| ssl_capath    |                            |
| ssl_cert      | /etc/mysql/SERVER.crt      |
| ssl_cipher    |                            |
| ssl_key       | /etc/mysql/SERVER.key      |
+---------------+----------------------------+
7 rows in set (0.00 sec)

mysql> \s
--------------
...
SSL:                    Not in use
...
Connection:             Localhost via UNIX socket
...
```

So the next bit is to connect over SSL. Normally, I would want to test using something like `openssl s_client -connect localhost:3306`, but unlike https, the thing inside the TCP connection is not SSL, but instead it is a clear-text connection, and it will invoke some STARTTLS-like operation to bring up SSL inside the TCP connection.

```bashsession
$ openssl s_client -connect localhost:3306 </dev/null
CONNECTED(00000003)
15595:error:140770FC:SSL routines:SSL23_GET_SERVER_HELLO:unknown protocol:s23_clnt.c:607:
```

So let's go directly to the client. First, I'll try to connect to localhost; I would expect this to fail to validate the SSL connections, because 'localhost' is not a name that appears in the certificate. But as you can see, it does appear to work; why is that?

```bashsession
$ mysql --host localhost --ssl-ca=/etc/mysql/SERVER.ca-bundle
...
mysql> \s
...
SSL:                    Cipher in use is DHE-RSA-AES256-SHA
...
Connection:             Localhost via UNIX socket
...
```

Ah, so its using a Unix-domain socket to connect to localhost... presumably it doesn't care to check SSL certificates for such connections. Let's specify TCP. Remember, I expect to fail and complain of a certificate mismatch.

```bashsession
$ mysql --host=localhost --protocol=TCP --ssl-ca=/etc/mysql/SERVER.ca-bundle
...
mysql> \s
...
SSL:                    Cipher in use is DHE-RSA-AES256-SHA
...
Connection:             localhost via TCP/IP
...
```

So its still not complaining about a certificate mismatch. That's a naughty SSL client! *(or rather, a dangerous default)*

Update 2014-04-21. Actually, if I dig a bit more into the MySQL documentation, I see the following (my emphasis):

> **--ssl-verify-server-cert** This option is available for client programs only, not the server. It causes the client to check the server's Common Name value in the certificate that the server sends to the client. The client verifies that name against the host name the client uses for connecting to the server, and the connection fails if there is a mismatch. This feature can be used to prevent man-in-the-middle attacks. **Verification is disabled by default**. This option was added in MySQL 5.1.11.

In order to make use of it, you would likely need to put it into your ~/.my.cnf on the client, and check (or instruct) your client to consume it. MySQL Workbench doesn't seem to have that option in its GUI.

```ini
[client]
ssl-verify-server-cert=1
```

Let's connect to ourselves using our own hostname. Note that this is different user (combination of username and IP address/pattern) in MySQL, as will mean we are using a different account (remote account)... which I haven't configured yet.

```bashsession
$ mysql --host=SERVER.example.com --ssl-ca=/etc/mysql/SERVER.ca-bundle
ERROR 1045 (28000): Access denied for user 'root'@'SERVER.example.com' (using password: YES)
```

Let's create ourselves a user that we can use for logging into the server FROM the server, and 'grant' (really, restrict in this case I think), the ability to log in via SSL. For lesser clients, you could use '%' as the host. The access privileges are too high though. In this example, we're only authenticating the server using SSL, not the client.

```mysqlsession
mysql> create user 'root'@'SERVER.example.com' identified by 'SOMEPASSWORD';
Query OK, 0 rows affected (0.00 sec)

mysql> grant all privileges on * to 'root'@'SERVER.example.com' require SSL;
Query OK, 0 rows affected (0.00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)
```

So now let's test.

```bashsession
$ mysql --host=SERVER.example.com --ssl-ca=/etc/mysql/SERVER.ca-bundle  -u root -p
Enter password:
...
mysql> \s
...
Current user:           root@SERVER.example.com
SSL:                    Cipher in use is DHE-RSA-AES256-SHA
...
Connection:             SERVER.example.com via TCP/IP
...
```

I would expect that connecting to the IP address (instead sy by hostname) should also fail to validate the certificate, but that fails as well. Hopefully that behaviour will be improved in a later release.

Let's drop that user (it has too many privileges for a remote account), and replace it with something more suited to testing the case at hand.

```mysqlsession
mysql> drop user 'root'@'SERVER.example.com';
Query OK, 0 rows affected (0.00 sec)

mysql> create user 'cameron'@'%' identified by 'SOMEPASSWORD';
Query OK, 0 rows affected (0.00 sec)

mysql> grant usage on *.* to 'cameron'@'%' require SSL;
Query OK, 0 rows affected (0.00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)
```

Okay, so let's move to testing with a different client: MySQL Workbench

## MySQL Workbench

Create a new connection with the following settings. Note that some of the SSL settings are only applicable to client-side SSL.

{{<figure
    title="MySQL Workbench SSL Standard Parameters"
    src="/post-images/distracted-it/mysql-ssl-standard-parameters.png"
    alt="Just a basic TCP/IP connection in the Parameters tab for the connection settings..."
>}}
{{<figure
    title="MySQL Workbench SSL Advanced Parameters"
    src="/post-images/distracted-it/mysql-ssl-advanced-parameters.png"
    alt="...specify 'Use SSL if available' and 'SSL CA File'"
>}}

Having to provide the CA certificate (chain) implies that MySQL Workbench doesn't use the system's CA certificate store. If I didn't tick 'Use SSL if available', it wouldn't let my user in because of the "require SSL" in the GRANT statement.

Next client: Python

## Python 2.6 on Linux using mysqldb module (and a diagnostic diversion)

My client, who I'm setting this up for, is using Python 2.7 on Windows... presumably she is using the 32-bit version of Python, because there doesn't appear to be a 64-bit version of the mysqldb module for Windows. So I'm doing my own testing with Python 2.6 on a 64-bit RHEL 6, because I'm more interested in learning how to use the mysqldb API to make SSL connections, rather than doing any meaningful testing.

If going the Windows route, you may well prefer to go the pure-Python route of the MySQL/Oracle supported MySQL Connector/Python, which should be easier to install.

So, firstly on RHEL 6, we need to make sure we have the appropriate package installed. Here are two commands, one to see if the package has been installed (you can run this as any user), and another to install the package (which requires root privileges)

```bash
rpm -qa | grep -i mysql   # notice case-insensitive search. Look for MySQL-python
yum install MySQL-python
```

You'll need to make sure you have the MySQL server's CA certificate handy. I'm copying mine into the same directory I'm using to do my testing, and so I will refer to it as ./SERVER.ca.crt  -- the name doesn't really matter other than for readability (so in my opinion, it does matter).

The key difference in using the API is that you need to add a keyword argument 'ssl' to the `MySQLdb.connect` method. This must be a dictionary, where the keys are from the C API function `mysql_ssl_set` (you can find such a list at http://dev.mysql.com/doc/refman/5.0/en/mysql-ssl-set.html). Briefly, the keys are 'key', 'cert', 'ca', 'capath', 'cipher'. Okay, so let's try it.

```python
Python 2.6.6 (r266:84292, Nov 21 2013, 10:50:32)
[GCC 4.4.7 20120313 (Red Hat 4.4.7-4)] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import MySQLdb
>>> conn = MySQLdb.connect(host='SERVER.example.com', user='cameron', passwd='MYPASSWORD', ssl={'ca': '/home/cameron/tmp/SERVER.ca.crt'})
Traceback (most recent call last):
  File "", line 1, in 
  File "/usr/lib64/python2.6/site-packages/MySQLdb/__init__.py", line 81, in Connect
    return Connection(*args, **kwargs)
  File "/usr/lib64/python2.6/site-packages/MySQLdb/connections.py", line 187, in __init__
    super(Connection, self).__init__(*args, **kwargs2)
_mysql_exceptions.OperationalError: (2026, 'SSL connection error')
```

Well, that's not particularly useful.... the documentation does say that if there is no support for SSL in the client an exception is raised, but I'm fairly confident that won't be the issue here. I wonder if we can turn on some debugging. Let's have a look at the Python documentation for MySQLdb: run `help(MySQLdb)` in a Python shell. As expected, an Operational Error is a miscellaneous external error. The exception does say 'SSL connection error', so something about the SSL connection failed. If you want to be the SSL support goto-guy in your organisation, its useful to determine how to figure out what that is.

```plain
...
    class OperationalError(DatabaseError)
     |  Exception raised for errors that are related to the database's
     |  operation and not necessarily under the control of the programmer,
     |  e.g. an unexpected disconnect occurs, the data source name is not
     |  found, a transaction could not be processed, a memory allocation
     |  error occurred during processing, etc.
...
```

### Diagnosis with Wireshark

My first step towards diagnosis is to determine if this is a client or server-side issue (if I can). I could fire up Wireshark or tcpdump, and determine if I can see a complete TCP transaction forming... and I do see a complete transaction, then some data going back and forth. Looking at it in Wireshark is instructive:

1. I see the server issue a MySQL Server Greeting message, advertising that it can do SSL
1. I see the client issue a MySQL Login Request; it does not specify a user to login as, but in the Client Capabilities, it does set the flag for 'Switch to SSL after handshake', so the client can do SSL, and it is attempting to do SSL.
1. The rest of the packets are just labelled TCP... if I use Wireshark's handy 'Follow TCP Stream', then I see the server transmitting the SSL Server certificate to the client.
  1. At this point, the TCP connection on port 3306 is no longer the MySQL protocol, but an SSL protocol. So right-click on one of the packets of that exchange, and select Decode As...
  1. Decode this particular TCP connection as SSL and click OK. You should now see SSL or TLS packets.
  1. In my case, I see a Client Hello, Server Hello, Certificate, Key Exchange and then an Alert. The alert states 'Fatal, Description: Unknown CA' from the client to the server.
1. I then the client abort the TCP connection.

So it seems that it's not picking up the CA certificate I gave it, or there is a problem with the CA certificate.

Using `strace -f -e trace=file,read python`, I can see that Python has opened and has read the contents of the CA certificate, so it must be something to do with the CA certificate itself (or it has opened the file, read the contents, but still hasn't used it). Here are the lines of output that show this behaviour:

```plain
open("/home/cameron/tmp/SERVER.ca.crt", O_RDONLY) = 5
read(5, "-----BEGIN CERTIFICATE-----\nMIIE"..., 4096) = 1704
```

In this case, I only provided the first CA certificate, the complete chain has three CAs, with the lowest CA (furthest from the root) being the certificate I supplied. Hypotheses:

- **Either** I need to provide more of the chain. If so, this should be considered faulty behaviour, or the (sparse) documentation for 'ca' means something other than "if you find a server certificate transitively trusted by one of these CA's then its okay."
  - Considering I gave the previous clients all three certificates, this is the biggest point of difference, so I'll make this client the same. 
  - Note that in order to build a chain of trust, the client should only need to be go as far up the provided chain until it finds a CA it knows about (and trusts). Since I gave it the server's own CA certificate, that should have been sufficient to build a valid chain of trust, and is more preferable to providing the entire chain, as it means that I'm only trusting one CA to do their job properly, not a lot of CA's (which often do not do their job properly).
- **Or** MySQL client library is not using the CA certificate supplied
- **Or** MySQL client library is using the systems's list of trusted CA certificates

And indeed, it was the first one. So to use MySQL with Python and the MySQLdb library, you would need to provide more of the certificate chain.

I believe that perhaps MySQL must generally (where it is deployed with SSL) be used with Self-Signed certificates or with a local CA -- particularly where client-certificates are used.

I think later I may also investigate using SSL with client-certificates. But at the moment, that is not a requirement.

Anyway, let's look at the equivalent of '\s' in the mysql command-line client: SHOW STATUS. Here's a simple piece of code which does that:

```python
cursor = conn.cursor()
cursor.execute("show status")
for (k,v) in cursor:
    print k, v
```

I prints out about 291 rows. Here are some of the more salient rows:

```plain
Ssl_accept_renegotiates 0
Ssl_accepts 0
Ssl_callback_cache_hits 0
Ssl_cipher DHE-RSA-AES256-SHA
Ssl_cipher_list DHE-RSA-AES256-SHA:AES256-SHA:DHE-RSA-AES128-SHA:AES128-SHA:AES256-RMD:AES128-RMD:DES-CBC3-RMD:DHE-RSA-AES256-RMD:DHE-RSA-AES128-RMD:DHE-RSA-DES-CBC3-RMD:RC4-SHA:RC4-MD5:DES-CBC3-SHA:DES-CBC-SHA:EDH-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC-SHA
Ssl_client_connects 0
Ssl_connect_renegotiates 0
Ssl_ctx_verify_depth 0
Ssl_ctx_verify_mode 0
Ssl_default_timeout 500
Ssl_finished_accepts 0
Ssl_finished_connects 0
Ssl_session_cache_hits 0
Ssl_session_cache_misses 0
Ssl_session_cache_mode Unknown
Ssl_session_cache_overflows 0
Ssl_session_cache_size 0
Ssl_session_cache_timeouts 0
Ssl_sessions_reused 0
Ssl_used_session_cache_entries 0
Ssl_verify_depth 0
Ssl_verify_mode 0
Ssl_version TLSv1
```

Python 2.7 with MySQL Connector on Windows

**ADDED 2014-07-31**

If you haven't already, download the MSI from

Documentation for that module can be found at dev.mysql.com:
http://dev.mysql.com/doc/connector-python/en/index.html

Here's some very brief code:

```python
>>> import mysql.connector
>>> connection = mysql.connector.connect(host='mydbms.example.com', database='mydb', user='myuser', password='mypass', ssl_ca=r'C:\Users\me\Desktop\mydbms.ca-bundle', ssl_verify_cert=True, buffered=True)
>>> cursor = connection.cursor()
>>> cursor.execute("select 1")
>>> cursor.fetchone()
(1,)
```

## Summary

- To use SSL to provide privacy and server authentication, you need to:
  - on the server:
    - get a certificate, making sure the permission are set correctly
    - configure the 'ssl-cert', 'ssl-ca', and 'ssl-key' elements in my.cnf
    - restart MySQL service
    - GRANT a user the ability to REQUIRE SSL  (which takes away the ability to not use SSL).
  - on the client:
    - pass it the complete CA chain
      - this may involve adding an extra connection argument
    - tick to enable the use of SSL (this may be implicit by the above)
- But beware that MySQL isn't a particularly good example of how a client should do SSL:
  - It doesn't appear to validate the server-name in the certificate (very bad)
  - It doesn't appear to do a particularly good job of establishing trust by using CA certificates.
  - The MySQL Connector/Python plugin, provided by Oracle may do a better job with regard to some of this (but I have not used or tested it).
  - Note that I have not tested this behaviour on the most latest version released of MySQL clients or server provided by Oracle (the vendor).

I won't trust MySQL to do a suitable job of securing data (using SSL) in a mission-critical environment. If you have to have access to the backend, I would suggest first requiring access over a VPN would be a useful requirement -- that would get you a good level of auditing (who, when, where-from) that MySQL doesn't provide well-enough. This would be particularly useful if the accounts in the database are shared with others, as the VPN should be using individual accounts.

> Bear in mind that this is an old post for a very old version of MySQL; hopefully the situation has improved, but, as you should for anything, test the basics when you deploy SSL.

## Other good sources of information

[The Grumpy Troll's blog post on MySQL, SSL/TLS and Ubuntu](http://bridge.grumpy-troll.org/2013/04/mysql-ssltls-and-ubuntu/)
Provides a very useful post for those dealing with Ubuntu, and further points out that it generally also a requirement (in newer releases than what I have to deal with, I think) to specify the cipher to use. Also mentions that on Ubuntu, with AppArmour in place, by default it wants the keying material to exist in /etc/mysql/


