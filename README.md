# Realizing Composition and Inheritance wiht OpenAPI Generator (v.3.0.1) in Spring Boot 3 and Maven
_A Practical Guide_
[![SonarQube Analysis](https://github.com/mhamp/spring-openapi-demo/actions/workflows/build.yml/badge.svg?branch=main)](https://github.com/mhamp/spring-openapi-demo/actions/workflows/build.yml)
[![Quality Gate Status](https://sonarcloud.io/api/project_badges/measure?project=mhamp_spring-openapi-demo&metric=alert_status)](https://sonarcloud.io/summary/new_code?id=mhamp_spring-openapi-demo)


by [Mathias Hamp](https://github.com/mhamp)  - Munich, June 2025


This guide covers advanced modeling techniques in OpenAPI with practical implementation details for Spring Boot applications, including package structures, annotations, and code generation configurations.

This guide demonstrates how to:

* Generate Java classes using OpenAPI Generator.
* Model inheritance and composition in OpenAPI.
* Share models across Maven modules in a Monorepo.
* Resolve common pitfalls (e.g., Spring Boot fat JARs, dependency scopes).

## Table of Contents

<!-- TOC -->
* [Table of Contents](#table-of-contents)
* [Project Structure](#project-structure)
* [General Project Setup](#general-project-setup)
  * [The Purpose of a Super POM](#the-purpose-of-a-super-pom)
  * [Best Practices](#best-practices)
* [General OpenAPI Configuration ](#general-openapi-configuration)
    * [Core Dependencies](#core-dependencies)
    * [Configuring the OpenAPI Generator Plugin](#configuring-the-openapi-generator-plugin)
    * [Writing the OpenAPI Specification](#writing-the-openapi-specification)
    * [Spec Validation with Swagger Editor](#validating-your-api-specification)
* [Realizing Composition (Has-A Relationsshiop)](#modeling-composition-in-openapi)
* [Modeling Inheritance (Is-A Relationship)](#modeling-inheritance-in-openapi)
* [Modeling Composition in OpenAPI](#modeling-composition-in-openapi)
* [Sharing Generated Resources across Maven Modules](#sharing-models-across-maven-modules)
* [Limitations of OpenAPI Code Generation for Other Protocols](#)
* [Conclusion](#conclusion)
 
<!-- TOC -->

## Project Structure
```Bash
spring-openapi-demo/  
├── api-contract/               # OpenAPI specs + generated code  
│   ├── src/main/resources/openapi.yml  
│   ├── pom.xml  
├── commons/                    # Module consuming shared models  
│   ├── src/main/java/com/xontext/  
│   ├── pom.xml  
├── commons/                    # Demo service
│   ├── src/main/java/com/xontext/  
│   ├── pom.xml  
└── pom.xml                     # Parent POM  
```

## General Project Setup
Here is a description of how to configure your POMs to have a working OpenAPI Generator ready to provide your with all your necessities.

### The Purpose of a Super POM
In a Maven multi-module project, the Super POM (or Parent POM) acts as the foundational configuration for all child modules. It serves two primary roles:
#### 1. Referencing All Project Modules
The Super POM is not a buildable Maven project itself but instead:
* Declares all child modules in the <modules> section.
* Ensures consistent project structure and build order.

Example:
```xml
<!-- Parent POM (pom.xml) -->
<modules>
    <module>api-contract</module>
    <module>commons</module>
    <module>app-web</module>
</modules>
```
#### 2. Managing Dependencies and Plugins
The Super POM centralizes dependency/plugin versions but avoids defining direct dependencies (to prevent unintended inheritance). Instead, it uses:

* `<dependencyManagement>`: Declares versions for shared dependencies (e.g., Spring Boot, OpenAPI).
* `<pluginManagement>`: Standardizes plugin configurations (e.g., compiler settings).

### Best Practices

* Do not define `<dependencies>` in the Super POM (to avoid forcing all modules to inherit them).
* Do use `<dependencyManagement>` to avoid version conflicts.

#### Example:

```xml
<!-- Parent POM (pom.xml) -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>3.2.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<build>
    <pluginManagement>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.11.0</version>
                <configuration>
                    <source>21</source>
                    <target>21</target>
                </configuration>
            </plugin>
        </plugins>
    </pluginManagement>
</build>
```
#### Why This Matters
1. Consistency: All modules use the same dependency versions.
2. Flexibility: Child modules opt-in to dependencies (no forced inheritance).
3. Maintainability: Version updates happen in one place (the Super POM).

#### Anti-Pattern:
❌ Defining <dependencies> directly in the Super POM forces all modules to inherit them, even if unused.

#### Key Takeaway:

> The Super POM is a configuration hub, not a buildable project. Use it to standardize versions, not enforce dependencies."

This revision improves readability, adds concrete examples, and clarifies the "why" behind the structure. Let me know if you'd like to expand further!

## General OpenAPI Configuration

### Core Dependencies
Your POM includes essential dependencies for a functional OpenAPI generator setup with Spring Boot 3:
#### 1. SpringDoc OpenApi (UI integration)
```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
</dependency>
```
* **Purpose:** Provides Swagger UI integration and auto-generates OpenAPI documentation from Spring controllers.
* **Version:** Inherited from Spring Boot parent POM (recommended).

#### 2. Jakarta EE 9+ Dependencies
Spring Boot 3+ uses Jakarta EE 9+ packages (replacing javax.*):
```xml
<!-- Annotations (@Generated, @PostConstruct, etc.) -->
<dependency>
    <groupId>jakarta.annotation</groupId>
    <artifactId>jakarta.annotation-api</artifactId>
    <version>3.0.0</version>
</dependency>

        <!-- Bean Validation (@Valid, @NotNull, etc.) -->
<dependency>
<groupId>jakarta.validation</groupId>
<artifactId>jakarta.validation-api</artifactId>
<version>3.1.1</version>
</dependency>

        <!-- Servlet API (for generated controllers) -->
<dependency>
<groupId>jakarta.servlet</groupId>
<artifactId>jakarta.servlet-api</artifactId>
<version>6.1.0</version>
<scope>provided</scope>
</dependency>
```
* <b>Key Point:</b> These replace their javax.* counterparts and are mandatory for Spring Boot 3 compatibility.

#### Final Dependency Checklist

| Scope | Dependency | Purpose |
|---|---|---|
|Compile | springdoc-openapi-starter-webmvc-ui	 | API Documentation |
|Compile | jakarta.annotation-api	 | Annotations |
|Compile | jakarta.validation-api	 | Validation |
|Provided | jakarta.servlet-api	 | Servlet API |
|Plugin | openapi-generator-maven-plugin	 | Code Generation |





### Configuring the OpenAPI Generator Plugin
The plugin is configured to generate Spring Boot 3-compatible code:

```xml
<plugin>
    <groupId>org.openapitools</groupId>
    <artifactId>openapi-generator-maven-plugin</artifactId>
    <version>7.5.0</version>
    <configuration>
        <generatorName>spring</generatorName>
        <configOptions>
            <useSpringBoot3>true</useSpringBoot3>  <!-- Critical for Jakarta EE 9+ -->
            <openApiNullable>false</openApiNullable>  <!-- Disable JSON nullable fields -->
        </configOptions>
    </configuration>
</plugin>
```
#### Key Option
| Option            | Purpose                          | Default | Recommendation                     |
|-------------------|----------------------------------|---------|------------------------------------|
| `useSpringBoot3`  | Generates Jakarta EE 9+ code     | `false` | **Always `true` for Spring Boot 3** |
| `openApiNullable` | Handles `nullable` in OpenAPI specs | `true`  | Set to `false` for non-nullable-by-default |


### Handling `openApiNullable=false`
When `openApiNullable` is `false`:
#### 1. Behavior
* Fields marked as `nullable`: `true` in OpenAPI specs **will not** generate `@Nullable` annotations .
* All fields are treated as **non-nullable** unless explicitly configured otherwise.
#### 2. Impact on Generated Code:
```java
// With openApiNullable=false
public class Product {
    private String id;  // No @Nullable annotation
}

// With openApiNullable=true (default)
public class Product {
    @Nullable  // Added if field is nullable in OpenAPI spec
    private String id;
}
```
#### 3. Validation:
* Combine with `jakarta.validation-api` to enforce non-nullability:
```java
public class Product {
    @NotBlank  // Explicit validation
    private String id;
}
```

### Optional: Jackson Nullable Support
If you need nullable fields, add this dependency to your POM:
```xml
<dependency>
    <groupId>org.openapitools</groupId>
    <artifactId>jackson-databind-nullable</artifactId>
    <version>0.2.6</version>
</dependency>
```

* **Purpose:** Provides `@JsonNullable` for explicit nullability in DTOs.
* **Use Case:** Only needed if `openApiNullable=true` and you want fine-grained control.

### Build Configuration Notes
#### Fat JAR Disabled:
As a default Spring Boot repackages classes under `BOOT-INF/classes/` which causes `Package not found` errors. You may prevent this kind of error by skipping fat JAR generation as follows:
```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <skip>true</skip>  <!-- Critical for sharing models as a library -->
        <classifier>exec</classifier>  <!-- Optional: Build a separate executable JAR -->
    </configuration>
</plugin>
```
* Prevents Spring Boot from repackaging classes into `BOOT-INF/classes/`, making them inaccessible to other modules.

### Writing the OpenAPI Specification
Find here a basic overview of how to define your API contract
#### Basic Structure
Every OpenAPI spec starts with these required fields:
```yml
openapi: 3.0.3  # Version
info:
  title: Pet Store API
  version: 1.0.0
  description: Manage pets and their owners
servers:
  - url: https://api.example.com/v1
```
#### Paths & Operations
Define endpoints with HTTP methods:

```yaml
paths:
  /pets:
    get:
      summary: List all pets
      responses:
        '200':
          description: A list of pets
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/Pet'
```
#### Components (Reusable Objects)
##### Schemas (Data Models)

```yaml
components:
  schemas:
    Pet:
      type: object
      required:
        - id
        - name
      properties:
        id:
          type: integer
          format: int64
        name:
          type: string
        tag:
          type: string
          nullable: true  # Optional field
```
##### Enums

```yaml
 PetStatus:
   type: string
   enum: [available, pending, sold]
```

##### External References
Split large specs into multiple files by using relative path (`./schemas/models.yml`) in you `$ref`. :

```yaml
# main.yaml
paths:
  /users:
    $ref: './paths/users.yaml'

# paths/users.yaml
get:
  summary: Get all users
  responses: ...
```
**Note**: For many schemas, you can split them across multiple files (e.g., User.yml, Product.yml) and reference them individually.

### Validating your API specification
You may want to write and validate your OpenAPI inputSpec in a dedicated editor offering an easy way to get started and validate your specification:
* [Swagger Editor](https://editor.swagger.io/) by Swagger
* [API Hub](https://swagger.io/api-hub/) by SmartBear.
* [Redoc](https://redocly.com/) by Redocly

## Modeling Composition in OpenAPI
Generated models will reside in whatever package is defined in the input specification configuration section. You may also :
```xml
<modelPackage>org.example.model</modelPackage>
```

### OpenAPI Definition
```yml
components:
  schemas:
    Order:
      type: object
      properties:
        id:
          type: string
        customer:
          $ref: '#/components/schemas/Customer'  # Composition
        items:
          type: array
          items:
            $ref: '#/components/schemas/OrderItem'
```
### Generated Java (Spring Boot)
```java
// com.xontext.ecommerce.model.Order
@Generated
public class Order {
    @JsonProperty("id")
    private String id;

    @Valid  // Enables validation cascade
    @JsonProperty("customer")
    private Customer customer;

    @JsonProperty("items")
    @Valid
    private List<OrderItem> items = new ArrayList<>();
}
```

**Key Annotations**
* `@Valid`: Enables validation of nested objects
* `@JsonProperty`: Jackson annotation for JSON mapping

## Modeling Inheritance in OpenAPI

### OpenAPI Definition (Polymorphism)
```yml
components:
  schemas:
    Vehicle:
      type: object
      discriminator:
        propertyName: vehicleType
      required:
        - vehicleType
      properties:
        vehicleType:
          type: string
        make:
          type: string
          
    Car:
      allOf:
        - $ref: '#/components/schemas/Vehicle'
        - type: object
          properties:
            trunkSize:
              type: number
```
### Generated Java Code

```java
// com.xontext.transport.model.Vehicle
@JsonTypeInfo(
  use = JsonTypeInfo.Id.NAME,
  include = JsonTypeInfo.As.PROPERTY,
  property = "vehicleType")
@JsonSubTypes({
  @JsonSubTypes.Type(value = Car.class, name = "Car")
})
public class Vehicle {
  @JsonProperty("vehicleType")
  private String vehicleType;
  
  @JsonProperty("make")
  private String make;
}

// com.xontext.transport.model.Car
public class Car extends Vehicle {
  @JsonProperty("trunkSize")
  private Double trunkSize;
}
```
**Key Annotations:**
* `@JsonTypeInfo`: Configures polymorphic type handling 
* `@JsonSubTypes`: Defines possible subtypes

## Sharing Models Across Maven Modules

To configure a module to share its classes the `openapi-generator-maven-plugin` plugin has to create the resources as executable JARs.
```xml
<plugin>
    <groupId>org.openapitools</groupId>
    <artifactId>openapi-generator-maven-plugin</artifactId>
    <configuration>
        <modelPackage>com.xontext.common.model</modelPackage>  <!-- Shared package -->
        <generateModels>true</generateModels>
        <generateApis>false</generateApis>  <!-- Skip controllers if not needed -->
    </configuration>
</plugin>

<!-- Disable Spring Boot fat JAR for dependency sharing -->
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <skip>true</skip> <!-- Critical for sharing models as a library -->
        <classifier>exec</classifier>  <!-- Optional: Build a separate executable JAR -->
    </configuration>
</plugin>
```
**Key Point for sharing resources**
* Disable Spring Boot fat JAR repackaging (<skip>true</skip>).
* Verify generated classes are in the JAR root (not under BOOT-INF).
* Add a dependency in your POM to the module holding the resources it depends upon.  

This setup ensures clean code generation and seamless integration with Spring Boot 3.

## Limitations of OpenAPI Code Generation for Other Protocols
OpenAPI is designed specifically to describe RESTful HTTP APIs. As such, its tooling and generators are well-suited for REST but fall short when applied to alternative architectures and protocols:
* **gRPC**: Not supported natively by OpenAPI: gRPC uses Protocol Buffers (.proto) instead of JSON/YAML-based specifications.
* **GraphQL**: OpenAPI is incompatible with GraphQL: GraphQL has a completely different schema language and runtime query model.
* **OpenFeign**: Partial compatibility: OpenFeign clients can be generated using OpenAPI (e.g., with the java generator using feign libraries).
* **SOAP**: Completely incompatible: SOAP uses WSDL, not OpenAPI.

| Protocol | OpenAPI Support | Notes |
| --- | --- | --- |
| gRPC	|  Not supported	| Use .proto and protoc | 
| GraphQL	|  Not supported	| Use .graphqls and GraphQL-specific tools | 
| OpenFeign	|  Partial support	| Use java + feign lib; expect customization | 
| SOAP	|  Not supported	| Use WSDL with CXF, JAX-WS, or wsimport | 

## Conclusion
* Inheritance: Use discriminator in OpenAPI.
* Composition: Nest schemas with $ref.
* Dependency Sharing: Disable Spring Boot fat JARs or use a dedicated api-models module.
* Troubleshooting: Verify JAR contents with jar tf and enforce proper Maven dependencies.
For a complete example, see the GitHub repository.

***
License: [MIT](LICENCE.txt)
Author: [Mathias Hamp](https://github.com/mhamp)
Last Updated: 2025-05-25
***