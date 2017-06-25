---
layout: post
title: "SpringMVC + Swagger"
description: "Configuring Swagger on a SpringMVC application"
date: 2016-08-30
tags: [spring, rest, swagger, java]
comments: false
share: true
---


[Swagger](http://swagger.io/) generates documentation for your REST api. It has out-of-the-box support for JAX-RS (e.g. Jersey), but with SpringMVC (in this case, 3.2.16) it’s a bit harder to get things running.

## Configure Springfox

[Springfox](http://springfox.github.io/springfox/docs/current/) (I use 2.5.0) scans the Spring annotations and generates the swagger API description. Add the dependency to the Maven configuration.


```xml
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>2.5.0</version>
    <exclusions>
        <exclusion>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
        </exclusion>
        <exclusion>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>2.5.0</version>
</dependency>
```

After this, adding a [Configuration](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/context/annotation/Configuration.html) bean is enough to get the [Swagger api-docs](http://swagger.io/specification/) resource running. If your *DispatcherServlet* is configured on the web-app root (/), you’ll find the api description on http://localhost:8080/v2/api-docs


```java
@Configuration
@EnableSwagger2
public class SwaggerConfig {    
    @Bean
    public Docket swaggerDocket() {
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(new ApiInfo("Api", "Api Documentation", "x.x.x", "", DEFAULT_CONTACT, "", "")) 
                .select()
                .apis(RequestHandlerSelectors.any())
                .paths(PathSelectors.any())
                .build();
    }
}
```


## Adding Swagger UI to the web application
We already added the springfox-swagger-ui dependency to Maven (see first code snippet). Springfox just wraps the normal [Swagger UI](http://swagger.io/swagger-ui/) to make it more convenient in Spring applications.

By default, swagger-ui will run under your base path. I like to put it under /documentation, so I had to fix 2 things. First, I had to map the static resources in the [Swagger UI](http://springfox.github.io/springfox/docs/current/#swagger-ui) [webjar](http://www.webjars.org/documentation#springmvc) to the right location.


```xml
<!-- Spring XML config to map the Swagger UI webjar -->
<mvc:resources mapping="/documentation/**" location="classpath:/META-INF/resources/"/>
```


Second, I had to redirect the existing swagger resources because the UI starts looking for them relative to its own location. (For this I couldn’t find a better solution, perhaps configuring this in XML will make it more convenient)


```java
@Controller
@Api(
        description = "SwaggerUI is located under /documentation. This mapping redirects the necessary resources for the ui.",
        hidden = true
)
@RequestMapping("/documentation")
public class DocumentationController {

    private static final String DOCUMENTATION_API_DOCS = "/documentation/api-docs";
    private static final String SWAGGER_RESOURCES = "/swagger-resources";
    private static final String CONFIGURATION_UI = "/swagger-resources/configuration/ui";
    private static final String CONFIGURATION_SECURITY = "/swagger-resources/configuration/security";

    @RequestMapping(SWAGGER_RESOURCES)
    public RedirectView swaggerResources() {
        return new RedirectView(SWAGGER_RESOURCES, true);
    }

    @RequestMapping(DOCUMENTATION_API_DOCS)
    public RedirectView swaggerApiDocs() {
        return new RedirectView(DOCUMENTATION_API_DOCS, true);
    }

    @RequestMapping(CONFIGURATION_UI)
    public RedirectView swaggerUI() {
        return new RedirectView(CONFIGURATION_UI, true);
    }

    @RequestMapping(CONFIGURATION_SECURITY)
    public RedirectView swaggerConfigSecurity() {
        return new RedirectView(CONFIGURATION_SECURITY, true);
    }
}
```


## Annotate your API’s

Springfox does a fine job in generating documentation based on the Spring annotations. If you want to provide more information, use the Swagger Core annotations. See [this JAX-RS resource as example](https://github.com/swagger-api/swagger-samples/blob/master/java/java-jaxrs/src/main/java/io/swagger/sample/resource/PetResource.java).


```java
@Controller
@io.swagger.annotations.Api(tags = "Agreement", description = "Agreement")
@RequestMapping(RestEndPoint.BASE_URL)
public class AgreementController extends AbstractController {
   ...
```


## Hide deprecated API’s
To finish my first Swagger implemenation, I tuned the Springfox Docket a bit.
1. I hide deprecated methods with a Guava predicate (see apis() method).
2. I specified a regex for the Paths I want to show (that drops the earlier created DocumentationController in one go)
3. I added some type conversions. If one use the API, these type conversions are done by our Jackson ObjectMapper to generate JSON. Springfox doesn’t seem to be clever enough to get these conversions right.

```java
return new Docket(DocumentationType.SWAGGER_2)
        ...
        .select()        .apis(Predicates.not(RequestHandlerSelectors.withClassAnnotation(Deprecated.class)))
        .paths(PathSelectors.regex("/api/v[0-9]+/.*"))
        .build()
        .directModelSubstitute(LocalDateTime.class, int[].class)
        .directModelSubstitute(LocalDate.class, int[].class)
        .directModelSubstitute(CountryCode.class, String.class)
```


## Conclusion
With a few steps I got Swagger to work for a SpringMVC project using Springfox. It doesn’t feel perfect yet, especially if you used Swagger for JAX-RS projects in the past.
