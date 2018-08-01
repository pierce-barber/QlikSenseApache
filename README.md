# QlikSense


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

Note: This documentation is only to used to validate and test while using Apache as a Reverse Web Server and Load Balancer with HTTPS/SSL is enabled. This example is under the assumption there's an understanding of the environment and having the proper permissions to perform the actions shown. Accounts used are all Local Administrators and the servers are open, with nothing blocked and no other programs installed on them.​Read the entire documentation to verify access and understanding of all actions stated within prior to starting the install and configuration. Any other versions or configurations of any software may need other steps/options/settings/etc ... that are not documented here. ​Use this at your own discretion as Qlik does NOT support Apache/OpenSSL/ADFS in their installation/configuration or use.
