# demo-openshift-rhsso
Assets to demonstrate usage of Red Hat Single Sign-On on Openshift Online Pro

## Configure your environment

First you need to [configure your environment](./docs/setup.md)

## Demonstration

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

Go to https://secure-sso-demo-rhsso.e4ff.pro-eu-west-1.openshiftapps.com/ to see RH-SSO running.
Click on the Administration Console link.
Login using default credentials (admin/password).
You should be logged into the "Demo" realm.
Look at the list of client, it is almost empty (except RH-SSO own clients).

Them look at the Realm Settings menu entry and go to Keys tabs.
Copy the Public Key (of RSA).

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

And you can try to contact the URL: (adjust suffix according to your account) 
https://secure-eap-app-demo-rhsso.suffix.pro-eu-west-1.openshiftapps.com/rest/hello

Now if we try :
https://secure-movies-backend-demo-rhsso.e4ff.pro-eu-west-1.openshiftapps.com/movies-backend/api/movies
We got "Unauthorized" response.

You're likely to be prompted for untrusted CA because we generated it ourselves

After this, under RH-SSO realm Demo, you need to create a role "demo" and an user "demo-user" into the realm demo.
You also have to map the role "demo" to the user "demo-user".
You also want to set a password (non-temporary) to the user.

But but... we are still Unauthorized:
- because we are not providing credentials.
But but... we are not redirected to RH-SSO ?
- because the client EAP is set to bearer-only.
(time to check the client list)

Untrusted CA - (twice, first for secure-eap then for secure-sso).

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

Look at the clients in RH-SSO demo Realm. You will need to register the movies-frontend client.
Client ID: movies-frontend
Root URL: https://movies-frontend-demo-rhsso.e4ff.pro-eu-west-1.openshiftapps.com/

Then you can go to https://movies-frontend-demo-rhsso.e4ff.pro-eu-west-1.openshiftapps.com
Redirection occurs.
Log with demo-user/password
Redirected back to the application.
Try to input value to the application makes calls to the backend.

