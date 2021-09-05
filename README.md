# SPF, DKIM, and DMARC Validation with Postfix on Ubuntu 20.04

This article details how to add SPF, DKIM, and DMARC validation to an Ubuntu 20.04 email server running Postfix.

SPF (Sender Policy Framework), DKIM (DomainKeys Identified Mail), and DMARC (Domain-based Message Authentication, Reporting and Conformance) are email authentication protocols that are becoming increasingly necessary to run a working email server. Even if you have valid MX, A, and PTR DNS records for your mail server, you may find that your emails are getting rejected as spam by major providers like Gmail and Outlook.com. Implementing SPF, DKIM, and DMARC will allow your messages to get through.

In addition to preventing your emails from getting bounced by other email providers, these protocols also give you a couple of additional tools to identify forged or illegitimate incoming emails.

This guide assumes you're running Postfix as your mail transport agent and know the basics of email troubleshooting and DNS. These instructions were tested against Ubuntu 20.04 and will work as written with my [Ubuntu Email Server](https://github.com/geoffstratton/Ubuntu-Email-Server) series of articles. Let's get started.

## Initial setup: install the opendkim and postfix-spf packages and environment:

```
root@ubuntu:/# apt install opendkim opendkim-tools postfix-policyd-spf-python
```

Add the postfix user to the opendkim group (just edit /etc/group or else run the gpasswd command):

```
root@ubuntu:/# gpasswd -a postfix opendkim
```

Set up the opendkim directory, key, and permissions:

```
root@ubuntu:/# mkdir -p /etc/opendkim/keys ; opendkim-genkey -b 2048 -d domainX.com -D /etc/opendkim/keys/domainX.key -s XXXX -v ; chown -hR opendkim:opendkim /etc/opendkim ; chmod -R 600 /etc/opendkim/keys
```

## Set up SPF

Create a TXT DNS record for your domain, leave the "name" blank, and give it a value like this:

```
v=spf1 mx ~all
```

