+++
title = "How to generate sources from multiple OpenAPI specifications with Gradle"
date = "2025-04-06"
updated = "2025-04-29"
copy_button = true
+++

This is a short note on configuring a Java Gradle project that uses the OpenAPI Generator plugin
to generate client/server code from multiple specifications, preventing unnecessary code regeneration
during project builds.  

{{ admonition(type="info", title="Maybe this is fixed already?", text="This is an issue I faced on a project 
in late 2024, and was reproducible at the time of writing this note, but no guarantees this is still relevant.") }}

# Problem

On a project that uses Gradle as a build system, there was an issue when all client/server code
generated from OpenAPI specs was regenerated for every build. This is expected when doing a full rebuild
(a `gradle clean build` locally or in a CI pipeline), but why would you want that when you are just running
your unit tests? This actually makes the process of writing unit tests not very pleasant.

> *I still don't know if that projects' lack of tests was the result of this,
> or just nobody bothered to fix this issue due to the lack of tests* 🤔

# The Cause

In the existing configuration of the project, each generator task used the same folder as a target -- `build/generated/openapi`. 
This worked fine for the API code since each spec was built to a different package and didn't mix.
However, besides the code, the plugin also generates a set of project files (like maven pom.xml, etc.).   
The code didn't mix due to different packages, but those files did, and each next task was overwriting project files 
from the previous.  
Because of that, for every build Gradle considered that the generated code (those project files specifically) 
did not match what was generated before, and it's right about time to go generate everything again.

## Example

Let's make a minimal reproduction of the issue first. Here's the recipe. 

1. Take two OpenAPI specs to your liking to generate Server API interfaces.

**Spec A:**

```yaml,name=book-catalog-api.yaml
openapi: 3.0.3
info:
  title: Book Catalog API
  description: API for managing book catalog information
  version: 1.0.0
servers:
  - url: /api/v1
paths:
  /books:
    get:
      summary: Get all books
      operationId: getAllBooks
      ...
    post:
      summary: Add a new book
      operationId: addBook
      ...
  /books/{bookId}:
    get:
      summary: Get book by ID
      operationId: getBookById
      ...
    put:
      summary: Update book
      operationId: updateBook
      ...
    delete:
      summary: Delete book
      operationId: deleteBook
      ...
components:
  schemas:
    ...
```

**Spec B:**

```yaml,name=book-inventory-api.yaml
openapi: 3.0.3
info:
  title: Book Inventory API
  description: API for managing book inventory and stock
  version: 1.0.0
servers:
  - url: /api/v1
paths:
  /inventory:
    get:
      summary: Get inventory status for all books
      operationId: getInventory
      ...
  /inventory/{bookId}:
    get:
      summary: Get inventory status for a specific book
      operationId: getInventoryItem
      ...
    put:
      summary: Update inventory for a book
      operationId: updateInventory
      ...
  /inventory/{bookId}/stock:
    post:
      summary: Add stock for a book
      operationId: addStock
      ...
  /inventory/{bookId}/checkout:
    post:
      summary: Checkout a book (reduce stock)
      operationId: checkoutBook
      ...
components:
  schemas:
    ...
```

2. Season them with a simple Gradle build

**Build Config:**

```gradle
import org.openapitools.generator.gradle.plugin.tasks.GenerateTask

plugins {
    id 'java'
    id 'org.springframework.boot' version '3.4.5'
    id 'io.spring.dependency-management' version '1.1.7'
    id 'org.openapi.generator' version '7.12.0'
}

// ... generic java project configuration ...

// Generate server stubs for book catalog API
tasks.register('catalogApi', GenerateTask) {
    generatorName = "spring"
    inputSpec = "$rootDir/src/main/resources/api/book-catalog-api.yaml"
    outputDir = layout.buildDirectory.dir("generated/openapi").get().asFile.path
    apiPackage = "com.example.bookmgmt.api.catalog"
    modelPackage = "com.example.bookmgmt.model.catalog"
    configOptions = [
            delegatePattern: "true",
            interfaceOnly  : "true",
            useSpringBoot3 : "true",
            useTags        : "true"
    ]
}

// Generate server stubs for book inventory API
tasks.register('inventoryApi', GenerateTask) {
    generatorName = "spring"
    inputSpec = "$rootDir/src/main/resources/api/book-inventory-api.yaml"
    outputDir = layout.buildDirectory.dir("generated/openapi").get().asFile.path
    apiPackage = "com.example.bookmgmt.api.inventory"
    modelPackage = "com.example.bookmgmt.model.inventory"
    configOptions = [
            delegatePattern: "true",
            interfaceOnly  : "true",
            useSpringBoot3 : "true",
            useTags        : "true"
    ]
}

// Make sure the generated sources are included in the source sets
sourceSets {
    main {
        java {
            srcDir layout.buildDirectory.dir("generated/openapi/src/main/java").get().asFile.path
        }
    }
}

// Make sure the code is generated before compiling
compileJava.dependsOn tasks.inventoryApi
compileJava.dependsOn tasks.catalogApi
```

3. Run `gradle build`

The full source of the initial example is available here -- 
[github](https://github.com/andts/openapi-multi-spec-build-example/tree/7818c8028647b06877565592f60b1767920895e5). 
It has a bit more code to be a fully functional app.

With this configuration, each time we run `gradle build` the OpenAPI generator tasks will execute, even if the specs didn't change.
Even if nothing has changed at all.
```
❯ gradle build

> Task :catalogApi
################################################################################
# Thanks for using OpenAPI Generator.                                          #
# Please consider donation to help us maintain this project 🙏                 #
# https://opencollective.com/openapi_generator/donate                          #
################################################################################
Successfully generated code to /***/build/generated/openapi

> Task :inventoryApi
################################################################################
# Thanks for using OpenAPI Generator.                                          #
# Please consider donation to help us maintain this project 🙏                 #
# https://opencollective.com/openapi_generator/donate                          #
################################################################################
Successfully generated code to /***/build/generated/openapi

> Task :compileJava
> Task :processResources UP-TO-DATE
> Task :classes
> Task :resolveMainClassName UP-TO-DATE
> Task :bootJar UP-TO-DATE
> Task :jar UP-TO-DATE
> Task :assemble UP-TO-DATE
> Task :compileTestJava NO-SOURCE
> Task :processTestResources NO-SOURCE
> Task :testClasses UP-TO-DATE
> Task :test NO-SOURCE
> Task :check UP-TO-DATE
> Task :build UP-TO-DATE

BUILD SUCCESSFUL in 1s
7 actionable tasks: 3 executed, 4 up-to-date
```

Let's try to fix that.

# The Crutch 🩼

A logical solution that comes to mind is to disable generation of those project files. 
Unfortunately, all I could find are issues like this one -- [[REQ] Option to generate only the models/controllers](https://github.com/OpenAPITools/openapi-generator/issues/20538)

Instead, we will do two things:
1. exclude as much project files as possible from generation, and 
2. extract each API into its own folder. 

Technically, you could only do the second, and it would solve the issue, 
but I wanted to keep the sources as clean as possible.

## Cleanup 🧹

The only working way to remove unnecessary project files happens to be through `.openapi-generator-ignore`. 
Put a file like this into the root (or wherever you like):
```
*
**/*
!**src/main/java/**/*
```

This file excludes from generation all files that are not in the source directory. Exactly what we needed.

But there is one issue still, which requires the second step. 
There are two metadata files added in `.openapi-generator` folder: `FILES` and `VERSION`. 
They contain a list of all generated files, and version of the generator accordingly. 
And they are not excluded by the ignore file :( 

<img src="/img/shorts/openapi/meta-files.png">

So now when several tasks generate sources -- each will overwrite those files from the previous. 

Let's continue.

## Extraction 🪏

The only option we have left is to put every API interface into its own folder.  
We will change the task configuration a bit, and will write some more config to attach the sources to the `sourceSets`.  
Significant changes are highlighted.

```gradle,hl_lines=5 8 19-28
// Generate server stubs for book catalog API
tasks.register('catalogApi', GenerateTask) {
    generatorName = "spring"
    inputSpec = "$rootDir/src/main/resources/api/book-catalog-api.yaml"
    outputDir = layout.buildDirectory.dir("generated/openapi/$name").get().asFile.path
    apiPackage = "com.example.bookmgmt.api.catalog"
    modelPackage = "com.example.bookmgmt.model.catalog"
    ignoreFileOverride = "$rootDir/.openapi-generator-ignore"
    configOptions = [
            delegatePattern: "true",
            interfaceOnly  : "true",
            useSpringBoot3 : "true",
            useTags        : "true"
    ]
}

//rest of the code ...

// Disable the default openApiGenerate task as we use our specific tasks
tasks.named('openApiGenerate') {
    enabled = false
}

// Configure our specific OpenAPI generator tasks to be dependencies of compileJava
tasks.withType(GenerateTask).all { task ->
    compileJava.dependsOn task
    sourceSets.main.java.srcDirs +=
        layout.buildDirectory.dir("generated/openapi/${task.name}/src/main/java").get().asFile.path
}

```

Another benefit of this approach is it's automated, so you don't have to add 
the dependencies for every new generator task by hand.

{{ admonition(type="note", text="Note we had to disable the default task from the plugin, 
so it wouldn't mix with our custom generator tasks. Otherwise the build wouldn't work.") }}



# Result


This is what we get for the first clean build:
```
❯ gradle build

> Task :catalogApi
################################################################################
# Thanks for using OpenAPI Generator.                                          #
# Please consider donation to help us maintain this project 🙏                 #
# https://opencollective.com/openapi_generator/donate                          #
################################################################################
Successfully generated code to /Users/andts/Personal/openapi-gen-post/build/generated/openapi/catalogApi

> Task :inventoryApi
################################################################################
# Thanks for using OpenAPI Generator.                                          #
# Please consider donation to help us maintain this project 🙏                 #
# https://opencollective.com/openapi_generator/donate                          #
################################################################################
Successfully generated code to /Users/andts/Personal/openapi-gen-post/build/generated/openapi/inventoryApi

> Task :openApiGenerate SKIPPED
> Task :compileJava
> Task :processResources
> Task :classes
> Task :resolveMainClassName
> Task :bootJar
> Task :jar
> Task :assemble
> Task :compileTestJava NO-SOURCE
> Task :processTestResources NO-SOURCE
> Task :testClasses UP-TO-DATE
> Task :test NO-SOURCE
> Task :check UP-TO-DATE
> Task :build

BUILD SUCCESSFUL in 1s
7 actionable tasks: 7 executed
```

And this is the process when only the application code has changed, but not the specs:
```
❯ gradle build

> Task :catalogApi UP-TO-DATE
> Task :inventoryApi UP-TO-DATE
> Task :openApiGenerate SKIPPED
> Task :compileJava
> Task :processResources UP-TO-DATE
> Task :classes
> Task :resolveMainClassName
> Task :bootJar
> Task :jar
> Task :assemble
> Task :compileTestJava NO-SOURCE
> Task :processTestResources NO-SOURCE
> Task :testClasses UP-TO-DATE
> Task :test NO-SOURCE
> Task :check UP-TO-DATE
> Task :build

BUILD SUCCESSFUL in 1s
7 actionable tasks: 4 executed, 3 up-to-date
```
As you can see, Gradle will correctly detect that the generated code is `UP-TO-DATE` and will skip the tasks.

Consider this approach if you encounter similar issues in your projects.   
This solution is particularly useful if you:
- build with Gradle
- several OpenAPI specs
- you build often

The example code is available here: [github](https://github.com/andts/openapi-multi-spec-build-example). 
You can see the changes for the steps of this post in the commit history. 