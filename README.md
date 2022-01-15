# Full-Stack-Java-with-React-Spring-Boot-and-JHipster
This tutorial teaches how to create a slick-looking, full-stack, secure application using React, Spring Boot, and JHipster.

## OVERVIEW
Being a full-stack developer can be exciting because you can create the backend and frontend of an app. There is business logic and algorithms as well as styling, and securing everything. It also pays well. Today, I'm going to show you how you can be a full-stack Java developer with Spring Boot, React, and JHipster.

**Prerequisites**:

Node.js 14+
Java 11+
Docker Compose
If you're on Windows, you may need to install the Windows Subsystem for Linux for some commands to work.

I recommend using SDKMAN to manage your OpenJDK installations. Just run ```sdk install java 11.0.2-open``` to install Java 11 and ```sdk install java 17-open``` for Java 17.

This tutorial won't provide the nitty-gritty details on how to write code in Java, React, or Spring Boot. That's because JHipster will write most of the code for you! 

# Full Stack Development with React and Spring Boot #

One of the easiest ways to get started with React is by using Create React App (CRA). You install it locally, then run ```create-react-app <project>``` to generate a React application with minimal dependencies. It uses webpack under-the-covers to build the project, launch a web server, and run its tests.

Spring Boot has a similar tool called Spring Initializr. Spring Initializer is a bit different than CRA because it's a website (and API) that you can create applications with.

Today, I'll show you how to build a Flickr clone with React and Spring Boot. However, I'm going to cheat. Rather than building everything using the aforementioned tools, I'm going to use JHipster. JHipster is an application generator that initially only supported Angular and Spring Boot. Now it supports Angular, React, and Vue for the frontend. JHipster also has support for Kotlin, Micronaut, Quarkus, .NET, and Node.js on the backend.

In this tutorial, we'll use React since it seems to be the most popular frontend framework today.

## Get Started with JHipster 7 ##

If you haven't heard of JHipster, I have a special treat for you! JHipster started as a Yeoman application generator back in 2013 and has grown to become a development platform. It allows you to quickly generate, develop, and deploy modern web apps and microservice architectures. Today, I'll show you how to build a Flickr clone with JHipster and lock it down with OAuth and OpenID Connect (OIDC).

To get started with JHipster, you'll need a fast internet connection and Node.js installed. The project recommends you use the latest LTS (Long Term Support) version, which is 14.7.6 at the time of this writing. To run the app, you'll need to have Java 11 installed. If you have Git installed, JHipster will auto-commit your project after creating it. This will allow you to upgrade between versions.

Run the following command to install JHipster:

```
npm i -g generator-jhipster@7
```

To create a full-stack app with JHipster, create a directory, and run jhipster in it:

```
mkdir full-stack-java
cd full-stack-java
jhipster
```

JHipster will prompt you for the type of application to create and what technologies you want to include. For this tutorial, make the following choices:



| Question  | Answer |
| ------------- | ------------- |
| Type of application?l  | Monolithic application  |
| Name?  | flickr2  |
| Spring WebFlux? | No |
| Java package name? | com.auth0.flickr2 |
| Type of authentication? | OAuth 2.0 / OIDC |
| Type of database? | SQL |
| Production database? | PostgreSQL |
| Development database? | H2 with disk-based persistence |
| Which cache? |	Ehcache |
| Use Hibernate 2nd level cache? |	Yes |
| Maven or Gradle? |	Maven |
| Use the JHipster Registry? |	No |
| Other technologies? |	<blank> |
| Client framework? |	React |
| Admin UI? |	Yes |
| Bootswatch theme? |	United > Dark |
| Enable i18n? |	Yes |
| Native language of application? |	English |
| Additional languages? |	Your choice! |
| Additional testing frameworks? |	Cypress |
| Install other generators? |	No |
  
Press Enter, and JHipster will create your app in the current directory and run ```npm install``` to install all the dependencies specified in package.json.

**Verify Everything Works with Cypress and Keycloak**

 
  
When you choose OAuth 2.0 and OIDC for authentication, the users are stored outside of the application rather than in it. 
  - You need to configure an identity provider (IdP) to store your users and allow your app to retrieve information about them. 
  - By default, JHipster ships with a Keycloak file for Docker Compose.
  - A default set of users and groups is imported at startup, and it has a client registered for your JHipster app.

Here's what the ```keycloak.yml``` looks like in your app's ```src/main/docker directory```:
  
```
# This configuration is intended for development purpose; it's **your** responsibility
# to harden it for production
version: '3.8'
services:
  keycloak:
    image: jboss/keycloak:15.0.2
    command:
      [
          '-b',
          '0.0.0.0',
          '-Dkeycloak.migration.action=import',
          '-Dkeycloak.migration.provider=dir',
          '-Dkeycloak.migration.dir=/opt/jboss/keycloak/realm-config',
          '-Dkeycloak.migration.strategy=OVERWRITE_EXISTING',
          '-Djboss.socket.binding.port-offset=1000',
          '-Dkeycloak.profile.feature.upload_scripts=enabled',
      ]
    volumes:
      - ./realm-config:/opt/jboss/keycloak/realm-config
    environment:
      - KEYCLOAK_USER=admin
      - KEYCLOAK_PASSWORD=admin
      - DB_VENDOR=h2
    # If you want to expose these ports outside your dev PC,
    # remove the "127.0.0.1:" prefix
    ports:
      - 127.0.0.1:9080:9080
      - 127.0.0.1:9443:9443
      - 127.0.0.1:10990:10990
  ```

Start Keycloak with the following command in your project's root directory.

```
  docker-compose -f src/main/docker/keycloak.yml up -d
```

Verify everything works by starting your app with Maven:

```
  ./mvnw
```
  
