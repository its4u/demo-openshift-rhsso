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

## Disclaimer

The following demonstration is intended for demonstrate purpose only.
It uses dummy passwords and self-signed certificates you don't want for anything other than a demonstration.
Choices have been made for default values in templates JSON files and in the following commands so they match. There are more advanced way to deal with those values but again the purpose is to demonstrate installation to an audience.

## Demonstration

### Create RH-SSO server

On Windows, run Powershell as administrator.

```
$domainsuffix = "e4ff"

oc new-project demo-rhsso

oc create -n demo-rhsso -f jboss-image-streams.json
oc create -n demo-rhsso -f demo-sso71-https.json

openssl req -new -newkey rsa:4096 -x509 -keyout demo.key -out demo.crt -days 365 -subj "/CN=demo.ca" -passout pass:password

keytool -genkeypair -alias https -keyalg RSA -keystore sso-https.jks --dname "CN=secure-sso-demo-rhsso.$domainsuffix.pro-eu-west-1.openshiftapps.com" -storepass password -keypass password
 
keytool -certreq -keyalg rsa -alias https -keystore sso-https.jks -file sso-https.csr -storepass password
 
openssl x509 -req -CA demo.crt -CAkey demo.key -in sso-https.csr -out sso-https.crt -days 365 -CAcreateserial -passin pass:password
 
keytool -import -file demo.crt -alias demo.ca -keystore sso-https.jks -noprompt -storepass password
 
keytool -import -file sso-https.crt -alias sso-https -keystore sso-https.jks -noprompt -storepass password
 
keytool -import -file demo.crt -alias demo.ca -keystore sso-truststore.jks -noprompt -storepass password
 
keytool -genseckey -alias jgroups -storetype JCEKS -keystore sso-jgroups.jceks -storepass password -keypass password 

oc secret new sso-https-secret sso-https.jks -n demo-rhsso
 
oc secret new sso-jgroups-secret sso-jgroups.jceks -n demo-rhsso
 
oc secret new sso-truststore-secret sso-truststore.jks -n demo-rhsso
 
oc create serviceaccount sso-service-account -n demo-rhsso
 
oc policy add-role-to-user view system:serviceaccount:demo-rhsso:sso-service-account -n demo-rhsso
 
oc secrets link sso-service-account sso-https-secret sso-jgroups-secret sso-truststore-secret -n demo-rhsso
```

Under the OpenShift Online console, you can then add to project an RH-SSO server.
The default values should be okay.

### Create EAP Server

```
$domainsuffix = "e4ff"
oc create -n demo-rhsso -f demo-eap70-sso-s2i.json

keytool -genkeypair -alias https -keyalg RSA -keystore eap-https.jks --dname "CN=secure-eap-app-demo-rhsso.e4ff.pro-eu-west-1.openshiftapps.com" -storepass password -keypass password
 
keytool -certreq -keyalg rsa -alias https -keystore eap-https.jks -file eap-https.csr -storepass password
 
openssl x509 -req -CA demo.crt -CAkey demo.key -in eap-https.csr -out eap-https.crt -days 365 -CAcreateserial -passin pass:password
 
keytool -import -file demo.crt -alias demo.ca -keystore eap-https.jks -noprompt -storepass password
 
keytool -import -file eap-https.crt -alias eap-https -keystore eap-https.jks -noprompt -storepass password
 
keytool -import -file demo.crt -alias demo.ca -keystore eap-truststore.jks -noprompt -storepass password
 
keytool -genseckey -alias jgroups -storetype JCEKS -keystore eap-jgroups.jceks -storepass password -keypass password
 
keytool -import -file demo.crt -alias demo.ca -keystore eap-saml.jks -noprompt -storepass password
 
keytool -import -trustcacerts -alias sso-https -file sso-https.crt -keystore eap-truststore.jks -noprompt -storepass password

oc secret new eap-https-secret eap-https.jks -n demo-rhsso
 
oc secret new eap-jgroups-secret eap-jgroups.jceks -n demo-rhsso
 
oc secret new eap-saml-secret eap-saml.jks -n demo-rhsso
 
oc secret new eap-truststore-secret eap-truststore.jks -n demo-rhsso

oc policy add-role-to-user view system:serviceaccount:demo-rhsso:eap-app-service-account

oc secrets link eap-app-service-account eap-https-secret eap-jgroups-secret eap-saml-secret eap-truststore-secret
```

Under the OpenShift Online console, you can then add to project an RH-SSO server.
The following default values must be changed:
- URL for SSO, you must change "suffix" to your suffix
- SSO Public Key, to be copied from RH-SSO demo Realm Keys (Public Key)


## References

Original templates come from:
https://github.com/jboss-openshift/application-templates/blob/ose-v1.3.7/jboss-image-streams.json
https://github.com/jboss-openshift/application-templates/blob/ose-v1.3.7/sso/sso71-https.json
https://github.com/jboss-openshift/application-templates/blob/ose-v1.3.7/eap/eap70-sso-s2i.json

