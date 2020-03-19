---
title: Secure Fuse Applications with RH SSO
tags: [Fuse, SSO, Red Hat, Spring Boot, Camel, Keycloak]
style: 
color: 
description: SSO can be a very useful security central point for microservices architectures
---

In this article, we’ll cover microservice security concepts by using protocols such as OpenID Connect with the support of Red Hat Single Sign-On. Working with a microservice-based architecture, user identity, and access control in a distributed, in-depth form must be carefully designed. Here, the integration of these tools will be detailed, step-by-step, in a realistic view.

We will use Red Hat SSO standalone which will help in securing a fuse-springboot microservice.

## 1. Getting started with standalone SSO 

### 1.1 Install Standalone SSO

- Download [RH SSO](https://access.redhat.com/jbossnetwork/restricted/listSoftware.html?downloadType=distributions&product=core.service.rhsso) from Red Hat Customer Portal

- Unzip

- Boot the server - go to the bin directory of the unzipped folder

```
$ cd bin
$ ./standalone.sh
```

- Creating the Admin Account - as soon as the server boots: 
  - open [http://localhost:8080/auth](http://localhost:8080/auth)
  - enter a username and password to create an initial admin user

{% include elements/figure.html image="/assets/images/sso/admin_registration.png" caption="Admin Creation" %}

- Logging in to the Admin Console: 
[http://localhost:8080/auth/admin/](http://localhost:8080/auth/admin/)

Now SSO is ready to be configured.

---

### 1.2 Configure SSO

We are going to create a new realm and a few clients for our microservice to be able to connect to SSO.

- Login to the Admin Console with admin/admin : 
[http://localhost:8080/auth/admin/](http://localhost:8080/auth/admin/)

- Add a new Realm. Hoover over the Master Realm and click `Add realm`. Lets call it `development`

{% include elements/figure.html image="/assets/images/sso/add_realm.png" caption="Add new realm" %}

- Create a client for our Fuse Spring microservice. 

Go to the newly created `development` realm, click on clients to add a new one with

```
Client ID       : fuse-springboot-rest
Client Protocol : openid-connect
```

- Configure the client

SSO is running port 8080, make sure your microservice runs on another port. In this example, the api is configured to run on 8888. (cofigured in application.properties - server.port=8888)

```
Access Type         : confidential
Valid Redirect URIs : http://localhost:8888
# Required for micro-service to micro-service secured calls
Service Accounts Enabled : On
Authorization Enabled : On
```

```
Note: Access Type confidential supports getting access token using client credentials grant as well as authorization code grant. If a micro-service needs to call another micro-service, caller will be ‘confidential’ and callee will be ‘bearer-only’.
```

{% include elements/figure.html image="/assets/images/sso/configure_client.png" caption="Configure client" %}

- Create Client Role

Create a Role within the client called `user`

{% include elements/figure.html image="/assets/images/sso/add_client_role.png" caption="Add role within client" %}


- Create User

{% include elements/figure.html image="/assets/images/sso/new_user.png" caption="Create new user" %}


- Assign password to user

We need to give a password to our created user, lets go ahead and give it `john` -> Super Safe!!

{% include elements/figure.html image="/assets/images/sso/user_password.png" caption="User Role" %}

```
NOTE:

If you want the user to change the password on the first login, select `Temporary`
```


- Associate Client Role to User

This is to provide access to the client (our fuse microservice). The Role within the Client needs to be assigned to the User.

{% include elements/figure.html image="/assets/images/sso/user_role.png" caption="User Role" %}


- Get Configuration From OpenID Configuration Endpoint

SSO is OpenID Connect and OAuth2 compliant, below is OpenID Connection configuration URL to get details about all security endpoints:

```
GET http://127.0.0.1:8080/auth/realms/development/.well-known/openid-configuration
```

Important URLS to be copied from response:

```
issuer : http://127.0.0.1:8080/auth/realms/development
authorization_endpoint: ${issuer}/protocol/openid-connect/auth
token_endpoint: ${issuer}/protocol/openid-connect/token

token_introspection_endpoint: ${issuer}/protocol/openid-connect/token/introspect
userinfo_endpoint: ${issuer}/protocol/openid-connect/userinfo
Response also contains grant types and scopes supported
grant_types_supported: ["client_credentials", …]
scopes_supported: ["openid", …]
```

- Get Access Token Using Postman (For Testing)

An SSO access token is a [JWT](https://jwt.io/). Each field in that JSON is called a claim. By default, logged in username is returned in a claim named “preferred_username” in the access token. Later we configure our Spring Boot application with the key-value keycloak.principal-attribute=preferred_username in the application.properties

__Steps__

Go to Postman and create a new request.

Select the Authorization tab and select type `OAuth 2.0` -> `Get New Access Token`

Select Authorization Type as OAuth 2.0, click on ‘Get New Access Token’ and enter following details.

{% include elements/figure.html image="/assets/images/sso/postman.png" caption="Request Token from Postman" %}

If you click `Request Token` it will redirect you to the SSO realm login page. Enter `John/John`

{% include elements/figure.html image="/assets/images/sso/postman_sso.png" caption="Post Redirected to SSO" %}

A window appears with the Token to be used by the client. Save it!! This is a temporary token, so each time you want to make a request, you have to request a new one.

{% include elements/figure.html image="/assets/images/sso/user_access_token.png" caption="Client Access Token" %}

```
NOTES:

- Make sure you select client authentication as “Send client credentials in body” while requesting token.

- Callback URL is redirect URL configured in SSO.

- Client secret may be different for you, copy the one from client configuration in SSO.

- You may also use https://jwt.io to inspect the contents of token received.
```

Lets go to [jwt](https://jwt.io) and validate our token. Just copy the token and paste it in the jwt Debugger.

{% include elements/figure.html image="/assets/images/sso/token_validation.png" caption="Token Validation" %}

__Looks Good!__

## 2. Fuse Spring Boot Microservice

Lets create a simple Microservice with a REST interface, running on Spring Boot Standalone.

### 2.1 Pre-Requisites

You should have Code Ready Studio and Fuse tooling installed. Not mandatory, but it makes things easier.
([If not, then follow this](https://developers.redhat.com/blog/2019/05/29/how-to-set-up-red-hat-codeready-studio-12-integration-tooling))

---
### 2.2 Steps to create the microservice

- On Code Ready Studio create a new `Fuse Integration Project`. I am using Fuse version `7.5.0.fuse-750029-redhat-00002 (Fuse 7.5.0 GA)`

- We are going to use Camel REST Component to implement our REST API. In your Maven POM, add camel-servlet-starter as a dependency. This will allow Camel to deploy our service into the embedded Tomcat container:

```xml
<dependency>
  <groupId>org.apache.camel</groupId>
  <artifactId>camel-servlet-starter</artifactId>
</dependency>
```

- Add camel-jackson-starter as a dependency. This will allow Camel to marshal to/from JSON:

```xml
<dependency>
  <groupId>org.apache.camel</groupId>
  <artifactId>camel-jackson-starter</artifactId>
</dependency>
```

- add a restConfiguration element to your camel context

`JAVA DSL `

In the RouteBuilder of your new project, add a restConfiguration() element.

This initialises the component that will provide the REST service. In this example we’re using the Servlet component for this.

We do this first before defining any operations for the service:

```java
public void configure() throws Exception {
    restConfiguration()
            .component("servlet")
            .bindingMode(RestBindingMode.auto);
}
```

`XML DSL`

Insert this inside the Camel Context xml element

```xml
<restConfiguration bindingMode="auto" component="servlet"/>
```

- Define a REST endpoint, and add skeletons for each of your operations - GET, POST, etc.

First, define a REST endpoint, using rest, with the path (URI) of your service. Then append each of your operations - these are your HTTP verbs.

`JAVA DSL`
```java
rest("/api/hello")
    .get().route().to("...")
    .post().route().to("...")
    .delete().route().to("...");
```

`XML DSL`
```xml
<rest path="/api/hello">
    <get>
        <route>
            <to uri="..."/>
        </route>
    </get>
    <post>
        <route>
            <to uri="..."/>
        </route>
    </post>
</rest>
```

- Now build your Camel routes for your REST operations! Here’s an example of a REST service that’s been filled out:

`JAVA DSL`

```java
    @Override
    public void configure() throws Exception {

        restConfiguration()
        .component("servlet")
        .bindingMode(RestBindingMode.auto);
        
        rest("/hello")
        .get().route().bean(HelloBean.class , "sayHello");
        //("bean:HelloBean?method=sayHello");
    }
    
    @Bean(name = "helloWorld")
    public HelloBean helloBean() {
        return new HelloBean();
    }
```

```xml
<bean id="helloBean" class="org.mycompany.HelloBean"/>
<rest path="/hello">
    <get>
        <route>
            <bean ref="helloBean" method="hello"/>
        </route>
    </get>
</rest>
```

- Create a simple Bean to handle your api operations

```java
@Component
public class HelloBean {
    /*
     * Add a method to say Hello
     * Use as input the parameter the body received and concatenate it with "Hello World ! " text
     */
	public String sayHello(Exchange exchange) {
		String body = (String)exchange.getIn().getBody();
		return "Hello world ! " + body;
	}
}
```

- Add servlet mapping to the application.properties (/src/main/resources) - by default, Camel maps incoming requests to /camel/*

```
camel.component.servlet.mapping.context-path=/api/*
```

- Run and test

Right Click on your project -> Run as -> Maven build ... -> Goals: spring-boot:run

Test

```sh
curl -X GET http://localhost:8888/api/hello

"Hello world !
```

`That’s it! Now keep building up your REST API with as many endpoints and operations as you need!!`

#### 2.2.1 Extra Steps

- To add support for automatically converting requests and responses to/from JSON, define POJOs representing your request and response messages, e.g.:

```java
public class CreateCustomerResponse {
    private String result;
    // Add some getters and setters here...
}
```

Now, declare your request and response types, so that Camel knows how to marshal them to/from JSON:

```java
rest()
    .post()
        .route()
        .type(Customer.class)
        .outType(CreateCustomerResponse.class)
        .to("direct:post");
```

And also ensure that you’ve set binding mode in the REST configuration:

```java
restConfiguration()
    .component("servlet")
    .bindingMode(RestBindingMode.auto);
```

---

### 2.3 Configure the microservice to be secured with SSO

The Keycloak Spring Boot adapter takes advantage of Spring Boot’s autoconfiguration so all you need to do is add the Keycloak Spring Boot starter to your project

Currently the following embedded containers are supported and do not require any extra dependencies if using the Starter:

```
Tomcat
Undertow
Jetty
```

- Dependencies

Add the Adapter BOM dependency

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.keycloak.bom</groupId>
      <artifactId>keycloak-adapter-bom</artifactId>
      <version>3.4.17.Final-redhat-00001</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

Add the Keycloak/SSO Spring dependency

```xml
<dependency>
    <groupId>org.keycloak</groupId>
    <artifactId>keycloak-spring-boot-starter</artifactId>
</dependency>
```

- Keycloak Spring Boot Adapter Configuration

Using Spring Boot Configuration (application.properties)

```
keycloak.realm = development
keycloak.principal-attribute=preferred_username
keycloak.auth-server-url = http://127.0.0.1:8080/auth
keycloak.ssl-required = external
keycloak.resource = fuse-springboot-rest
keycloak.credentials.secret = 1ff63f52-9a6d-4da4-8d38-0eaafd0fd860
keycloak.use-resource-role-mappings = true
```

If you don't remember all the details, go to SSO with the admin account, select our `fuse-springboot-rest client` -> Installation -> Format option=`Keycloak OIDC JSON` and you get the necessary details such as secret

{% include elements/figure.html image="/assets/images/sso/sso_client_installation.png" caption="Client Installation" %}

```
NOTE:
You can disable the Keycloak Spring Boot Adapter (for example in tests) by setting keycloak.enabled = false
```

- Specify JavaEE security configuration

You also need to specify the Java EE Security Constraints that would normally go in the `web.xml`. In this case we can simply use our Spring Configuration.

The Spring Boot Adapter will set the login-method to KEYCLOAK and configure the security-constraints at startup time. 

Here’s an example configuration:

```
keycloak.securityConstraints[0].authRoles[0]=user
keycloak.securityConstraints[0].securityCollections[0].patterns[0]=/*
```
With this configurations we are defining that every request to all our api endpoints should be done with an authenticated user. That user should have a role `user`.

- Final Spring Configuration

{% include elements/figure.html image="/assets/images/sso/spring_config.png" caption="Spring Configuration" %}

- Test

Lets start our application!!

```
mvn clean spring-boot:run
```

Go back to Postman, remember we already have our Access Token. If it has expired, then request a new one.

{% include elements/figure.html image="/assets/images/sso/postman_access_token.png" caption="Access Token" %}

Send a request to your microservice api endpoint and analyze the request headers. Note the Authorization Bearer, which is our token.

{% include elements/figure.html image="/assets/images/sso/request_header.png" caption="Request Headers" %}

__DONE!__

Feel free to look at the microservice [repo](https://github.com/seravat/sso-fuse-springboot-rest)

## References

[Quickstart Project](https://github.com/aelkz/ocp-sso/blob/master/README.pt-br.md)

[Red Hat SSO Quickstarts](https://github.com/redhat-developer/redhat-sso-quickstarts#keycloak)

[SSO Installation Guide](https://access.redhat.com/documentation/en-us/red_hat_single_sign-on/7.3/html/getting_started_guide/install-boot)

[SSO Configuration](https://medium.com/@bcarunmail/securing-rest-api-using-keycloak-and-spring-oauth2-6ddf3a1efcc2)

[Tom Donohue's Blog](https://tomd.xyz/camel-rest/)

[Red Hat Documentation](https://access.redhat.com/documentation/en-us/red_hat_single_sign-on/7.2/html-single/securing_applications_and_services_guide/index#spring_boot_adapter)

[Baeldung](https://www.baeldung.com/spring-boot-keycloak)

[Secure your Spring Boot Application](https://www.keycloak.org/2017/05/easily-secure-your-spring-boot.html)

[Secure Spring Boot with Keycloak](https://medium.com/keycloak/secure-spring-boot-2-using-keycloak-f755bc255b68)










