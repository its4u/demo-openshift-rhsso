# demo-openshift-rhsso
Assets to demonstrate usage of Red Hat Single Sign-On on Openshift Online Pro

## Configure your environment

In order to run the demonstration you need:
- An OpenShift Online account
- A JDK properly installed (check java -version on command line)
- If your locale is not en, you may want to set JAVA_TOOL_OPTION environment variable to -Duser.language=en (with -D in the value) to avoid weird certificate errors with keytool
if your locale is not en)
- The OpenShift command line tool "oc". See command line tools documentation of your OpenShift Online account
- OpenSSL installed and in your PATH
- Java keytool in your PATH

Git clone this project to a local folder

### Disclaimer

The following demonstration is intended for demonstrate purpose only.
It uses dummy passwords and self-signed certificates you don't want for anything other than a demonstration.
Choices have been made for default values in templates JSON files and in the following commands so they match. There are more advanced ways to deal with those values but again the purpose here is to demonstrate installation to an audience.

### Domain suffix

Exposed route in Openshift Online have a structure:
name.suffix.zone.openshiftapps.com
- name depends on your application and is configurable per application
- suffix is linked to your account, it is a short identifier (so far, the best to find is to deploy an http server and look at its public URL)
- zone, depends on the location your account (pro-eu-west-1)

This documentation needs that you adjust suffix from time to time.

## Demonstration

### Create certificates

On Windows, run Powershell as administrator.

```
$domainsuffix = "e4ff"

openssl req -new -newkey rsa:4096 -x509 -keyout demo.key -out demo.crt -days 365 -subj "/CN=demo.ca" -passout pass:password

keytool -genkeypair -alias https -keyalg RSA -keystore sso-https.jks --dname "CN=secure-sso-demo-rhsso.$domainsuffix.pro-eu-west-1.openshiftapps.com" -storepass password -keypass password
 
keytool -certreq -keyalg rsa -alias https -keystore sso-https.jks -file sso-https.csr -storepass password
 
openssl x509 -req -CA demo.crt -CAkey demo.key -in sso-https.csr -out sso-https.crt -days 365 -CAcreateserial -passin pass:password
 
keytool -import -file demo.crt -alias demo.ca -keystore sso-https.jks -noprompt -storepass password
 
keytool -import -file sso-https.crt -alias sso-https -keystore sso-https.jks -noprompt -storepass password
 
keytool -import -file demo.crt -alias demo.ca -keystore sso-truststore.jks -noprompt -storepass password
 
keytool -genseckey -alias jgroups -storetype JCEKS -keystore sso-jgroups.jceks -storepass password -keypass password 

keytool -genkeypair -alias https -keyalg RSA -keystore eap-https.jks --dname "CN=secure-eap-app-demo-rhsso.$domainsuffix.pro-eu-west-1.openshiftapps.com" -storepass password -keypass password
 
keytool -certreq -keyalg rsa -alias https -keystore eap-https.jks -file eap-https.csr -storepass password
 
openssl x509 -req -CA demo.crt -CAkey demo.key -in eap-https.csr -out eap-https.crt -days 365 -CAcreateserial -passin pass:password
 
keytool -import -file demo.crt -alias demo.ca -keystore eap-https.jks -noprompt -storepass password
 
keytool -import -file eap-https.crt -alias eap-https -keystore eap-https.jks -noprompt -storepass password
 
keytool -import -file demo.crt -alias demo.ca -keystore eap-truststore.jks -noprompt -storepass password
 
keytool -genseckey -alias jgroups -storetype JCEKS -keystore eap-jgroups.jceks -storepass password -keypass password
 
keytool -import -file demo.crt -alias demo.ca -keystore eap-saml.jks -noprompt -storepass password
 
keytool -import -trustcacerts -alias sso-https -file sso-https.crt -keystore eap-truststore.jks -noprompt -storepass password
```

### Create RH-SSO server

