---
layout: post
title:  "Setting up Shibboleth like a BOSS: Tales of my journey into the SSO Dark World"
author: israel
categories: [ 'Identity Management', 'Cloud Native'  ]
tags: [sso, ldap, identity, adfs, shibboleth ]
image: https://user-images.githubusercontent.com/2548160/178162960-63208a2b-8812-4fae-8920-8d98b09ad890.jpg

date:   2020-12-12 06:01:35 +0300
description: "Have you ever wondered how SSO work under the hood? I was battered by various complex SSO/LDAP integration issues, I decided to make it personal..."
---
Single Sign-On (SSO) is here to stay, and SSO's importance cannot be overemphasised; but have you ever wondered how SSO work under the hood? After being battered by various complex SSO/LDAP integration issues, I decided to unravel the mystery behind the working principle of this authentication method.

> *"One ring to rule them all, One ring to find them, One ring to bring them all, and in the darkness bind them" - Lord of the Rings*

## What is SSO authentication?

The Lord of the rings' analogy of "one ~~credential~~ ring to rule them" works magic every time I have had to explain what SSO is to my customers; one password for everything.

SSO is an authentication method that enables users to securely log in to one or multiple applications by using just one set of credentials. This single credential is, in most cases,  stored and managed from a central repository called LDAP. When done correctly, users should only have to log in once to access various applications and services across different domains.

There are various types of SSO authentication, such as Kerberos, Smart-card, IWA, SAML, etc.  This blog post focuses on SAML 2.0, which has three main components, as shown in the diagram below:

![SSO]({{ site.baseurl }}/assets/images/sso_comp.jpg)

I have worked on the Service Provider side of the SSO coin for nearly all of my career; thus, I have seen my fair share of SSO integration complexities and issues.

If you have felt this pain like me, you will agree that the culprit for most SSO integration issues is the Identity Provider &ndash; it's the black box that a few people know how it works and how to troubleshoot. So, I gave myself an unusual challenge to delve into the IdP black box - to understand how it works and explain the concept to my colleagues. Doing this will also mean that we have an SSO lab that can be used for internal testing and PoCs.  