Here is [some further discussion](https://mailtrap.io/blog/spf-records-explained/) of the SPF flags and options.

Next, set up Postfix to test incoming emails for SPF validity. Edit /etc/postfix/main.cf and add a couple of directives: 

```
policyd-spf_time_limit = 3600   <----------
smtpd_recipient_restrictions =
   permit_mynetworks,
   permit_sasl_authenticated,
   reject_unauth_destination,
   check_policy_service unix:private/policyd-spf   <----------
```

Then edit /etc/postfix/master.cf and tell Postfix to start up the policyd-spf daemon when Postfix starts:
 
```
policyd-spf  unix  -       n       n       -       0       spawn
    user=policyd-spf argv=/usr/bin/policyd-spf
```

Restart Postfix (`systemctl restart postfix`) and send a test email from your domain. You could send your test to a major provider like Gmail or to a tool like [https://www.mail-tester.com](https://www.mail-tester.com). If SPF is working, the emails you send out will show headers like these (you can see this easily in Gmail webmail -> Show Original):

```
Received-SPF: pass (google.com: domain of me@mydomain designates 255.255.255.255 as permitted sender) client-ip=255.255.255.255;
Authentication-Results: mx.google.com;
       spf=pass (google.com: domain of me@mydomain designates 255.255.255.255 as permitted sender) smtp.mailfrom=me@mydomain.com
```

If you don't see any SPF header at all, or your test reports a fail, your /var/log/mail.log may tell you where the problem is, and running `dig mydomain.com txt` from your server's command line will show you the TXT DNS records for your domain. You may also just need to wait a while -- it can take some time for your new DNS record to propagate to the wider internet.

The Postfix-SPF handoff on your server will also test incoming messages for SPF validity. When this is working, you'll see a header on incoming mails like this (example from Amazon):

```
Received-SPF: Pass (mailfrom) identity=mailfrom; client-ip=54.240.15.17; helo=a15-17.smtp-out.amazonses.com; envelope-from=202109041901209bed980622544ade94d7d0afeca0p0na-c3ut7qv6c8llc0@bounces.amazon.com; receiver=<UNKNOWN>
```

This would seem to be a valid email from Amazon.com. You wouldn't want to miss any emails from Jeff Bezos.

## Set Up DKIM

To set up DKIM, first edit /etc/opendkim.conf (most of these are defaults, but there are a few changes):

```
# This is a basic configuration that can easily be adapted to suit a standard
# installation. For more advanced options, see opendkim.conf(5) and/or
# /usr/share/doc/opendkim/examples/opendkim.conf.sample.

# Log to syslog
Syslog                  yes
# Required to use local socket with MTAs that access the socket as a non-
# privileged user (e.g. Postfix)
# UMask                 007

# Sign for example.com with key in /etc/dkimkeys/dkim.key using
# selector '2007' (e.g. 2007._domainkey.example.com)
#Domain                 example.com
#KeyFile                        /etc/opendkim/keys/mydomain.com
KeyTable                refile:/etc/opendkim/key.table
SigningTable            refile:/etc/opendkim/signing.table
Selector                XXXX  # Change this to something meaningful
InternalHosts           refile:/etc/opendkim/trusted.hosts
ExternalIgnoreList      refile:/etc/opendkim/trusted.hosts

AutoRestart         yes
AutoRestartRate     10/1M
Background          yes
DNSTimeout          5
SignatureAlgorithm  rsa-sha256

# Commonly-used options; the commented-out versions show the defaults.
Canonicalization        relaxed/simple
Mode                    sv
SubDomains              no

# Socket smtp://localhost
#
# ##  Socket socketspec
# ##
# ##  Names the socket where this filter should listen for milter connections
# ##  from the MTA.  Required.  Should be in one of these forms:
# ##
# ##  inet:port@address           to listen on a specific interface
# ##  inet:port                   to listen on all interfaces
# ##  local:/path/to/socket       to listen on a UNIX domain socket
#
#Socket                  inet:8892@localhost
Socket                local:/var/spool/postfix/opendkim/opendkim.sock

##  PidFile filename
###      default (none)
###
###  Name of the file where the filter should write its pid before beginning
###  normal operations.
#

# Always oversign From (sign using actual From and a null From to prevent
# malicious signatures header fields (From and/or others) between the signer
# and the verifier.  From is oversigned by default in the Debian pacakge
# because it is often the identity key used by reputation systems and thus
# somewhat security sensitive.
OversignHeaders         From

##  ResolverConfiguration filename
##      default (none)
##
##  Specifies a configuration file to be passed to the Unbound library that
##  performs DNS queries applying the DNSSEC protocol.  See the Unbound
##  documentation at http://unbound.net for the expected content of this file.
##  The results of using this and the TrustAnchorFile setting at the same
##  time are undefined.
##  In Debian, /etc/unbound/unbound.conf is shipped as part of the Suggested
##  unbound package

# ResolverConfiguration     /etc/unbound/unbound.conf

##  TrustAnchorFile filename
##      default (none)
##
## Specifies a file from which trust anchor data should be read when doing
## DNS queries and applying the DNSSEC protocol.  See the Unbound documentation
## at http://unbound.net for the expected format of this file.

TrustAnchorFile       /usr/share/dns/root.key

##  Userid userid
###      default (none)
###
###  Change to user "userid" before starting normal operation?  May include
###  a group ID as well, separated from the userid by a colon.
#
UserID                opendkim
```

Create the signing table at /etc/opendkim/signing.table. This file tells opendkim that outgoing messages from this list of domains should be signed with the private key specified for them in /etc/opendkim/key.table. The format here is the selector field from opendkim.conf (above) + ._domainkey.domain.name.

```
*@domain1.com       XXXX._domainkey.domain1.com
*@domain2.com           XXXX._domainkey.domain2.com
*@domain3.com           XXXX._domainkey.domain3.com
```

Create the key table at /etc/opendkim/key.table. This maps the selector.domain value to the specific signing key that you created when you ran opendkim-genkey way back in step 2. You can use one key for all domains, or a separate key for each (I'd recommend the latter). If you need to create additional keys, run `opendkim-genkey -b 2048 -d domainX.com -D /etc/opendkim/keys/domainX.key -s XXXX -v`. Also ensure that any new key you create is readable-writeable only by opendkim (chmod 600).

```
XXXX._domainkey.domain1.com    domain1.com:XXXX:/etc/opendkim/keys/domain1.key
XXXX._domainkey.domain2.com        domain2:XXXX:/etc/opendkim/keys/domain2.key
XXXX._domainkey.domain3.com        domain3:XXXX:/etc/opendkim/keys/domain3.key
```

Next create the trusted.hosts file at /etc/opendkim/trusted.hosts. This file tells opendkim that emails originating from localhost or one of your hosted domains should not undergo DKIM validation.

```
127.0.0.1
localhost

*.domain1.com
*.domain2.com
*.domain3.com
```

When you run opendkim-genkey, it creates two files: the private key that should live in a protected place on your server, and a XXXX.txt file named after the the -s (selector) value. The .txt file contains the public key that goes in your DNS record to permit other email systems to validate your DKIM signature. So create a TXT DNS record for your domain -- name it XXXX._domainkey, and give it the value in quotes from your XXXX.txt file:

```
v=DKIM1; h=sha256;k=rsa;p=long_string_of_characters
```

Make sure you remove the additional quotes or whitespace from the strings here, or lookups against this record may not work.

Now test your setup:

```
root@ubuntu:/# opendkim-testkey -d domain1.com -s XXXX -vvv

opendkim-testkey: using default configfile /etc/opendkim.conf
opendkim-testkey: checking key 'XXXX._domainkey.domain1.com'
opendkim-testkey: key secure
opendkim-testkey: key OK
```

If you see "key not secure", it just means you're not using DNSSEC, so not a fatal problem.

Next we need to tell Postfix to pass mail through opendkim. They work together by both pointing to the socket you specified in /etc/opendkim.conf (/var/spool/postfix/opendkim/opendkim.sock). First, edit /etc/default/opendkim and change the SOCKET value to your desired one:

```
#SOCKET=local:$RUNDIR/opendkim.sock
SOCKET="local:/var/spool/postfix/opendkim/opendkim.sock"
```
 
Then edit /etc/postfix/main.cf and add a Milter configuration to the end of the file:

```
# Milter configuration
milter_default_action = accept
milter_protocol = 6
smtpd_milters = local:/opendkim/opendkim.sock
non_smtpd_milters = $smtpd_milters
```

Before you restart everything and test, I had to make one additional change to make opendkim play nice with systemd. Edit 
/lib/systemd/system/opendkim.service and remove the PID line:

```
[Service]
Type=forking
PIDFile=/var/run/opendkim/opendkim.pid  <----- Delete this
```

This will remove the "Can't open PID file /run/opendkim/opendkim.pid" that you may see when opendkim starts up. Remember to run systemctl daemon-reload to update the generator file after making this change.

Finally, restart everything and test (`systemctl restart opendkim postfix`). As with SPF, you can send email from your domain to Gmail (Show Original) or a service like [https://www.mail-tester.com](https://www.mail-tester.com) to view your results. The mail.log will also show DKIM status messages. If DKIM checks are working, you'll see "DKIM: PASS" in your tests, and incoming messages will show DKIM verification in the headers as well (again, using Amazon as an example):

```
Authentication-Results: domain1.com; dkim=pass (1024-bit key; unprotected) header.d=amazon.com etc.; dkim=pass (1024-bit key; unprotected) etc. dkim-atps=neutral 
```

For more on DKIM, check out the [OpenDKIM home page](http://www.opendkim.org/).

### DMARC Verification

With SPF and DKIM in place, we might as well add a [DMARC](https://dmarc.org/) record for additional fraud protection and more reliable email delivery. Many anti-spam systems will do DMARC lookups and, if you pass, use simplified filtering rules based on what they view as a good reputation.

To implement DMARC, just add a TXT DNS record for your domain named _dmarc and give it the following value:

```
v=DMARC1; p=none; pct=100; fo=1; rua=mailto:dmarc-reports@domain1.com
```

You can test this with dig (`dig txt +short _dmarc.domain1.com`) or by using Gmail or [https://www.mail-tester.com](https://www.mail-tester.com). For receiving and interpreting DMARC reports, [Postmark](https://dmarc.postmarkapp.com/) offers a nice service.
