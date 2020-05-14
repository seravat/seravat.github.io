---
title: Deploy and Configure Nexus as registry proxy on Openshift 3.11
tags: [Red Hat, Openshift, Nexus]
style: 
color: 
---

In this example we deploy a Nexus 3.x instance to Openshift 3.11 with persistence to serve as a registry proxy to the official [Red Hat Registry](registry.redhat.io)

## 1. Deploy Nexus on Openshift

Lets use a persistent nexus template forked from [Openshift Demos](https://github.com/OpenShiftDemos/nexus)

- Import the template to Openshift. Usually templates exist in the openshift namespace

```
oc create -f https://raw.githubusercontent.com/OpenShiftDemos/nexus/master/nexus3-persistent-template.yaml -n openshift
```

- Create a new project to host our Nexus

```
oc new-project nexus
```

- Create a new Nexus application using the imported template. Lets also modify the Nexus version and the persistent volume claim size.

> NOTE: We are assuming your cluster already has provisioned PVs or can dynamically provision PVs

```
oc new-app nexus3-persistent -p NEXUS_VERSION=3.21.2 -p VOLUME_CAPACITY=5Gi
```

- Wait for the pod to start and verify Nexus is running properly - Login using admin/admin123

```
oc get routes

...

NAME      HOST/PORT                                   PATH      SERVICES   PORT       TERMINATION   WILDCARD
nexus     nexus-nexus.apps.xxxx.example.opentlc.com             nexus      8081-tcp                 None
```

- Login using the route, it will say that the admin password is located in /nexus-data/admin.password - ssh into your pod and retrieve the admin password. Follow the wizard to setup a new admin password.

{% include elements/figure.html image="/assets/images/nexus/login.png" caption="Nexus first login" %}

## 2. Configure as redhat registry proxy

### 2.1 Create a new Blob Store on Nexus to keep our images

{% include elements/figure.html image="/assets/images/nexus/blob_store.png" caption="Create new blob store" %}


### 2.2 Create a new proxy repository

- Select the name, configure and http connector on port 5000, allow Docker v1 API and add the  [Red Hat registry](https://registry.redhat.io)  

{% include elements/figure.html image="/assets/images/nexus/proxy_repo.png" caption="Create new proxy repository" %}

- Add the certificate to Nexus truststore - click `View Certificate` and add to the truststore on the windows that opens

{% include elements/figure.html image="/assets/images/nexus/proxy_repo1.png" caption="Create new proxy repository" %}

- Storage - select the blob store we created before

{% include elements/figure.html image="/assets/images/nexus/proxy_repo2.png" caption="Create new proxy repository" %}


### 2.3 Expose the registry port

Now we want to be able to access our proxy registry from outside Openshift.
Remember we configured an http connector on our proxy registry on port 5000 of our Nexus instance.

We had to expose the port `5000` on our container - Lets edit the DeploymentConfig of our nexus. This can be done through the console, using `oc edit` or `oc patch`

```
  oc patch dc nexus -p '{"spec":{"template":{"spec":{"containers":[{"name":"nexus","ports":[{"containerPort": 5000,"protocol":"TCP","name":"docker"}]}]}}}}'
```

A new pod will be provisioned due to the DeploymentConfig change.

After exposing the port on the container we need to add that port to our service

```
oc expose dc nexus --name=nexus-registry --port=5000
```

### 2.4 Create a secure Route to the Registry (TLS)

We can expose an insecure route and add that exception to the configuration of our local docker daemon, however it is not the best way to go about it. We are going to expose a secure Route 

We’ll encrypt traffic using OpenShift’s edge SSL termination. When this option is chosen, the service will be exposed externally on port 443. A very basic way of doing this is using:

```
oc create route edge nexus-registry --service=nexus-registry
```

`HOWEVER!` If you’re following this guide using minishift, or another local demo OpenShift cluster then you’re probably going to need to create a certificate for this service - normally Openshift production servers have root CA's created by the organization. So follow the steps below to create a root certificate authority (CA), a certificate for your service, and then configure Docker to trust your cert.

#### 2.4.1 Create root certificate authority (CA) using openssl

```
NEXUS_HOSTNAME=nexus-registry-nexus.apps.8aa4.example.opentlc.com
openssl genrsa -des3 -out rootCA.key 4096

cat << EOF > openssl.conf
[ req ]
distinguished_name=dn
[ dn ]
[ root_ca ]
basicConstraints = critical, CA:true
EOF

openssl req -x509 -config openssl.conf \
-extensions root_ca -new -nodes \
-key rootCA.key -sha256 -days 1024 \
-subj "/CN=EggRootCertificate" \
-out rootCA.crt
```

#### 2.4.2 Create a certificate signing request (CSR) for the Nexus registry route and sign it with the root CA

```
openssl genrsa -out ${NEXUS_HOSTNAME}.key 2048

openssl req -new -sha256 \
-key ${NEXUS_HOSTNAME}.key \
-subj "/C=GB/ST=London/L=London/O=EggCorp/CN=${NEXUS_HOSTNAME}" \
-out ${NEXUS_HOSTNAME}.csr

openssl x509 -req -in ${NEXUS_HOSTNAME}.csr \
-CA rootCA.crt -CAkey rootCA.key \
-CAcreateserial -out ${NEXUS_HOSTNAME}.crt \
-days 500 -sha256
```

#### 2.4.3 Create the Route, using the certificate, key and CA certificate

```
oc create route edge nexus-registry --service=nexus-registry \
--hostname=${NEXUS_HOSTNAME} \
--key=${NEXUS_HOSTNAME}.key \
--cert=${NEXUS_HOSTNAME}.crt \
--ca-cert=rootCA.crt
```

### 2.5 Configure Docker to trust the certificate

In this case I am using a Mac and need to add it to the keychain:

```
sudo security add-trusted-cert -d \
    -r trustRoot \
    -k /Library/Keychains/System.keychain \
    rootCA.crt
```

Restart your Docker Daemon
If you’re still having issues with “certificate signed by unknown authority” then try restarting your Mac entirely (fixed it for me).

#### 2.5.1 Test the connection to our Nexus Docker Registry

```
oc get route | grep nexus-registry

...
nexus-registry-nexus.apps.8aa4.example.opentlc.com

docker login nexus-registry-nexus.apps.8aa4.example.opentlc.com

```

### 2.6 Allow Openshift to pull images from the Docker Proxy Registry

We need to add a new configuration to allow Openshift to pull images through our repository. Also, just for this test we are going to create a new https route without certificates that allows insecure traffic.

### 2.6.1 Create unsecured route

```
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  labels:
    app: nexus
    template: nexus3-persistent-template
  name: nexus-registry
  namespace: nexus
spec:
  host: nexus-registry-nexus.apps.8aa4.example.opentlc.com
  tls:
    insecureEdgeTerminationPolicy: Allow
    termination: edge
  to:
    kind: Service
    name: nexus-registry
    weight: 100
  wildcardPolicy: None
```

### 2.6.2 Configure Openshift to allow this new registry

To add our new registry as accepted on Openshift, lets edit the Openshift Master configuration file `/etc/origin/master-config.yaml`.

Find the section `imagePolicyConfig`

{% include elements/figure.html image="/assets/images/nexus/imagepolicyconfig.png" caption="Openshift master-config.yaml" %}

Add a new entry, which should be our secure Route created before:

```
- domainName: '*.apps.8aa4.example.opentlc.com'
  insecure: true
```

This entry means that Openshift allows image-pull from using that hostname.

Restart your master(s)

### 2.6.3 Create a new secret with the repository credentials OR allow anonymous pullers from Nexus

`METHOD 1 - Create pull secret`

If you can't pull images from the registry, add a new secret and link it to the default service account


```
oc create secret docker-registry rh-proxy  \
--docker-server=http://nexus-registry-nexus.apps.8aa4.example.opentlc.com  \
--docker-username=admin     \
--docker-password=admin123

oc secrets link default rh-proxy --for=pull
```

`METHOD 2 - Allow anonymous pulls from Nexus`

Go into our registry configuration in Nexus and allow anonymous pulls

{% include elements/figure.html image="/assets/images/nexus/anonymous_pull.png" caption="Allow anonymous pull from the registry" %}

Activate the Docker Bearer Token realm in Nexus security tab

{% include elements/figure.html image="/assets/images/nexus/anonymous_pull1.png" caption="Docker Bearer Token realm" %}


> `NOTE 1:` 

If we wanted to use the secure route to our repo, we would need to [add the route certificates to the Openshift chain](https://docs.openshift.com/container-platform/3.11/install_config/certificate_customization.html)

> `NOTE 2:`

We tried to configure an https connector on our Nexus repository exposed on port 443. Also exposed that port on the container and created a service.


{% include elements/figure.html image="/assets/images/nexus/nexus_https.png" caption="Configure https connector in Nexus" %}

```
oc patch dc nexus -p '{"spec":{"template":{"spec":{"containers":[{"name":"nexus","ports":[{"containerPort": 443,"protocol":"TCP","name":"docker-ssl"}]}]}}}}'

oc expose dc nexus --name=nexus-registry-ssl --port=443
```

We were unable to use this to pull to Openshift - adding also the domain to the master-config.yaml


## 3. References

https://help.sonatype.com/repomanager3/security/realms

https://medium.com/flowfactor/creating-a-private-external-image-registry-for-openshift-4-3-a67fc27cd5a4


https://tomd.xyz/openshift-nexus-docker-registry/

https://community.sonatype.com/t/to-proxy-openshift-registry-with-nexus-oss-registry-proxy/3021

https://help.sonatype.com/repomanager3/formats/docker-registry/proxy-repository-for-docker

https://blog.sonatype.com/using-nexus-3-as-your-repository-part-3-docker-images

https://community.sonatype.com/t/registry-connect-redhat-com-registries-proxy-in-nexus/3718

https://issues.sonatype.org/browse/NEXUS-16718

https://access.redhat.com/solutions/4616891

https://qiita.com/leechungkyu/items/86cad0396cf95b3b6973

https://www.ivankrizsan.se/2016/06/09/create-a-private-docker-registry/

https://blog.sonatype.com/using-nexus-3-as-your-repository-part-3-docker-images

https://mtijhof.wordpress.com/2018/07/23/using-nexus-oss-as-a-proxy-cache-for-docker-images/

https://access.redhat.com/solutions/3654811

https://blog.sonatype.com/using-nexus-3-as-your-repository-part-3-docker-images



https://access.redhat.com/documentation/en-us/openshift_container_platform/3.11/html/developer_guide/tutorials#nexus-maven-tutorial

https://github.com/seravat/openshift-nexus