I decided to use [Shibboleth IdP](https://www.shibboleth.net/) for three main reasons :

- It is an open-source solution.
- It is widely used, especially in the Education sector.
- It comes without any bells and whistles - meaning I can tinker with the configurations as much as I like.

## Setting up Shibboleth and OpenLDAP like a BOSS

This post consists mainly of my notes from this rather unusual challenge. It may also serve as a hands-on tutorial to install and configure OpenSAML, and OpenLDAP and integrate it with an application's Single Sign-On (SSO) authentication mechanism.

The diagram below depicts the end goal of this setup guide:
 
 ![SSO]({{ site.baseurl }}/assets/images/sso_flow.png)

### Hardware Requirements

- CPU: 2 Core
- RAM: 4-8 GB
- HDD: 10 GB
- Ubuntu 14 or greater

### Software Requirements

- ntp
- JRE 1.8
- Tomcat 8

### Install OpenLDAP

LDAP, or Lightweight Directory Access Protocol, is a protocol for managing related information from a centralised location by using a file and directory hierarchy.

It functions in a similar way to a relational database in certain ways and can be used to organise and store any information. LDAP is commonly used for centralised authentication.

This section will cover how to install and configure an OpenLDAP server on an Ubuntu server..

```c
sudo apt-get update
sudo apt-get install slapd ldap-utils
```

You will be asked to enter and confirm an administrator password for the administrator LDAP account.

#### Reconfigure slapd

When the installation is complete, we will need to reconfigure the LDAP package. Type the following to bring up the package configuration tool:

`sudo dpkg-reconfigure slapd`

You will be asked a series of questions about how you&#39;d like to configure the software.

- Omit OpenLDAP server configuration? No
- DNS domain name?
  - This will create the base structure of your directory path. Read the message to understand how it works.
  - There are no set rules for how to configure this, use whatever makes sense for your use case
- Organization name?
  - theCompany
- Administrator password?
  - Use the password you configured during installation, or choose another one
- Database backend to use? HDB
- Remove the database when slapd is purged? No
- Move old database? Yes
- Allow LDAPv2 protocol? No

### Connecting to LDAP server.

Download and install Java LDAP browser - [http://jxplorer.org/downloads/index.html](http://jxplorer.org/downloads/index.html)

Test your LDAP server on port 389

![ldap](https://user-images.githubusercontent.com/2548160/44148801-9aeac750-a091-11e8-9e07-d2ae9bf51621.jpg)

Alternatively, download and [setup phpLDAPAdmin](https://support.eapps.com/index.php?/Knowledgebase/Article/View/437/55/user-guide---openldap-and-phpldapadmin#installing-openldap-and-phpldapadmin), but this may require a few iterations to complete the setup.

For consistency with this guide, create an Organisation unit called UK, that way, we will refer to our Base DN as `ou=uk,dc=app,dc=com` going forward.

Create a user, or use this shell script to create LDAP users in bulk - [https://github.com/iogbole/bulk-add-ldap-users](https://github.com/iogbole/bulk-add-ldap-users)

### Install JDK 1.8.x

sudo apt-add-repository ppa:webupd8team/java
sudo apt-get update
sudo apt-get install oracle-java8-installer

Once that&#39;s done, execute:

`sudo update-alternatives --config java`

copy the jdk path, run `sudo vi /etc/environment` , then add

`JAVA_HOME='/usr/lib/jvm/java-8-oracle'`

to set the JAVA_HOME environment variable.

### Install Shibboleth

You have probably heard of OKTA, OneLogin, ADFS, PingId etc, but perhaps, not so much about Shibboleth. Shibboleth performs exact functions (and even more IMHO) as the popular Identity Providers; it is an open-source SAML Identity Provider and it has out-of-the-box support for LDAP, Kerberos, JAAS, X.509, SPNEGO. It also supports multifactor authentication with DUO, google, OpenID etc.

Modify your `/etc/hosts`:

```c
 vim /etc/hosts
<vm ip address> idp.appd.com

```

_Note, ignore the step above if your server is assigned a domain name, especially if you do not have a static IP_

Download the latest version of shib from  - [https://shibboleth.net/downloads/identity-provider](https://shibboleth.net/downloads/identity-provider/). version 3.3.1 is the latest as at the time I was tinkering with it.

```c
cd /tmp
wget https://shibboleth.net/downloads/identity-provider/3.3.1/shibboleth-identity-provider-3.3.1.zip
unzip shibboleth-identity-provider-3.3.1.zip
```

Next, change to your desired user (I am using root), then change the directory to the shibboleth directory.

```c
sudo su -
cd /tmp/shibboleth-identity-provider-3.3.1/bin & source /etc/environment
```

Execute the install script.

`./install.sh`

Sample response to the install prompts are:

```c
Installation Directory: _[/opt/shibboleth-idp]_
Hostname: _[idp.appd.com]_
SAML EntityID: _[https://idp.appd.com/idp/shibboleth]_
Attribute Scope: _[appd.com]_
Backchannel PKCS12 Password: _#PASSWORD-FOR-BACKCHANNEL#_
Re-enter password: _#PASSWORD-FOR-BACKCHANNEL#_
Cookie Encryption Key Password: _#PASSWORD-FOR-COOKIE-ENCRYPTION#_
Re-enter password: _#PASSWORD-FOR-COOKIE-ENCRYPTION#_
```

### Configure Shibboleth

Import the JST libraries to visualize the IdP status page:

```c

cd /opt/shibboleth-idp/edit-webapp/WEB-INF/lib
wget https://build.shibboleth.net/nexus/service/local/repositories/thirdparty/content/javax/servlet/jstl/1.2/jstl-1.2.jar
cd /opt/shibboleth-idp/bin ; ./build.sh -Didp.target.dir=/opt/shibboleth-idp

```

Shib&#39;s configuration needs a lot of patience, it&#39;s like peeling an onion - one layer at a time and sometimes it makes tear up.  I&#39;d open a new terminal and tail the logs in the test case sessions.

Let&#39;s get started!

Change the directory to the conf folder and let&#39;s start with the `attrribute-resolver-full.xml` file.

##### 1. attribute-resolver-full.xml

This configuration file defines how user attributes are to be constructed and then encoded prior to being sent on to a relying party trust.

Uncomment the &#39;core schema attributes&#39; block, then scroll to the bottom of the file and define LDAP server attributes as shown below:

 ```xml
   <DataConnector id="myLDAP" xsi:type="LDAPDirectory"
        ldapURL="%{idp.attribute.resolver.LDAP.ldapURL}"
        baseDN="%{idp.attribute.resolver.LDAP.baseDN}" 
        principal="%{idp.attribute.resolver.LDAP.bindDN}"
        principalCredential="thecompany"
        connectTimeout="%{idp.attribute.resolver.LDAP.connectTimeout}"
        responseTimeout="%{idp.attribute.resolver.LDAP.responseTimeout}">
        <FilterTemplate>
            <![CDATA[
                %{idp.attribute.resolver.LDAP.searchFilter}
            ]]>
        </FilterTemplate>
       <!-- <StartTLSTrustCredential id="LDAPtoIdPCredential" xsi:type="sec:X509ResourceBacked">
            <sec:Certificate>%{idp.attribute.resolver.LDAP.trustCertificates}</sec:Certificate>
        </StartTLSTrustCredential> -->
    </DataConnector>`
```

Note that the StartTLSTrustCredential block is commented out because we do not require SSL connection to our LDAP server.  You&#39;ll also need to delete this line: _useStartTLS=&quot;%{idp.attribute.resolver.LDAP.useStartTLS:true}&quot;_

Ref: [https://gist.github.com/iogbole/944996b728af4464c21ecdb7625351a1#file-attribute-resolver-full-xml](https://gist.github.com/iogbole/944996b728af4464c21ecdb7625351a1#file-attribute-resolver-full-xml)

##### 2. services.xml

This is the master configuration file that defines authentication flow. In the services.xml file, locate attribute-resolver.xml, and change it to attribute-resolver-full.xml

 ```xml 
    <util:list id ="shibboleth.AttributeResolverResources">
        <value>%{idp.home}/conf/attribute-resolver-full.xml</value>
    </util:list>
```

Ref: [https://gist.github.com/iogbole/944996b728af4464c21ecdb7625351a1#file-services-xml](https://gist.github.com/iogbole/944996b728af4464c21ecdb7625351a1#file-services-xml)

##### 3. attribute-filter.xml

This configuration file acts like shib&#39;s custom officer, it ensures only attributes that are defined are released to a Service Provider.  Add this block of code to the file, and change the value parameter to your controller&#39;s IP or URL.

 ```xml 
    <AttributeFilterPolicy id="releaseToAppD">
        <PolicyRequirementRule xsi:type="Requester" value="http://192.168.33.1:8090/controller" />
        <AttributeRule attributeID="givenName">
            <PermitValueRule xsi:type="ANY" />
        </AttributeRule>

        <AttributeRule attributeID="uid">
            <PermitValueRule xsi:type="ANY" />
        </AttributeRule>

        <AttributeRule attributeID="mail">
            <PermitValueRule xsi:type="ANY" />
        </AttributeRule>
    </AttributeFilterPolicy>
```

In this case, we will be releasing giveName, uid and mail attributes to the Service Provider which is the Service Provider (SP) in this context.

Further, if you need to provision more that one relying party trust (i.e 2 or more Service Providers), use an OR condition in the PolicyRequirementRule element instead:

 ```xml 
         <PolicyRequirementRule xsi:type="OR">
            <Rule xsi:type="Requester" value="http://192.168.33.1:8090/controller" />
            <Rule xsi:type="Requester" value="http://192.168.33.2:8090:8090/controller" />
        </PolicyRequirementRule>
```
The value&#39;s value should be the same as what you&#39;ve specified in the service provider metedata.

Ref:  [https://gist.github.com/iogbole/944996b728af4464c21ecdb7625351a1#file-attribute-filter-xml](https://gist.github.com/iogbole/944996b728af4464c21ecdb7625351a1#file-attribute-filter-xml)

#### 4. ldap.properties

  -   Comment out line 19 : idp.authn.LDAP.trustCertificates
  -   Comment out line 22: idp.authn.LDAP.trustStore
  -   Update idp.authn.LDAP.baseDN  to whatever your baseDn value is, mine is  ou=uk,dc=appd,dc=com
  -   Update idp.authn.LDAP.bindDN and idp.authn.LDAP.bindDNCredential with LDAP username and password:   uid=israel,ou=system  and thecompany
  -   Change line 40 to reflect your base DN: idp.authn.LDAP.dnFormat  = uid=%s,ou=uk,dc=appd,dc=com
  -   Change line 24 to :   idp.authn.LDAP.returnAttributes  = cn,givenName

Ref: [https://gist.github.com/iogbole/944996b728af4464c21ecdb7625351a1#file-ldap-properties](https://gist.github.com/iogbole/944996b728af4464c21ecdb7625351a1#file-ldap-properties)

##### 5. idp.properties

Locate _idp.encryption.optional_ property at line 60. uncomment it and change the value to true. This is necessary because the Controller requires SAML responses to be signed and encrypted. It took a while to figure this out.

> #If true, encryption will happen whenever a key to use can be located, 
>
> #but failure to encrypt won&#39;t result in request failure.
>
> idp.encryption.optional = true

Ref : [https://gist.github.com/iogbole/944996b728af4464c21ecdb7625351a1#file-idp-properties](https://gist.github.com/iogbole/944996b728af4464c21ecdb7625351a1#file-idp-properties)

##### 6. controller.xml

This is where it got a bit challenging, unlike most applications I have worked with in the past, The Service Provider I used to test this out, AppD, does not provide or generate it&#39;s own SAML metadata. This limitation makes it slightly difficult to integrate the Controller with non-cloud-based IdPs like ADFS, PingFed, and especially Shibboleth. I was able to generate a working Controller metadata after a few iterations, and it can be re-used by changing the controller&#39;s URL. Download it from  [https://gist.github.com/iogbole/944996b728af4464c21ecdb7625351a1#file-controller-xml](https://gist.github.com/iogbole/944996b728af4464c21ecdb7625351a1#file-controller-xml), modify it i.e change the controller URL and copy it to _/opt/shibboleth-idp/metadata_

##### 7. metadata-providers.xml

This file defines the IdP&#39;s own metadata location and all other service provider&#39;s  metadata.
Scroll to the bottom of the file and add this line - to define the controller&#39;s metadata as shown in 6 above.

 ```xml 
 <MetadataProvider id="LocalMetadata" xsi:type="FilesystemMetadataProvider" metadataFile="%{idp.home}/metadata/controller.xml"/>
```
repeat the above line (but change the providerID) for each service provider (ie. controller) you&#39;d like to add.

Ref: [https://gist.github.com/iogbole/944996b728af4464c21ecdb7625351a1#file-metadata-provider-xml](https://gist.github.com/iogbole/944996b728af4464c21ecdb7625351a1#file-metadata-provider-xml)

##### 8. access-control.xml

The IdP has a status page that gives a high-level detail of your configurations. In order to access this page from a remote machine, you&#39;d need to provide the IP address of your machine

(whatismyip.com) in the allowed IP address ranges.

```xml 
<entry key="AccessByIPAddress">
<bean id="AccessByIPAddress" parent="shibboleth.IPRangeAccessControl"
p:allowedRanges="#{ {'127.0.0.1/32', '::1/128', 'your-internet-ip/32' } }" />
</entry>
```

##### 9. logback.xml

Last but not the least, bump IdP logging level to temporarily DEBUG or TRACE to help you figure out any issues that may arise.

> <!-- Logging level shortcuts. -->
>
> <variable name="idp.loglevel.idp" value="TRACE" />
>
> <variable name="idp.loglevel.ldap" value="TRACE" />
>
> <variable name="idp.loglevel.messages" value="INFO" />
>
> <variable name="idp.loglevel.encryption" value="INFO" />
>
> <variable name="idp.loglevel.opensaml" value="INFO" />
>
> <variable name="idp.loglevel.props" value="INFO" />
>

Ref: [https://gist.github.com/iogbole/944996b728af4464c21ecdb7625351a1#file-logback-xml](https://gist.github.com/iogbole/944996b728af4464c21ecdb7625351a1#file-logback-xml)

##### 10. Install NTP

```c
sudo apt-get install ntp
sudo service ntp restart
```
### Install Tomcat 8

Download tomcat 8 from  [https://tomcat.apache.org/download-80.cgi](https://tomcat.apache.org/download-80.cgi) . unzip it to /opt/tomcat or where ever you prefer.

1. Set **CATALINA_HOME** in your /etc/environment file

    `sudo vi /etc/environment`

    paste: `export CATALINA_HOME='/opt/apache-tomcat-8.5.15'`

    `source /etc/environment`

2. Copy this [sh file](https://gist.github.com/iogbole/944996b728af4464c21ecdb7625351a1#file-setenv-sh)  into CATALINA_HOME/bin

3. Whilst in CATALINA_HOME/bin, make the following scripts executable.

    `sudo chmod +x setenv.sh catalina.sh shutdown.sh startup.sh`

4. `vi server.xml` and comment out port 8080, and add the following block of code to open port 8433.

    ```xml
    <Connector protocol="org.apache.coyote.http11.Http11NioProtocol" port="8443" maxThreads="200" scheme="https" secure="true" SSLEnabled="true" keystoreFile="/opt/shibboleth-idp/credentials/idp-backchannel.p12" keystorePass="thecompany"
    clientAuth="false" sslProtocol="TLS"/>
    ```

    _Note : KeystorePass should be the same as the backchannel password you provided whilst installing shibboleth_

    Ref  [https://gist.github.com/iogbole/944996b728af4464c21ecdb7625351a1#file-server-xml](https://gist.github.com/iogbole/944996b728af4464c21ecdb7625351a1#file-server-xml)

5. In `idp.xml`

    In order to run the IdP, Tomcat must be informed of the location of the IdP war file. This should be done with a context descriptor by creating the file CATALINA_BASE/conf/Catalina/localhost/idp.xml and placing the following content in it:

    ```xml
    <Context docBase="/opt/shibboleth-idp/war/idp.war"
    privileged="true"
    antiResourceLocking="false"
    unpackWAR="false"
    swallowOutput="true" />
   ```

_Note: Unpacking the WAR file is not part of the default settings, I added it to optimize tomcat startup time_

#### TEST CASE #1

1. start tomcat:  `sudo ./catalina.sh run`
2. Access tomcat home page [https://tomcat-server-fqdn:8443](https://server-fqdn:8443)
3. Access idp status page at [https://tomcat-server-fqdn:8443/idp/status](https://tomcat-server-fqdn:8443/idp/status)

    ![shibb](https://user-images.githubusercontent.com/2548160/44152318-24ed39b4-a09d-11e8-84de-c0f4663b7fe9.png)

4. Use IDP_HOME/log/idp-process.log and catalina.out logs to debug any issues until the 2 test cases pass.

### SP Integration
Note: I used an on-premise version of AppDynamics as a Service Provider to test this out. The steps should however be similar to any other SAML 2.0 SP integration. I would probably use NextCloudPi to test it out later too. 

Copy the IdP signing certificate (without any whitespace) by executing :

`sudo cat /opt/shibboleth-idp/credentials/idp-signing.crt`

Next, log in to the Service Provider (in my case, the AppDynamics controller) and follow the instructions as shown below:

![appdcontroller](https://user-images.githubusercontent.com/2548160/44152163-b30fe102-a09c-11e8-8d93-21dda2e1b2e5.png)

#### Connecting the dots...

1. The login URL is in two parts:   [https://idp.localhost.com:8443/idp/profile/SAML2/POST/SSO?providerId=http://192.168.33.1:8090/controller](https://idp.localhost.com:8443/idp/profile/SAML2/POST/SSO?providerId=http://192.168.33.1:8090/controller). It consists of the IdP URL and the providerID parameter. The value of this parameter must correspond with the EntityID value in the controller.xml file, and it should be the Controller&#39;s URL
2. SAML Attribute Mappings: These values correspond with the attributes that were specified in the attribute-filter.xml, uid is the user id in LDAP, givenName is the user&#39;s first name and mail are.. duh!
3. Assign a default role to SAML users, save the settings and let&#39;s authenticate!

#### TEST CASE #2

Attempt to log in to your Service Provider. I used the following steps whilst testing from a browser with an AppDynamics controller. 

1. AppDynamics Controller successfully redirects to IdP login page
2. IdP successfully authenticates the user against your LDAP server
3. IdP successfully redirects back the controller
4. AppDynamics controller is able to decode SAML response.

See the attached video for successful validation of the above test cases.

![2017-05-26_12-00-57 1](https://user-images.githubusercontent.com/2548160/44155067-2ff75c0c-a0a4-11e8-9926-4adff74d265a.gif)

**Troubleshooting tools**

1. https://idp.ssocircle.com/sso/toolbox/samlDecode.jsp : Decode SAML response
2. SAML Tracer - Live debug of SAML responses and transposes
3. https://www.samltool.com/validate_response.php : Validate SAML Responses and generate SAML metadata.


**References:**

1. Troubleshooting SSO SAML Integration - https://prezi.com/si2iwzwwnogy/saml-sso-basics-integration-troubleshooting/ 

2. Shibboleth and OpenDS installation - https://prezi.com/nc8_vxgzdryv/saml-ssoldap/ 

3. Shibboleth documentation - https://wiki.shibboleth.net/confluence/display/IDP30/Configuration 

4. How to use ADFS to restrict access to a relying party trust  - https://blogs.technet.microsoft.com/israelo/2015/03/27/restricting-access-to-yammer-using-adfs-claims-transformation-rule/ 

5. How to Install OpenLDAP - https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-a-basic-ldap-server-on-an-ubuntu-12-04-vps 


