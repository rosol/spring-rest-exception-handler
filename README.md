Spring REST Exception handler
=============================
[![Build Status](https://travis-ci.org/jirutka/spring-rest-exception-handler.svg)](https://travis-ci.org/jirutka/spring-rest-exception-handler)
[![Coverage Status](https://img.shields.io/coveralls/jirutka/spring-rest-exception-handler/master.svg?style=flat)](https://coveralls.io/r/jirutka/spring-rest-exception-handler?branch=master)
[![Maven Central](https://maven-badges.herokuapp.com/maven-central/cz.jirutka.spring/spring-rest-exception-handler/badge.svg?style=flat)](https://maven-badges.herokuapp.com/maven-central/cz.jirutka.spring/spring-rest-exception-handler)

The aim of this project is to provide a convenient exception handler (resolver) for RESTful APIs that meets a
best-practices for error responses without repeating yourself. It’s very easy to handle custom exceptions, customize
error responses and even localize them. Also solves some pitfalls\* in Spring MVC with a content negotiation when
producing error responses.

\* _Nothing terrible, Spring MVC is still a far better then JAX-RS for RESTful APIs!  ;)_


Error message
-------------

Error messages generated by `ErrorMessageRestExceptionHandler` follows the [Problem Details for HTTP APIs][http-problem]
specification.

For an example, the following error message describes a validation exception.

**In JSON format:**

```json
{
    "type": "http://example.org/errors/validation-failed",
    "title": "Validation Failed",
    "status": 422,
    "detail": "The content you've send contains 2 validation errors.",
    "errors": [{
        "field": "title",
        "message": "must not be empty"
    }, {
        "field": "quantity",
        "rejected": -5,
        "message": "must be greater than zero"
    }]
}
```

**… or in XML:**

```xml
<problem>
    <type>http://example.org/errors/validation-failed</type>
    <title>Validation Failed</title>
    <status>422</status>
    <detail>The content you've send contains 2 validation errors.</detail>
    <errors>
        <error>
            <field>title</field>
            <message>must not be empty</message>
        </error>
        <error>
            <field>quantity</field>
            <rejected>-5</rejected>
            <message>must be greater than zero</message>
        </error>
    </errors>
</problem>

```


How does it work?
-----------------

### RestHandlerExceptionResolver

The core class of this library that resolves all exceptions is [RestHandlerExceptionResolver]. It holds a registry of
`RestExceptionHandlers`.

When your controller throws an exception, the `RestHandlerExceptionResolver` will:

1.  Find an exception handler by the thrown exception type (or its supertype, supertype of the supertype… up to the
    `Exception` class if no more specific handler is found) and invoke it.
2.  Find the best matching media type to produce (using [ContentNegotiationManager], utilises _Accept_ header by
    default). When the requested media type is not supported, then fallback to the configured default media type.
3.  Write the response.


### RestExceptionHandler

Implementations of the [RestExceptionHandler] interface are responsible for converting the exception into Spring’s
[ResponseEntity] instance that contains a body, headers and a HTTP status code.

The main implementation is [ErrorMessageRestExceptionHandler] that produces the `ErrorMessage` body (see above for an
example). All the attributes (besides status) are loaded from a properties file (see the section about
[localizable attributes](#localizable-attributes)). This class also logs the exception (see the
[Exception logging](#exception-logging) section).


Configuration
-------------

### Java-based configuration

```java
@EnableWebMvc
@Configuration
public class RestContextConfig extends WebMvcConfigurerAdapter {

    @Override
    public void configureHandlerExceptionResolvers(List<HandlerExceptionResolver> resolvers) {
        resolvers.add( exceptionHandlerExceptionResolver() ); // resolves @ExceptionHandler
        resolvers.add( restExceptionResolver() );
    }

    @Bean
    public RestHandlerExceptionResolver restExceptionResolver() {
        return RestHandlerExceptionResolver.builder()
                .messageSource( httpErrorMessageSource() )
                .defaultContentType(MediaType.APPLICATION_JSON)
                .addErrorMessageHandler(EmptyResultDataAccessException.class, HttpStatus.NOT_FOUND)
                .addHandler(MyException.class, new MyExceptionHandler())
                .build();
    }

    @Bean
    public MessageSource httpErrorMessageSource() {
        ReloadableResourceBundleMessageSource m = new ReloadableResourceBundleMessageSource();
        m.setBasename("classpath:/org/example/messages");
        m.setDefaultEncoding("UTF-8");
        return m;
    }

    @Bean
    public ExceptionHandlerExceptionResolver exceptionHandlerExceptionResolver() {
        ExceptionHandlerExceptionResolver resolver = new ExceptionHandlerExceptionResolver();
        resolver.setMessageConverters(HttpMessageConverterUtils.getDefaultHttpMessageConverters());
        return resolver;
    }
}
```

### XML-based configuration

```xml
<bean id="compositeExceptionResolver"
      class="org.springframework.web.servlet.handler.HandlerExceptionResolverComposite">
    <property name="order" value="0" />
    <property name="exceptionResolvers">
        <list>
            <ref bean="exceptionHandlerExceptionResolver" />
            <ref bean="restExceptionResolver" />
        </list>
    </property>
</bean>

<bean id="restExceptionResolver"
      class="cz.jirutka.spring.exhandler.RestHandlerExceptionResolverFactoryBean">
    <property name="messageSource" ref="httpErrorMessageSource" />
    <property name="defaultContentType" value="application/json" />
    <property name="exceptionHandlers">
        <map>
            <entry key="org.springframework.dao.EmptyResultDataAccessException" value="404" />
            <entry key="org.example.MyException">
                <bean class="org.example.MyExceptionHandler" />
            </entry>
        </map>
    </property>
</bean>

<bean id="exceptionHandlerExceptionResolver"
      class="org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver" />

<bean id="httpErrorMessageSource"
      class="org.springframework.context.support.ReloadableResourceBundleMessageSource"
      p:basename="classpath:/org/example/errorMessages"
      p:defaultEncoding="UTF-8" />
```

### Another resolvers

The [ExceptionHandlerExceptionResolver] is used to resolve exceptions through [@ExceptionHandler] methods. It must be
registered _before_ the RestHandlerExceptionResolver. If you don’t have any `@ExceptionHandler` methods, then you can
omit the `exceptionHandlerExceptionResolver` bean declaration.


### Default handlers

Builder and FactoryBean registers a set of the default handlers by default. This can be disabled by setting
`withDefaultHandlers` to false.


### Localizable error messages

Message values are read from a _properties_ file through the provided [MessageSource], so it can be simply customized
and localized. Library contains a default [messages.properties] file that is implicitly set as a parent (i.e. fallback)
of the provided message source. This can be disabled by setting `withDefaultMessageSource` to false (on a builder or
factory bean).

The key name is prefixed with a fully qualified class name of the Java exception, or `default` for the default value;
this is used when no value for a particular exception class exists (even in the parent message source).

Value is a message template that may contain [SpEL] expressions delimited by `#{` and `}`. Inside an expression, you
can access the exception being handled and the current request (instance of [HttpServletRequest]) under the `ex`, resp.
`req` variables.

**For an example:**

```properties
org.springframework.web.HttpMediaTypeNotAcceptableException.type=http://httpstatus.es/406
org.springframework.web.HttpMediaTypeNotAcceptableException.title=Not Acceptable
org.springframework.web.HttpMediaTypeNotAcceptableException.detail=\
    This resource provides #{ex.supportedMediaTypes}, but you've requested #{req.getHeader('Accept')}.
```

### Exception logging

Exceptions handled with status code 5×× are logged on ERROR level (incl. stack trace), other exceptions are logged on
INFO level without a stack trace, or on DEBUG level with a stack trace if enabled. The logger name is
`cz.jirutka.spring.exhandler.handlers.RestExceptionHandler` and a Marker is set to the exception’s full
qualified name.


### Why is 404 bypassing exception handler?

When the [DispatcherServlet] is unable to determine a corresponding handler for an incoming HTTP request, it sends 404
directly without bothering to call an exception handler (see
[on StackOverflow](http://stackoverflow.com/a/22751886/2217862)). This behaviour can be changed, **since
Spring 4.0.0**, using `throwExceptionIfNoHandlerFound` init parameter. You should set this to true for a consistent
error responses.


**When using WebApplicationInitializer:**

```java
public class AppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    protected void customizeRegistration(ServletRegistration.Dynamic reg) {
        reg.setInitParameter("throwExceptionIfNoHandlerFound", "true");
    }
    ...
}
```

**…or classic web.xml:**

```xml
<servlet>
    <servlet-name>rest-dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>throwExceptionIfNoHandlerFound</param-name>
        <param-value>true</param-value>
    </init-param>
    ...
</servlet>
```


Maven
-----

Released versions are available in The Central Repository. Just add this artifact to your project:

```xml
<dependency>
    <groupId>cz.jirutka.spring</groupId>
    <artifactId>spring-rest-exception-handler</artifactId>
    <version>1.0.3</version>
</dependency>
```

However if you want to use the last snapshot version, you have to add the Sonatype OSS repository:

```xml
<repository>
    <id>sonatype-snapshots</id>
    <name>Sonatype repository for deploying snapshots</name>
    <url>https://oss.sonatype.org/content/repositories/snapshots</url>
    <snapshots>
        <enabled>true</enabled>
    </snapshots>
</repository>
```


Requirements
------------

*  Spring 3.2.0.RELEASE and newer is supported, but 4.× is highly recommended.
*  Jackson 1.× and 2.× are both supported and optional.


References
----------

*  [Improve Your Spring REST API by M. Severson](http://www.jayway.com/2012/09/23/improve-your-spring-rest-api-part-ii)
*  [Spring MVC REST Exception Handling Best Practices by L. Hazlewood](https://stormpath.com/blog/spring-mvc-rest-exception-handling-best-practices-part-1/)
*  [Exception Handling in Spring MVC by P. Chapman](http://spring.io/blog/2013/11/01/exception-handling-in-spring-mvc)
*  [IETF draft Problem Details for HTTP APIs by M. Nottingham][http-problem]


License
-------

This project is licensed under [Apache License 2.0](http://www.apache.org/licenses/LICENSE-2.0.html).


[http-problem]: http://tools.ietf.org/html/draft-nottingham-http-problem-06
[ContentNegotiationManager]: http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/accept/ContentNegotiationManager.html
[DispatcherServlet]: http://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/DispatcherServlet.html
[@ExceptionHandler]: http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/bind/annotation/ExceptionHandler.html
[ExceptionHandlerExceptionResolver]: http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/mvc/method/annotation/ExceptionHandlerExceptionResolver.html
[HttpServletRequest]: http://docs.oracle.com/javaee/7/api/javax/servlet/http/HttpServletRequest.html
[MessageSource]: http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/context/MessageSource.html
[ResponseEntity]: http://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/http/ResponseEntity.html
[SpEL]: http://docs.spring.io/spring/docs/current/spring-framework-reference/html/expressions.html

[messages.properties]: src/main/resources/cz/jirutka/spring/exhandler/messages.properties
[ErrorMessageRestExceptionHandler]: cz/jirutka/spring/exhandler/handlers/ErrorMessageRestExceptionHandler.java
[RestExceptionHandler]: cz/jirutka/spring/exhandler/handlers/RestExceptionHandler.java
[RestHandlerExceptionResolver]: cz/jirutka/spring/exhandler/RestHandlerExceptionResolver.java
