---
title: "Java Spring Boot Getting Started"
date: 2017-11-04T16:04:16Z
tags: [ "Java", "Spring boot" ]
---

## Part 1: Basic set up

> Steps inspired by [spring boot getting started guide](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#getting-started-first-application).

Add some basic dependencies and Java file to return Hello world!

> [See the code on github](https://github.com/iyp-uk/sprint-boot-basics/tree/part1).

## Part 2: Using spring boot

### Typical project layout

```
com
 +- example
     +- myapplication
         +- Application.java
         |
         +- customer
         |   +- Customer.java
         |   +- CustomerController.java
         |   +- CustomerService.java
         |   +- CustomerRepository.java
         |
         +- order
             +- Order.java
             +- OrderController.java
             +- OrderService.java
             +- OrderRepository.java
```

Where `Application.java` declares the `main` method, being the app's entry point, like so:

```java
package com.example.myapplication;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@EnableAutoConfiguration
@ComponentScan
public class Application {

	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}

}
```
> Reference: [Locating the Main Application Class](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#using-boot-locating-the-main-class).

### Configuration

Favour **Java-based configuration** classes over XML.

> Note that you can mix and match XML and Classes. Loading XML-based config via `@ImportResource`.

> Reference: [Configuration Classes](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#using-boot-configuration-classes).

#### Gradually replace the auto-configuration with yours

It could be handy to start with auto-configuration, while seeing which config is applied using `--debug` when starting the app like so:

```console
$ java -jar myproject-0.0.1-SNAPSHOT.jar --debug
```

You would need to use the  `@EnableAutoConfiguration` or `@SpringBootApplication` on your `@Configuration` class.

> `@SpringBootApplication` is equivalent to `@Configuration`, `@EnableAutoConfiguration` and `@ComponentScan` together.  

> Reference: [Auto-configuration](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#using-boot-auto-configuration).

### Developer tools

Check the `spring-boot-devtools` module! 
It provides useful things like [automatic restart](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#using-boot-devtools-restart),
[LiveReload](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#using-boot-devtools-livereload),
and [Remote tools](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#using-boot-devtools-remote).

> Reference: [Developer Tools](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#using-boot-devtools).

## Part 3: Spring boot features

### Application Events and Listeners

Events are sent in the following order as the application runs:

| Event | When |
|---|---|
| `ApplicationStartingEvent` | Start of a run, before any processing **except** registration of listeners and initialisers |
|`ApplicationEnvironmentPreparedEvent`| When `Environment` to use is known but *before* the context is created |
| `ApplicationPreparedEvent` | After bean definitions are loaded but *before* refresh is started |
| `ApplicationReadyEvent` | After refresh + related callbacks have been processed = App ready for service requests |
| `ApplicationFailedEvent` | Startup exception has been raised |

Be careful when using a hierarchy of `SpringApplication` instances, as **a listener may receive multiple instances of the same type of application event**.
Implement `ApplicationContextAware` to inject context and compare it with the event's context.
If the listener is a `@Bean`, use `@Autowired`. 

> Reference: [Application Events and Listeners](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-application-events-and-listeners).

### Web environment

Call `setWebEnvironment(false)` when using `SpringApplication` within a JUnit test.

> Reference: [Web Environment](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-web-environment).

### Externalised configuration

Run the same application code no matter what the environment!

Pass your configuration with:

* Properties files
* YAML files
* Environment variables
* Command-line arguments

Property values can then be injected into beans using `@Value` accessed via:

* Spring's `Environment` abstraction
* `@ConfigurationProperties` with [structured objects bindings](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-external-config-typesafe-configuration-properties)

Properties are considered [in a specific order](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-external-config), go have a look at it.

> Reference: [Externalized configuration](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-external-config).

#### Relaxed binding

Interestingly, there is no need for an exact match between `Environment` and `@ConfigurationProperties` beans property names.

Example:

```java
@ConfigurationProperties(prefix="person")
public class OwnerProperties {

	private String firstName;

	public String getFirstName() {
		return this.firstName;
	}

	public void setFirstName(String firstName) {
		this.firstName = firstName;
	}

}
```

To pass the `firstName` property you can use:

| Property | Format | Usage |
|---|---|---|
| `person.firstName` | Camel case | Standard |
| `person.first-name`| Kebab case | Recommended in `.properties` and `.yml` files |
| `person.first_name`| Underscore notation | *Alternative format* in `.properties` and `.yml` files |
| `PERSON_FIRSTNAME`| Upper case | Environment variables |

> Reference: [Relaxed binding](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-external-config-relaxed-binding).

### Profiles

Segregate parts of your application to make them available only in certain environments.

> Applies to `@Component` and `@Configuration` when marked with `@Profile`.

```java
@Configuration
@Profile("production")
public class ProductionConfiguration {

	// ...

}
```
Then, specify which profiles are active using `spring.profiles.active=dev,hsqldb` or on the command line with `--spring.profiles.active=dev,hsqldb`. 

Profiles also allow [per-profile configuration files](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-external-config-profile-specific-properties).

> Reference: [Profiles](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-profiles).

### Developing web applications

* [Spring MVC](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-spring-mvc) which covers:
    * Auto-config
    * `HttpMessageConverters`
    * Custom JSON serializers and deserializers
    * `MessageCodesResolver`
    * Static content, including Favicon
    * `ConfigurableWebBindingInitializer`
    * Template engines
    * Error Handling, including custom error pages (Possibly outside of Spring MVC)
    * Spring [HATEOAS](https://spring.io/understanding/HATEOAS)
    * [CORS](https://spring.io/understanding/CORS)
* [Spring WebFlux](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-webflux) for async which covers:
    * Auto-config
    * HTTP Codecs with HttpMessageReaders and HttpMessageWriters
    * Static content
    * Template engines
    * Error Handling, including custom error pages
* [JAX-RS and Jersey](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-jersey)
* [Embedded Servlet Container Support](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-embedded-container)   

> Reference: [Developing web applications](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-developing-web-applications).

> More on [Spring webflux](https://docs.spring.io/spring/docs/current/spring-framework-reference/web-reactive.html)

### Security

Defaults to basic auth when enabled. OAuth2 also supported OOTB.

Refer to the [Spring Security Reference](https://docs.spring.io/spring-security/site/docs/5.0.0.RC1/reference/htmlsingle#jc-method).

> Reference: [Security](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-security)

### Data stores

Depending on your needs, 
refer to [SQL databases](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-sql)
and [NoSQL databases](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-nosql)
(which includes [SolR](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-solr) 
and [ElasticSearch](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-elasticsearch)) 
documentations where you will find details about common data stores. 

### Caching

Refer to [Spring documentation](https://docs.spring.io/spring/docs/5.0.1.RELEASE/spring-framework-reference/integration.html#cache) for details.

Example:

```java
import org.springframework.cache.annotation.Cacheable
import org.springframework.stereotype.Component;

@Component
public class MathService {

	@Cacheable("piDecimals")
	public int computePiDecimal(int i) {
		// ...
	}

}
```

Here, before invoking `computePiDecimal`, 

1. abstraction looks for an entry in `piDecimals` cache that matches the `i` argument
1. **if** an entry is found, 
    * content retrieved from the cache is returned to the caller and `computePiDecimal` is not invoked
1. **else**
    * Method is invoked and `piDecimals` cache is updated before returning the value to the caller.

> Reference: [Caching](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-caching).
 
#### Cache providers

Many cache providers are supported, including Couchbase and Redis. [See the list of supported cache providers](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#_supported_cache_providers).

#### Disabling caching

You may want to disable caching altogether in certain environments even if `@EnableCaching` is present in configuration.

A handy way of doing it would be to force the `none` cache type:

```
spring.cache.type=none
```

### Messaging

Support for both JMS and AMPQ.

* JMS (Java Message Service)
    * ActiveMQ
    * Artemis
* AMQP (Advanced Message Queueing Protocol)
    * RabbitMQ
    * Apache Kafka

> Reference: [Messaging](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-messaging).

### REST clients

#### `RestTemplate` 

`RestTemplate` is a convenient way of handling your calls to other REST services. 

Example:

```java
@Service
public class MyService {

	private final RestTemplate restTemplate;

	public MyBean(RestTemplateBuilder restTemplateBuilder) {
		this.restTemplate = restTemplateBuilder.build();
	}

	public Details someRestCall(String name) {
		return this.restTemplate.getForObject("/{name}/details", Details.class, name);
	}

}
```

> Reference: [Calling REST Services with RestTemplate](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-resttemplate)

#### `WebClient`

When in use with WebFlux. `WebClient` is fully reactive.

Example:

```java
@Service
public class MyService {

	private final WebClient webClient;

	public MyBean(WebClient.Builder webClientBuilder) {
		this.webClient = webClientBuilder.baseUrl("http://example.org").build();
	}

	public Mono<Details> someRestCall(String name) {
		return this.webClient.get().url("/{name}/details", name)
						.retrieve().bodyToMono(Details.class);
	}

}
```

> Reference: [Calling REST Services with WebClient](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-webclient).

### Testing

> Reference: [Testing](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-testing)

### WebSockets

Accessible through the `spring-boot-starter-websocket` module.
Documentation on [spring framework websockets](https://docs.spring.io/spring/docs/5.0.1.RELEASE/spring-framework-reference/web.html#websocket)

> [MQTT](http://mqtt.org/) is available through [sprint integration](https://docs.spring.io/spring-integration/reference/html/mqtt.html)

> Reference: [Websockets](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-websockets).

## Further reading

* [Spring and IntelliJ IDEA getting started guide](https://spring.io/guides/gs/intellij-idea/)
* [Spring boot docker getting started guide](https://spring.io/guides/gs/spring-boot-docker/)