Connect your command line tool to OpenShift (https://console.pro-eu-west-1.openshift.com/console/command-line)

The following commands will:
- create the OpenShift project named demo-rhsso
- add to this project catalog the imagestreams and templates directly from this GitHub project (if you make modification, you need to adjust the path your files)
- add the required keystores and truststores to the project

```
oc new-project demo-rhsso

oc create -n demo-rhsso -f https://raw.githubusercontent.com/its4u/demo-openshift-rhsso/master/jboss-image-streams.json
oc create -n demo-rhsso -f https://raw.githubusercontent.com/its4u/demo-openshift-rhsso/master/demo-sso71-https.json

oc secret new sso-https-secret sso-https.jks -n demo-rhsso
oc secret new sso-jgroups-secret sso-jgroups.jceks -n demo-rhsso
oc secret new sso-truststore-secret sso-truststore.jks -n demo-rhsso
oc create serviceaccount sso-service-account -n demo-rhsso
oc policy add-role-to-user view system:serviceaccount:demo-rhsso:sso-service-account -n demo-rhsso
oc secrets link sso-service-account sso-https-secret sso-jgroups-secret sso-truststore-secret -n demo-rhsso
```

Now you can create the Red Hat Single Sign-On pod:
- Under the OpenShift Online console, you can then add to project an RH-SSO server (from project demo-rhsso catalog).
- The default values should be okay as, for convience, they are hardcoded in the template.


### Create EAP Server

Now, it is time to create the EAP server to deploy your API backend application.

The following commands will:
- add to the project the tempate from EAP with SSO adapter
- add the required keystores and truststores to the project

```
oc create -n demo-rhsso -f https://raw.githubusercontent.com/its4u/demo-openshift-rhsso/master/demo-eap70-sso-s2i.json

oc secret new eap-https-secret eap-https.jks -n demo-rhsso
oc secret new eap-jgroups-secret eap-jgroups.jceks -n demo-rhsso
oc secret new eap-saml-secret eap-saml.jks -n demo-rhsso
oc secret new eap-truststore-secret eap-truststore.jks -n demo-rhsso
oc create serviceaccount eap-service-account -n demo-rhsso
oc policy add-role-to-user view system:serviceaccount:demo-rhsso:eap-service-account
oc secrets link eap-service-account eap-https-secret eap-jgroups-secret eap-saml-secret eap-truststore-secret
```

Under the OpenShift Online console, you can then add to project an "Red Hat JBoss EAP 7.0 + Single Sign-On (with https)".
The following default values must be changed to match your environment:
- Custom http Route Hostname, you must change "suffix" to your domain suffix
- Custom https Route Hostname, you must change "suffix" to your domain suffix
- URL for SSO, you must change "suffix" to your domain suffix
- SSO Public Key, to be copied from RH-SSO demo Realm Keys (Public Key). Go to https://secure-sso-demo-rhsso.suffix.pro-eu-west-1.openshiftapps.com/auth/ (adjust suffix according to your account) in the Realm Demo, Realm settings, Keys, and copied the rsa-generated public key.

After this, under RH-SSO realm Demo, you need to create a role "demo" and an user into the realm demo.
You also have to map the role to the user.
You also want to set a password (non-temporary) to the user.

And you can try to contact the URL: (adjust suffix according to your account) 
https://secure-eap-app-demo-rhsso.suffix.pro-eu-west-1.openshiftapps.com/rest/hello

You're likely to be prompted for untrusted CA because we generated ourselves (twice, first for secure-eap then for secure-sso).

You can check that the RH-SSO adapter prevents you from accessing the API by trying to access the following URL (Unauthorized)
https://secure-movies-backend-demo-rhsso.e4ff.pro-eu-west-1.openshiftapps.com/movies-backend/api/movies


### Create NodeJS Server

This final deploys the frontend application.

The only required command:
- creates the nodejs template with default values to use our front git repository

```
oc create -n demo-rhsso -f https://raw.githubusercontent.com/its4u/demo-openshift-rhsso/master/demo-nodejs6-s2i-https.json
```

Under the OpenShift Online console, you can then add to project an "Node.js" from the project catalog.
Then you can go to https://movies-frontend-demo-rhsso.e4ff.pro-eu-west-1.openshiftapps.com

## References

Original templates come from:
https://github.com/jboss-openshift/application-templates/blob/ose-v1.3.7/jboss-image-streams.json

https://github.com/jboss-openshift/application-templates/blob/ose-v1.3.7/sso/sso71-https.json

https://github.com/jboss-openshift/application-templates/blob/ose-v1.3.7/eap/eap70-sso-s2i.json

https://github.com/openshift/nodejs-ex/blob/master/openshift/templates/nodejs.json