Open another terminal to run your new app's Cypress tests:

```
  npm run e2e
```

You should see output like the following:

```

  (Run Finished)

       Spec                                              Tests  Passing  Failing  Pending  Skipped
  ┌────────────────────────────────────────────────────────────────────────────────────────────────┐
  │ ✔  administration/administration.spec.      00:12        5        5        -        -        - │
  │    ts                                                                                          │
  └────────────────────────────────────────────────────────────────────────────────────────────────┘
    ✔  All specs passed!                        00:12        5        5        -        -        -

```
  
**Change Your Identity Provider to Auth0**
  
JHipster uses Spring Security's OAuth 2.0 and OIDC support to configure which IdP it uses. When using Spring Security with Spring Boot, you can configure most settings in a properties file. You can even override properties with environment variables.

To switch from Keycloak to Auth0, you only need to override the default properties (for Spring Security OAuth). You don't even need to write any code!

To see how it works, create a ```.auth0.env``` file in the root of your project, and code it like below to override the default OIDC settings:

```
export SPRING_SECURITY_OAUTH2_CLIENT_PROVIDER_OIDC_ISSUER_URI=https://<your-auth0-domain>/
export SPRING_SECURITY_OAUTH2_CLIENT_REGISTRATION_OIDC_CLIENT_ID=<your-client-id>
export SPRING_SECURITY_OAUTH2_CLIENT_REGISTRATION_OIDC_CLIENT_SECRET=<your-client-secret>
export JHIPSTER_SECURITY_OAUTH2_AUDIENCE=https://<your-auth0-domain>/api/v2/
```
  
⚠️ WARNING: Modify your existing ```.gitignore``` file to have ```*.env``` so you don't accidentally check in your secrets for public view!

You'll need to create a new web application in Auth0 and fill in the ```<...>``` placeholders before this works.

**Create an OpenID Connect App on Auth0**
  
Log in to your Auth0 account (or sign up if you don't have an account). You should have a unique domain like ```dev-xxx.eu.auth0.com```.

Press the "**Create Application**" button in "Applications" section. Use a name like ```JHipster Baby!```, select ```Regular Web Applications```, and click **"Create"**.

Switch to the **Settings** tab and configure your application settings:

  - Allowed Callback URLs: ```http://localhost:8080/login/oauth2/code/oidc```
  - Allowed Logout URLs: ```http://localhost:8080/```

Scroll to the bottom and click **Save Changes**.

In the **roles** section, create new roles named ```ROLE_ADMIN``` and ```ROLE_USER```.

Create a new user account in the **users** section. Click on the **Role** tab to assign the roles you just created to the new account.

Make sure your new user's email is verified before attempting to log in!

Next, head to **Auth Pipeline > Rules > Create**. Select the ```Empty rule``` template. Provide a meaningful name like ```Group claims``` and replace the Script content with the following.

```
function(user, context, callback) {
  user.preferred_username = user.email;
  const roles = (context.authorization || {}).roles;

  function prepareCustomClaimKey(claim) {
    return `https://www.jhipster.tech/${claim}`;
  }

  const rolesClaim = prepareCustomClaimKey('roles');

  if (context.idToken) {
    context.idToken[rolesClaim] = roles;
  }

  if (context.accessToken) {
    context.accessToken[rolesClaim] = roles;
  }

  callback(null, user, context);
}
```
  
This code is adding the user's roles to a custom claim (prefixed with ```https://www.jhipster.tech/roles```). This claim is mapped to Spring Security authorities in ```SecurityUtils.java```.

```
public static List<GrantedAuthority> extractAuthorityFromClaims(Map<String, Object> claims) {
    return mapRolesToGrantedAuthorities(getRolesFromClaims(claims));
}

@SuppressWarnings("unchecked")
private static Collection<String> getRolesFromClaims(Map<String, Object> claims) {
    return (Collection<String>) claims.getOrDefault(
        "groups",
        claims.getOrDefault("roles", claims.getOrDefault(CLAIMS_NAMESPACE + "roles", new ArrayList<>()))
    );
}

private static List<GrantedAuthority> mapRolesToGrantedAuthorities(Collection<String> roles) {
    return roles.stream().filter(role -> role.startsWith("ROLE_")).map(SimpleGrantedAuthority::new).collect(Collectors.toList());
}
```
  
The ```SecurityConfiguration.java``` class has a bean which calls this method to configure a user's roles from their OIDC data.

```
@Bean
public GrantedAuthoritiesMapper userAuthoritiesMapper() {
    return authorities -> {
        Set<GrantedAuthority> mappedAuthorities = new HashSet<>();

        authorities.forEach(authority -> {
            // Check for OidcUserAuthority because Spring Security 5.2 returns
            // each scope as a GrantedAuthority, which we don't care about.
            if (authority instanceof OidcUserAuthority) {
                OidcUserAuthority oidcUserAuthority = (OidcUserAuthority) authority;
                mappedAuthorities.addAll(SecurityUtils.extractAuthorityFromClaims(oidcUserAuthority.getUserInfo().getClaims()));
            }
        });
        return mappedAuthorities;
    };
}
```

Click **Save changes** to continue.


**Run Your JHipster App with Auth0**

Stop your JHipster app using ```Ctrl+C```, set your Auth0 properties in ```.auth0.env```, and start your app again.

```
source .auth0.env
./mvnw
```


**Boom**! - your full-stack app is now using Auth0! Open your favorite browser to ```http://localhost:8080```.
  
![](https://github.com/DrVicki/Full-Stack-Java-with-React-Spring-Boot-and-JHipster/blob/main/hipster-images/04_jhipster-homepage.png) 
  
  
You should see your app's homepage with a link to sign in. Click **sign in**, and you'll be redirected to Auth0 to log in.
  



