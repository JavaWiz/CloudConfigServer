### Overview
Spring Cloud Config is Spring's client/server approach for storing and serving distributed configurations across multiple applications and environments.

This configuration store is ideally versioned under Git version control and can be modified at application runtime. While it fits very well in Spring applications using all the supported configuration file formats together with constructs like Environment, PropertySource or @Value, it can be used in any environment running any programming language.
### Spring Cloud Config Server
Spring Cloud Config Server provides an HTTP resource-based API for external configuration (name-value pairs or equivalent YAML content). The server is embeddable in a Spring Boot application, by using the `@EnableConfigServer` annotation. Consequently, the following application is a config server:
```
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.config.server.EnableConfigServer;

@EnableConfigServer
@SpringBootApplication
public class SbConfigServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(SbConfigServerApplication.class, args);
	}
}
```
Like all Spring Boot applications, it runs on port 8080 by default, but we can switch it to the more conventional port `8888` in various ways. The easiest, which also sets a default configuration repository, is by launching it with `spring.config.name=configserver` (there is a `configserver.yml` in the Config Server jar). Another is to use your own `application.properties`, as shown in the following example:
**application.yml**
```
spring.application.name: sb-config-server
server.port: 8888
spring:
  cloud:
    config:
      server:
        git:
################## Local repo config ####################
          uri: file:${user.home}\config-repo
          searchPaths: config-server-client
          default-label: master
################## Remote repo config ####################
#          uri: https://github.com/JavaWiz/config-server
#          username: de2bc7cc571df3bc7a81074f8ad9a01eaf2aa03c
#          password:
```
where `${user.home}/config-repo` is a local git repository location containing YAML and properties configuration files.

Now all basic configurations are done. Let's run the application as SpringBootApp. The Tomcat server will start at 8888 as configured. Check the following URL (This will pick the default configuration file from the repository).
```
http://localhost:8888/config-server-client/default
```
### What is the above url? 
It is `http://localhost:8888/{config-file-name-without-extention}/{prifile}`

If below response is received, then all configurations are OK, i.e., Config Server is fetching default property from Local Git repository.
```
{
    "name": "config-server-client",
    "profiles": [
    "default"
    ],
    "label": null,
    "version": "bc73563231eeb3882caa497b5cb0edb5fc45fc8b",
    "state": null,
    "propertySources": [
        {
        "name": "file:C:\\contents\\workspace\\github\\config-server/file:C:\\contents\\workspace\\github\\config-server\\config-server-client.properties",
            "source": {
            "message": "Hello world - this is from config server"
            }
        }
    ]
}
```
To access the development environment file, we can use the following URL:  `http://localhost:8888/config-server-client/dev`.

### The Client Implementation
Next, let's take care of the client. This will be a very simple client application, consisting of a REST controller with one GET method.

The configuration, to fetch our server, must be placed in `application.properties` file. Spring Boot 2.4 introduced a new way to load configuration data using the `spring.config.import` property which is now the default way to bind to Config Server.
```
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RefreshScope
@RestController
public class MessageRestController {

    @Value("${message:Hello default - Config Server is not working..pelase check}")
    private String message;

    @RequestMapping("/message")
    String getMessage() {
        return this.message;
    }
}
```

In addition to the application name, we also put the active profile and the connection-details in our `application.properties`:
```
spring.application.name=config-server-client

#Active Profile - will relate to development properties file in the server.
#If this property is absent then,default profile will be activated which is
#theh property file without any environment name at the end.
spring.profiles.active=dev
spring.config.import=optional:configserver:http://localhost:8888

management.endpoints.web.exposure.include=*
management.security.enabled=false
```
This will connect to the Config Server at http://localhost:8888 and will also use HTTP basic security while initiating the connection. We can also set username and password separately using spring.cloud.config.username and spring.cloud.config.password properties respectively.

In some cases, we may want to fail the startup of a service if it isn't able to connect to the Config Server. If this is the desired behavior, we can remove the optional: prefix to make the client halt with an exception.

To test, if the configuration is properly received from our server and the message value gets injected in our controller method, we simply curl it after booting the client:
```
> curl http://localhost:8080/message
```
If the response is as follows, our Spring Cloud Config Server and its client are working fine for now:
```
Hello Javawiz- this is from config server - Development Environment
```
Change the above message key in the `config-server-client-dev.properties` file in the Git repository to something different (`Hello User- this is from config server - Development Environment`, perhaps?). We can confirm that the Config Server sees the change by visiting `http://localhost:8888/config-server-client/dev`. 

We need to invoke the refresh Spring Boot Actuator endpoint in order to force the client to refresh itself and draw in the new value. Spring Boot’s Actuator exposes operational endpoints (such as health checks and environment information) about an application. To use it, we must add `org.springframework.boot:spring-boot-starter-actuator` to the client application’s classpath. We can invoke the refresh Actuator endpoint by sending an empty HTTP POST to the client’s refresh endpoint: `http://localhost:8080/actuator/refresh`. Then we can confirm it worked by visiting the `http://localhost:8080/message` endpoint.

The following command invokes the Actuator’s refresh command:
```
> curl localhost:8080/actuator/refresh -d {} -H "Content-Type: application/json
```
We should set `management.endpoints.web.exposure.include=*` in the client application to make this is easy to test (since Spring Boot 2.0, the Actuator endpoints are not exposed by default). By default, we can still access them over JMX if you do not set the flag.

Congratulations! We have just used Spring boot to centralize configuration for all of our services by first standing up a service and then dynamically updating its configuration.








