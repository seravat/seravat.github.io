---
title: Deploy RHPAM on Openshift 3.11 with SSO as auth provider
tags: [Red Hat, Openshift, PAM, Process Automation Manager]
style: 
color: 
description: SSO can be a very useful security central point for microservices architectures
---

In this example we deploy a RHPAM 7.4 authoring environment to Openshift 3.11 with authentication handled by RHSSO

## 1. Deploy PAM on Openshift

https://access.redhat.com/documentation/en-us/red_hat_process_automation_manager/7.4/html/deploying_a_red_hat_process_automation_manager_authoring_environment_on_red_hat_openshift_container_platform/index

---

## 1.1 Pre-requisites

### 1.1.1 Verify that Openshift has the required image streams. 

```
$ oc get imagestreamtag -n openshift | grep rhpam74-businesscentral
$ oc get imagestreamtag -n openshift | grep rhpam74-kieserver
```

### 1.1.2 If the output of the command is empty, we have to import the image streams into Openshift

- Follow the steps to be able to get images from the [red hat registry](https://access.redhat.com/RegistryAuthentication#registry-service-accounts-for-shared-environments-4)

- Select the OpenShift Secret tab and click the link under Download secret to download the YAML secret file.

- View the downloaded file and note the name that is listed in the name: entry.

- Enter the following commands:

```
oc create -f <file_name>.yaml
oc secrets link default <secret_name> --for=pull
oc secrets link builder <secret_name> --for=pull

Replace <file_name> with the name of the downloaded file and <secret_name> with the name that is listed in the name: entry of the file.
```

- Download the rhpam-7.4.0-openshift-templates.zip product deliverable file from the [Software Downloads](https://access.redhat.com/jbossnetwork/restricted/listSoftware.html?downloadType=distributions&product=rhpam&productChanged=yes) page and extract the rhpam74-image-streams.yaml file.

- Enter the following command:

```
oc create -f rhpam74-image-streams.yaml
```

### 1.1.3 Creating a new Openshift Project

Either throught the console or oc client

```
oc new-project rhpam-authoring
```

### 1.1.4 Creating the secrets for Process Server

Create an SSL certificate for HTTP access to Process Server and provide it to your OpenShift environment as a secret.

Generate an SSL keystore with a private and public key for SSL encryption for Process Server. For more information on how to create a keystore with self-signed or purchased SSL certificates, see [Generate a SSL Encryption Key and Certificate](https://access.redhat.com/documentation/en-US/JBoss_Enterprise_Application_Platform/6.1/html-single/Security_Guide/index.html#Generate_a_SSL_Encryption_Key_and_Certificate)

- Generate jks Keystore

```
$ keytool -genkeypair -alias jboss -keyalg RSA -keystore keystore.jks -storepass mykeystorepass --dname "CN=jsmith,OU=Engineering,O=mycompany.com,L=Raleigh,S=NC,C=US"

Verify the keystore:
$ keytool -list -keystore keystore.jks
```

- Generate a certificate signing request using the public key from the keystore

```
$ keytool -certreq -keyalg RSA -alias jboss -keystore keystore.jks -file certreq.csr

Test the newly generated certificate:
openssl req -in certreq.csr -noout -text
```

- Optional: Submit your certificate to a Certificate Authority (CA).

A Certificate Authority (CA) can authenticate your certificate so that it is considered trustworthy by third-party clients. The CA supplies you with a signed certificate, and optionally with one or more intermediate certificates.

- Our case: Export a self-signed certificate from the keystore.

If you only need it for testing or internal purposes, you can use a self-signed certificate. You can export one from the keystore you created in step 1 as follows:

```
keytool -export -alias jboss -keystore keystore.jks -file server.crt
```

- Import the signed certificate, along with any intermediate certificates.

```
$ keytool -import -keystore keystore.jks -alias intermediateCA -file intermediate.ca

$ keytool -import -alias jboss -keystore keystore.jks -file server.crt

Test that your certificates imported successfully:
$ keytool -list -keystore keystore.jks
```

- Use the oc command to generate a secret named kieserver-app-secret from the new keystore file

```
oc create secret generic kieserver-app-secret --from-file=keystore.jks
```

### 1.1.5 Creating the secrets for Business Central

Same steps as `Creating the secrets for Process Server`, just change the name of the secret

```
oc create secret generic businesscentral-app-secret --from-file=keystore.jks
```

### 1.1.6 Persistent Volumes

We need to either have dynamic PV provisioning or previously created PVs.

If using GlusterFS, please follow this [configuration - step 2.5](https://access.redhat.com/documentation/en-us/red_hat_process_automation_manager/7.4/html/deploying_a_red_hat_process_automation_manager_authoring_environment_on_red_hat_openshift_container_platform/dm-openshift-prepare-con)


### 1.1.7 Preparing a Maven mirror repository for offline use

If your Red Hat OpenShift Container Platform environment does not have outgoing access to the public Internet, you must prepare a Maven repository with a mirror of all the necessary artifacts and make this repository available to your environment.

Follow [step 2.6](https://access.redhat.com/documentation/en-us/red_hat_process_automation_manager/7.4/html/deploying_a_red_hat_process_automation_manager_authoring_environment_on_red_hat_openshift_container_platform/dm-openshift-prepare-con)


---

## 1.2 Deployment

We will use the `rhpam74-authoring-ha.yaml` template to deploy a HA authoring environment. It creates a MySQL pod to provide the database for the Process Server - if we want another database, then we have to [customize the template](https://access.redhat.com/documentation/en-us/red_hat_process_automation_manager/7.4/html/deploying_a_red_hat_process_automation_manager_authoring_environment_on_red_hat_openshift_container_platform/environment-authoring-con#environment-authoring-ha-modify-proc)

### 1.2.1 Get the templates and import them into Openshift

Download the rhpam-7.4.0-openshift-templates.zip product deliverable file from the [Software Downloads](https://access.redhat.com/jbossnetwork/restricted/listSoftware.html?downloadType=distributions&product=rhpam&productChanged=yes) page of the Red Hat Customer Portal.

- Extract the templates from the zip file

- Investigate the template rhpam74-authoring-ha.yaml and import it into the openshift namespace (if its not there already)

```
$ oc get template -n openshift | grep rhpam74-*

If the return is empty (done by cluster-admin):
$ oc create -f rhpam74-authoring-ha.yaml -n openshift
```

### 1.2.2 Choose the required parameters for the template

- __BUSINESS_CENTRAL_HTTPS_SECRET__: The name of the secret for Business Central - `businesscentral-app-secret`

- __KIE_SERVER_HTTPS_SECRET__: The name of the secret for Process Server - `kieserver-app-secret`

- __BUSINESS_CENTRAL_HTTPS_NAME__: The name of the certificate in the keystore - `jboss`

- __BUSINESS_CENTRAL_HTTPS_PASSWORD__: The password for the keystore - `mykeystorepass`

- __KIE_SERVER_HTTPS_NAME__: The name of the certificate in the keystore - `jboss`

- __KIE_SERVER_HTTPS_PASSWORD__: The password for the keystore - `mykeystorepass`

- __APPLICATION_NAME__: The name of the OpenShift application. OpenShift uses the application name to create a separate set of deployment configurations, services, routes, labels, and artifacts - `rhpam`

- __IMAGE_STREAM_NAMESPACE__: The namespace where the image streams are available. If the image streams were already available in your OpenShift environment (see Section 2.1, “Ensuring the availability of image streams and the image registry”), the namespace is openshift - `openshift`

- __KIE_ADMIN_USER__ and __KIE_ADMIN_PWD__: The user name and password for the administrative user. If you want to use the Business Central to control or monitor any Process Servers other than the Process Server deployed by the same template , you must set and record the user name and password. - `admin/r3dh4t1!`

- __KIE_SERVER_USER__ and __KIE_SERVER_PWD__: The user name and password that a client application can use to connect to any of the Process Servers. - `client-user/r3dh4t1!`


### 1.2.3 Choose the optional parameters for the template

### 1.2.3.1 Setting an optional Maven repository

When configuring the template to deploy an authoring environment, if you want to place the built KJAR files into an external Maven repository, you must set parameters to access the repository.

- __MAVEN_REPO_URL__: The URL for the Maven repository - `http://nexus-nexus.apps.8aa4.example.opentlc.com/repository/maven-releases/`
- __MAVEN_REPO_ID__: An identifier for the Maven repository. The default value is repo-custom - `nexus-maven-releases`
- __MAVEN_REPO_USERNAME__: The username for the Maven repository - `admin`
- __MAVEN_REPO_PASSWORD__: The password for the Maven repository - `admin123`

> IMPORTANT
> To export or push Business Central projects as KJAR artifacts to the external Maven repository, you must also add the repository information in the > pom.xml file for every project. For information about exporting Business Central projects to an external repository, see [Packaging and deploying a Red Hat Process Automation Manager project](https://access.redhat.com/documentation/en-us/red_hat_process_automation_manager/7.4/html-single/packaging_and_deploying_a_red_hat_process_automation_manager_project#maven-external-export-proc_packaging-deploying)

### 1.2.3.2 Specifying credentials to access the built-in Maven repository for an authoring environment

When configuring the template to deploy an authoring environment, if you want to use the Maven repository that is built into Business Central and to connect additional Process Servers to the Business Central, you must configure credentials for accessing this Maven repository. You can then use these credentials to configure the Process Servers.

Also, if you are configuring RH-SSO or LDAP authentication, you must set the credentials for the built-in Maven repository to a username and password configured in RH-SSO or LDAP. This setting is required so that the Process Server can access the Maven repository.

- __BUSINESS_CENTRAL_MAVEN_USERNAME__: The user name for the built-in Maven repository in Business Central. - `mavenUser`
- __BUSINESS_CENTRAL_MAVEN_PASSWORD__: The password for the built-in Maven repository in Business Central. - `r3dh4t1!`

### 1.2.3.3 Configuring access to a Maven mirror in an environment without a connection to the public Internet for an authoring environment

When configuring the template to deploy an authoring environment, if your OpenShift environment does not have a connection to the public Internet, you must configure access to a Maven mirror that you set up according to [Section 2.6, “Preparing a Maven mirror repository for offline use”.](https://access.redhat.com/documentation/en-us/red_hat_process_automation_manager/7.4/html/deploying_a_red_hat_process_automation_manager_authoring_environment_on_red_hat_openshift_container_platform/dm-openshift-prepare-con#offline-repo-proc)

- __MAVEN_MIRROR_URL__: The URL for the Maven mirror repository. This URL must be accessible from a pod in your OpenShift environment. - `http://nexus-nexus.apps.8aa4.example.opentlc.com/repository/maven-public/`
- __MAVEN_MIRROR_OF__: The value that determines which artifacts are to be retrieved from the mirror. For instructions about setting the mirrorOf value, see [Mirror Settings](https://maven.apache.org/guides/mini/guide-mirror-settings.html) in the Apache Maven documentation. The default value is external:*,!repo-rhpamcentr; with this value, Maven retrieves artifacts from the built-in Maven repository of Business Central directly and retrieves any other required artifacts from the mirror. If you configure an external Maven repository (MAVEN_REPO_URL), change MAVEN_MIRROR_OF to exclude the artifacts in this repository, for example, external:*,!repo-custom. Replace repo-custom with the ID that you configured in MAVEN_REPO_ID.

### 1.2.3.4 Specifying the Git hooks directory for an authoring environment

You can use Git hooks to facilitate interaction between the internal Git repository of Business Central and an external Git repository

- __GIT_HOOKS_DIR__: The fully qualified path to a Git hooks directory, for example, /opt/kie/data/git/hooks. You must provide the content of this directory and mount it at the specified path. For instructions about providing and mounting the Git hooks directory using a configuration map or a persistent volume, see Section 3.3, [“(Optional) Providing the Git hooks directory”.](https://access.redhat.com/documentation/en-us/red_hat_process_automation_manager/7.4/html/deploying_a_red_hat_process_automation_manager_authoring_environment_on_red_hat_openshift_container_platform/environment-authoring-con#githooks-proc)

### 1.2.3.5 Setting parameters for RH-SSO authentication for an authoring environment

If you want to use RH-SSO authentication, complete the following additional configuration when configuring the template to deploy an authoring environment.

- __Prerequisites__
  * A realm for Red Hat Process Automation Manager is created in the RH-SSO authentication system.

    {% include elements/figure.html image="/assets/images/pam_sso/realm.png" caption="Create Realm for PAM" %}

  * User names and passwords for Red Hat Process Automation Manager are created in the RH-SSO authentication system. For a list of the available roles, see [Chapter 4, Red Hat Process Automation Manager roles and users](https://access.redhat.com/documentation/en-us/red_hat_process_automation_manager/7.4/html/deploying_a_red_hat_process_automation_manager_authoring_environment_on_red_hat_openshift_container_platform/roles-users-con). The following users are required in order to set the parameters for the environment:

      - An administrative user with the __kie-server__,__rest-all__,__admin__ roles. This user can administer and use the environment. Process Servers use this user to authenticate with Business Central. ---- for Business Central
      - A server user with the __kie-server__,__rest-all__,__user__ roles. This user can make REST API calls to the Process Server. Business Central uses this user to authenticate with Process Servers. --- for Process Server

  * Clients are created in the RH-SSO authentication system for all components of the Red Hat Process Automation Manager environment that you are deploying. The client setup contains the URLs for the components. You can review and edit the URLs after deploying the environment. Alternatively, the Red Hat Process Automation Manager deployment can create the clients. However, this option provides less detailed control over the environment.

    Create the clients for Business Central and Process Server - after deploying PAM we will come back and add the urls of the routes

    ```
    Client Protocol: openid-connect
    Access Type: confidential
    Valid Redirect URIs: http://dummypam.org (to change)
    Base URL: http://dummypam.org (to change)
    ```

    {% include elements/figure.html image="/assets/images/pam_sso/clients.png" caption="Clients for Business Central and Process Server" %}
    
    Lets create the necessary client-roles for both clients

    {% include elements/figure.html image="/assets/images/pam_sso/client_roles.png" caption="Create Client Roles for PAM" %}

    Create admin and client-user users and give them passwords (r3dh4t1!)

    {% include elements/figure.html image="/assets/images/pam_sso/roles.png" caption="Create Users for PAM" %}

    Associate the client-roles to the created users. Associate both the business-central and the process-server client roles

    {% include elements/figure.html image="/assets/images/pam_sso/admin_client_roles.png" caption="Associate business-central client roles to admin" %}

    {% include elements/figure.html image="/assets/images/pam_sso/admin_client_roles1.png" caption="Associate process-server client roles to admin" %}

    Since we are already on SSO lets also retrive the Business Central and Process Server Client Names and Client Secrets

    {% include elements/figure.html image="/assets/images/pam_sso/client_details.png" caption="Associate process-server client roles to admin" %}

    ```
    Business Central
    client name: business-central
    client secret: 49dfc82f-301a-4f42-96bc-1da278474302

    Process Server
    client name: process-server
    client secret: 8d33123a-a54c-4adf-b64f-d2b055974c11
    ```
  
- __Parameters__

  * __KIE_ADMIN_USER__ and __KIE_ADMIN_PASSWORD__ parameters of the template to the user name and password of the administrative user that you created in the RH-SSO - `admin/r3dh4t1!`

  * __KIE_SERVER_USER__ and __KIE_SERVER_PASSWORD__ parameters of the template to the user name and password of the server user that you created in the RH-SSO - `client-user/r3dh4t1!`

  * __SSO_URL__: The URL for RH-SSO - `https://sso-sso-73.apps.8aa4.example.opentlc.com/auth`

  * __SSO_REALM__: The RH-SSO realm for Red Hat Process Automation Manager. - `PAM`

  * __SSO_DISABLE_SSL_CERTIFICATE_VALIDATION__: Set to true if your RH-SSO installation does not use a valid HTTPS certificate. - `true` 

  * __BUSINESS_CENTRAL_SSO_CLIENT__: The RH-SSO client name for Business Central - `business-central`

  * __BUSINESS_CENTRAL_SSO_SECRET__: The secret string that is set in RH-SSO for the client for Business Central. - `49dfc82f-301a-4f42-96bc-1da278474302`

  * __KIE_SERVER_SSO_CLIENT__: The RH-SSO client name for Process Server. - `process-server`

  * __KIE_SERVER_SSO_SECRET__: The secret string that is set in RH-SSO for the client for Process Server. - `8d33123a-a54c-4adf-b64f-d2b055974c11`


### 1.2.3.6 Enabling Prometheus metric collection for an authoring environment

If you want to configure your Process Server deployment to use Prometheus to collect and store metrics, enable support for this feature in Process Server at deployment time.

- __PROMETHEUS_SERVER_EXT_DISABLED__ parameter to `false`


### 1.2.4 Completing deployment of the template for an authoring environment

- Using the Openshift Console, go to the rhpam-authoring Project and click `Browse Catalog`, select the `Red Hat Process Automation Manager 7.4 authoring environment (HA, persistent, with https)`

- Fill the parameters defined in the previous sections and create!!

Wait for all pods to start...

- Go back to SSO and reconfigure the urls in the clients

```
Example:
Root URL: https://rhpam-rhpamcentr-rhpam-authoring.apps.8aa4.example.opentlc.com
Valid Redirect URIs: https://rhpam-rhpamcentr-rhpam-authoring.apps.8aa4.example.opentlc.com/*
```

{% include elements/figure.html image="/assets/images/pam_sso/client_reconfigure.png" caption="Reconfigure the business-central and process-server clients" %}


# REFERENCES
{% capture list_items %}
Google,https://www.google.com
GitHub,https://www.github.com
{% endcapture %}
{% include elements/list.html title="Websites" %}

