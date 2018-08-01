# QlikSense

The items given have been tested to allow one Apache Web Server to act as a Load Balancer AND Reverse Proxy to Qlik Sense traffic. This includes the Apache componements for  Sticky / Persistent Sessions for the User Session, HTTPS/SSL, WebSocket upgrades from HTTP(S) and SAML (ADFS tested)/Header and Windows Authentication. 

Prerequisites:

- Valid 3rd Party SSL certificates will be trusted by the Apache Web Server/Qlik Sense Server (Other: ADFS server) and are configured 

  - Note: Tested with all the certificates being created by the same Certificate Authority (CA - Tinycert.org) accompanied by the same Trusted Root across all servers.
  - Note 2: Tested Qlik Sense with a valid SSL certificate bound to the Proxy Service - How to: Change the certificate used by the Qlik Sense Proxy to a custom third party certificate (https://qliksupport.force.com/articles/000005458) / https://help.qlik.com/en-US/sense/April2018/Subsystems/ManagementConsole/Content/change-proxy-certificate.htm
  - Note 3: Tested using SHA256 certificates for SAML and verify that all certificates are configured correctly with the proper Cryptographic Providers - Error 500 - Internal server error in the Hub/QMC when connecting through SAML authentication (https://qliksupport.force.com/articles/000041680)
  - Note 4: For HTTPS/SSL for Apache needs the certificate to be split into two files (.crt and .key) - Same process is used for NPrinting and is described in the article: How to convert a certificate for NPrinting to the .key and .crt files for HTTPS/SSL in the Web Console and/or the NewsStand (https://qliksupport.force.com/articles/000043517)

- Testing SAML can be done using ADFS or another SAML provider: Access to Sense installed on a server that is configured to use SAML ADFS - Configuration of ADFS can be found in the article: Quick Guide to installing ADFS for testing SAML (https://qliksupport.force.com/articles/000041751)

- Access to a server to install and configure Apache Web Server

Example Environment:

  - Qlik Sense: QlikServer1.domain.local - IP: 172.16.16.100
  - Apache Web Server: QlikServer3.domain.local  - IP: 172.16.16.102
      - Other Active Servers: AD FS: DC1.domain.local
  - Qlik Sense February 2018 GA
  - Windows 2016
  - ADFS 4.0 
  - Apache 2.4 (httpd-2.4.33-o110h-x64-vc14-r2)
  - HTTPS / SSL - SHA256 with "Microsoft Enhanced RSA and AES Cryptographic Provider" added Enabled / Active on Sense, ADFS and Apache.

Note: This documentation is only to used to validate and test while using Apache as a Reverse Web Server and Load Balancer with HTTPS/SSL is enabled. This example is under the assumption there's an understanding of the environment and having the proper permissions to perform the actions shown. Accounts used are all Local Administrators and the servers are open, with nothing blocked and no other programs installed on them. Any other versions or configurations of any software may need other steps/options/settings/etc ... that are not documented here. ​Use this at your own discretion as Qlik does NOT support Apache/OpenSSL/ADFS in their installation/configuration or use.

--

http-vhosts.conf

#Put the IP Address OR FQDN/Server Name of Qlik Sense Server as SENSE_SERVER_1 and SENSE_SERVER_2. EX: qlikserver1.domain.local
#Put the IP Address OR FQDN/Server Name of Reverse Proxy as LOCAL_ADDR. EX: 172.16.16.102
#Put the FQDN/Server Name of Reverse Proxy as APACHE_SERVER. EX: qlikserver3.domain.local
#Put the Virtual Proxy prefix as VIRTUAL_PROXY - EX: header
#Put the desired name for the Balancer configuration as BALANCER_NAME -EX: balancer

Define SENSE_SERVER_1 qlikserver1.domain.local
Define SENSE_SERVER_2 qlikserver2.domain.local
Define APACHE_SERVER qlikserver3.domain.local
Define LOCAL_ADDR 172.16.16.102
Define VIRTUAL_PROXY header
Define VIRTUAL_PROXY_1 adfsapache
Define BALANCER_NAME balancer
 
<VirtualHost *:443>

    ServerAdmin name@qlik.com
    DocumentRoot "${SRVROOT}/htdocs"
    ServerName ${LOCAL_ADDR}:443
    ServerAlias ${APACHE_SERVER}
    
    SSLProxyEngine on
    SSLEngine on
    SSLProxyCheckPeerCN off
    SSLProxyCheckPeerName off
    
   #Location of the SSL certificate used for this virtual host in their .crt and .key file format
    SSLCertificateFile  "${SRVROOT}/conf/ssl/QlikServer3Certificate.crt"
    SSLCertificateKeyFile   "${SRVROOT}/conf/ssl/QlikServer3Certificate.key"
 
    ProxyRequests Off
    ProxyPreserveHost On
    KeepAlive On
 
    RewriteEngine On
 
    # If it is a websocket request forward as websocket traffic
    RewriteCond %{HTTP:UPGRADE} ^WebSocket$ [NC]
    RewriteCond %{HTTP:CONNECTION} ^Upgrade$ [NC]
	RewriteRule ^/(.*)  balancer://wss-${BALANCER_NAME}/$1 [P,L]
	
    <Proxy *>
         Require all granted
    </Proxy>
	
	#Adding a cookie with the RouteID of a server to maintain Sticky Sessions.
	
	Header add Set-Cookie "ROUTEID=.%{BALANCER_WORKER_ROUTE}e; path=/" env=BALANCER_ROUTE_CHANGED
 
	#Balancer configuration and load balancing methods defined for SSL / HTTPS and WSS / WebSocket traffic
	
	<Proxy balancer://${BALANCER_NAME}>
		BalancerMember "https://${SENSE_SERVER_1}:443" route=1
		BalancerMember "https://${SENSE_SERVER_2}:443" route=2
		ProxySet lbmethod=byrequests
		ProxySet stickysession=ROUTEID
	</Proxy>
	
	<Proxy balancer://wss-${BALANCER_NAME}>
		BalancerMember "wss://${SENSE_SERVER_1}:443" route=1
		BalancerMember "wss://${SENSE_SERVER_2}:443" route=2
		ProxySet lbmethod=byrequests
		ProxySet stickysession=ROUTEID
	</Proxy>
	
	# 	Uncomment to open up all URLs from the Reverse Proxy / Load Balancer to Qlik Sense
	
	#	ProxyPass "/"  "balancer://${BALANCER_NAME}/" 
	#	ProxyPassReverse "/"  "balancer://${BALANCER_NAME}/"
	
	#	Can point to certain Virtual Proxies instead of opening up the environment. EX: "/header" 
	#	Note: If an ending slash is used, "/header/" "balancer://${BALANCER_NAME}/${VIRTUAL_PROXY}/" it needs to be in the URL when accessing through the browser 
	#	EX: "https://qlikserver3.domain.local/header/ - will work / https://qlikserver3.domain.local/header - will not work"
	
		ProxyPass "/${VIRTUAL_PROXY}"  "balancer://${BALANCER_NAME}/${VIRTUAL_PROXY}" 
		ProxyPassReverse "/${VIRTUAL_PROXY}"  "balancer://${BALANCER_NAME}/${VIRTUAL_PROXY}"
		
		ProxyPass "/${VIRTUAL_PROXY_1}"  "balancer://${BALANCER_NAME}/${VIRTUAL_PROXY_1}" 
		ProxyPassReverse "/${VIRTUAL_PROXY_1}"  "balancer://${BALANCER_NAME}/${VIRTUAL_PROXY_1}"
		
	#	Balancer configuration page. EX: https://qlikserver3.domain.local/balancer-manager
	
	<Location "/balancer-manager">
		SetHandler balancer-manager
		Require host ${APACHE_SERVER}
	</Location>
	
	#	Server status page. EX: https://qlikserver3.domain.local/server-status
	
	<Location /server-status>
		SetHandler server-status
		Require host ${APACHE_SERVER}
	</Location>

</Virtualhost>

-- 

httpd.conf can be used, but some modules may not be needed in your environment, please review it and disable any non-needed entries.
