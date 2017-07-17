---
title: Ubuntu 配置 Gerrit 与 Apache
date: 2017-05-26 16:07:15
categories: Gerrit
tags: [Ubuntu, Gerrit, Apache]
---

## Gerrit 相关配置

### Review Site 配置文件
``` ini
[gerrit]
	basePath = git
	serverId = 9e2f813e-92e4-43a7-a3ad-8c16bc7492d0
	canonicalWebUrl = http://xxx.com/gerrit/
[database]
	type = mysql
	hostname = localhost
	database = reviewdb
	username = gerrit2
[index]
	type = LUCENE
[auth]
	type = HTTP
	logoutUrl = http://someone:xxx@xxx.com/gerrit/
[receive]
	enableSignedPush = false
[sendemail]
	smtpServer = smtp.ym.163.com
	smtpServerPort = 25
	smtpUser = gerrit@xxx.com
	from = gerrit@xxx.com
	sslVerify = false
[container]
	user = gerrit2
	javaHome = /usr/lib/jvm/java-8-oracle/jre
[sshd]
	listenAddress = *:29418
[httpd]
	listenUrl = proxy-http://*:8081/gerrit/
[cache]
	directory = cache
[gitweb]
	type = gitweb
	cgi = /usr/lib/cgi-bin/gitweb.cgi
```

## Apache 配置文件
下面的配置文件包含了：
  - http://xxx.com
  - http://xxx.com/gerrit/
  - http://xxx.com/redmine/
``` apache
<VirtualHost *>
    ServerName xxx.com
    DocumentRoot /var/www/html

    #
    # Gerrit configurations
    # Access http://xxx.com/gerrit/
    #

    ProxyRequests Off
    ProxyVia Off
    ProxyPreserveHost On

    <Proxy http://localhost:8081/gerrit/>
        Order deny,allow
        Allow from all
    </Proxy>

    # Access limited
    <Location "/gerrit/">
        AuthType Basic
        AuthName "Gerrit Code Review"
        Require valid-user
        AuthBasicProvider file
        AuthUserFile "/home/passwords"
    </Location>

    # Can access without athorization
    <Location "/gerrit/external">
        # All access controls and authentication are disabled in this directory
        Satisfy Any
        Allow from all
    </Location>

    AllowEncodedSlashes On
    ProxyPass /gerrit/ http://localhost:8081/gerrit/ nocanon

    #
    # Redmine configurations
    # Rewrite xxx.com/redmine/ to xxx.com:8080/redmine/
    #

    RewriteEngine On
    RewriteCond %{REQUEST_URI} ^/redmine/ [NC]
    RewriteRule /redmine/(.*) http://xxx.com:8080/redmine/$1 [R=permanent,L]
</VirtualHost>

# vim: syntax=apache ts=4 sw=4 sts=4 sr noet
```