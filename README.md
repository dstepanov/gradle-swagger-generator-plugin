Gradle Swagger Generator Plugin [![Build Status](https://travis-ci.org/int128/gradle-swagger-generator-plugin.svg?branch=master)](https://travis-ci.org/int128/gradle-swagger-generator-plugin) [![Gradle Status](https://gradleupdate.appspot.com/int128/gradle-swagger-generator-plugin/status.svg)](https://gradleupdate.appspot.com/int128/gradle-swagger-generator-plugin/status)
=============================

This is a Gradle plugin to do following tasks:

- Validate an OpenAPI YAML.
- Generate source from an OpenAPI YAML using [Swagger Codegen](https://github.com/swagger-api/swagger-codegen).
- Generate Swagger UI with an OpenAPI YAML.

See also following examples:

- [Swagger UI generated by the plugin](https://int128.github.io/gradle-swagger-generator-plugin/swagger-ui-petstore/)
- [HTML document generated by the plugin](https://int128.github.io/gradle-swagger-generator-plugin/swagger-html/)


## Code Generation

Create a project with following build script.

```groovy
plugins {
  id 'org.hidetake.swagger.generator' version '2.6.0'
}

repositories {
  jcenter()
}

dependencies {
  // Add dependency for Swagger Codegen CLI
  swaggerCodegen 'io.swagger:swagger-codegen-cli:2.2.3'
}

swaggerSources {
  petstore {
    inputFile = file('petstore.yaml')
    code {
      language = 'spring'
      configFile = file('config.json')
    }
  }
}
```

The task generates source code into `build/swagger-code-petstore`.

```
% ./gradlew generateSwaggerCode
:resolveSwaggerTemplate NO-SOURCE
:generateSwaggerCodePetstore
:generateSwaggerCode NO-SOURCE
```

Run the task with `Help` postfix to show available JSON configuration.

```
% ./gradlew generateSwaggerCodePetstoreHelp
:generateSwaggerCodePetstoreHelp
Available JSON configuration for language spring:

CONFIG OPTIONS
	sortParamsByRequiredFlag
  ...
```


## Swagger UI Generation

Create a project with following build script.

```groovy
plugins {
  id 'org.hidetake.swagger.generator' version '2.6.0'
}

dependencies {
  // Add dependency for Swagger UI
  swaggerUI 'org.webjars:swagger-ui:2.2.6'
}

swaggerSources {
  petstore {
    inputFile = file('petstore.yaml')
    ui {
      options.docExpansion = 'full'
    }
  }
}
```

The task generates an API document as `build/swagger-ui-petstore`.

```
% ./gradlew generateSwaggerUI
:generateSwaggerUIPetstore
:generateSwaggerUI NO-SOURCE
```


## Recipes

There are some examples in [acceptance-test project](acceptance-test).

### Build generated code

It is recommended to generate code into `build` directory and exclude from a Git repository.
We can compile generated code as follows:

```groovy
swaggerSources {
  petstore {
    inputFile = file('petstore.yaml')
    code {
      language = 'spring'
      configFile = file('config.json')
      // Generate only models and controllers
      components = ['models', 'apis']
    }
  }
}

// Configure compile task dependency and source
compileJava.dependsOn swaggerSources.petstore.code
sourceSets.main.java.srcDir swaggerSources.petstore.code.outputDir
```

For more, see [code-generator project](acceptance-test/code-generator) in examples.


### Validate YAML before code genetation

It is recommended to validate an OpenAPI YAML before code generation in order to avoid invalid code generated.
We can validate a YAML as follows:

```groovy
swaggerSources {
  petstore {
    inputFile = file('petstore.yaml')
    code {
      language = 'spring'
      configFile = file('config.json')
      // Validate YAML before code generation
      dependsOn validation
    }
  }
}
```

For more, see [code-generator project](acceptance-test/code-generator) in examples.


### Using custom template

We can use a custom template for the code generation as follows:

```groovy
// build.gradle
swaggerSources {
  inputFile = file('petstore.yaml')
  petstore {
    language = 'spring'
    // Path to the template directory
    templateDir = file('templates/spring-mvc')
  }
}
```

For more, see [custom-template project](acceptance-test/custom-template) in examples.


### Using custom generator class

We can use a custom generator class for the code generation as follows:

```groovy
// build.gradle
swaggerSources {
  petstore {
    inputFile = file('petstore.yaml')
    code {
      // FQCN of the custom generator class
      language = 'CustomGenerator'
    }
  }
}
```

```groovy
// buildSrc/build.gradle
repositories {
  jcenter()
}

dependencies {
  // Add dependency here (do not specify in the parent project)
  compile 'io.swagger:swagger-codegen-cli:2.2.3'
}
```

```groovy
// buildSrc/src/main/groovy/CustomGenerator.groovy
import io.swagger.codegen.languages.SpringCodegen

class CustomGenerator extends SpringCodegen {
}
```

For more, see [custom-class project](acceptance-test/custom-class) in examples.


### Externalize template or generator class

In some large use case, we can release a template or generator to an external repository and use them from projects.

```groovy
// build.gradle
repositories {
  // Use external repository for the template and the generator class
  maven {
    url 'https://example.com/nexus-or-artifactory'
  }
  jcenter()
}

dependencies {
  swaggerCodegen 'io.swagger:swagger-codegen-cli:2.2.3'
  // Add dependency for the template
  swaggerTemplate 'com.example:swagger-templates:1.0.0'
  // Add dependency for the generator class
  swaggerCodegen 'com.example:swagger-generators:1.0.0'
}

swaggerSources {
  petstore {
    inputFile = file('petstore.yaml')
    code {
      language = 'spring'
      // The plugin automatically extracts template JAR into below destination
      templateDir = file("${resolveSwaggerTemplate.destinationDir}/spring-mvc")
    }
  }
}
```

For more, see [externalize-template project](acceptance-test/externalize-template) and
[externalize-class project](acceptance-test/externalize-class) in examples.


### Multiple sources

We can handle multiple sources in a project as follows:

```groovy
// build.gradle
swaggerSources {
    petstoreV1 {
        inputFile = file('v1-petstore.yaml')
        code {
            language = 'spring'
            configFile = file('v1-config.json')
        }
    }
    petstoreV2 {
        inputFile = file('v2-petstore.yaml')
        code {
            language = 'spring'
            configFile = file('v2-config.json')
        }
    }
}

compileJava.dependsOn swaggerSources.petstoreV1.code, swaggerSources.petstoreV2.code
sourceSets.main.java.srcDirs swaggerSources.petstoreV1.code.outputDir, swaggerSources.petstoreV2.code.outputDir
```

For more, see [multiple-sources project](acceptance-test/multiple-sources) in examples.


## Settings

The plugin adds `validateSwagger`, `generateSwaggerCode` and `generateSwaggerUI` tasks.
A task will be skipped if no input file is given.


### Task type `ValidateSwagger`

The task accepts below properties.

Key           | Type              | Value                                   | Default value
--------------|-------------------|-----------------------------------------|--------------
`inputFile`   | File              | Swagger spec file.                      | Mandatory
`reportFile`  | File              | File to write validation report.        | `$buildDir/tmp/validateSwagger/report.yaml`


### Task type `GenerateSwaggerCode`

The task accepts below properties.

Key           | Type              | Value                                   | Default value
--------------|-------------------|-----------------------------------------|--------------
`language`    | String            | Language to generate.                   | Mandatory
`inputFile`   | File              | Swagger spec file.                      | Mandatory
`outputDir`   | File              | Directory to write generated files, wiped before generation. Do not specify the project directory. | `$buildDir/swagger-code`
`library`     | String            | Library type.                           | None
`configFile`  | File              | [JSON configuration file](https://github.com/swagger-api/swagger-codegen#customizing-the-generator). | None
`templateDir` | File              | Directory containing the template.      | None
`components`  | List of Strings   | [Components to generate](https://github.com/swagger-api/swagger-codegen#selective-generation) that is a list of `models`, `apis` and `supportingFiles`. | All components
`additionalProperties` | Map of String, String | [Additional properties](https://github.com/swagger-api/swagger-codegen#to-generate-a-sample-client-library). | None


### Task type `GenerateSwaggerUI`

The task accepts below properties.

Key           | Type              | Value                                   | Default value
--------------|-------------------|-----------------------------------------|--------------
`inputFile`   | File              | Swagger spec file.                      | Mandatory
`outputDir`   | File              | Directory to write Swagger UI files, wiped before generation. Do not specify the project directory. | `$buildDir/swagger-ui`
`options`     | Map of Objects    | [Swagger UI options](https://github.com/swagger-api/swagger-ui#parameters). | Empty map
`header`      | String            | Custom tags before loading Swagger UI.  | None

The plugin replaces the Swagger UI loader with custom one containing following and given `options`:

```json
{
  "url":"",
  "validatorUrl":null,
  "spec":{"swagger":"2.0","info":"... converted from YAML input ..."}
}
```


## Contributions

This is an open source software licensed under the Apache License Version 2.0.
Feel free to open issues or pull requests.

Travis CI builds the plugin continuously.
Following variables should be set.

Environment Variable        | Purpose
----------------------------|--------
`$GRADLE_PUBLISH_KEY`       | Publish the plugin to Gradle Plugins
`$GRADLE_PUBLISH_SECRET`    | Publish the plugin to Gradle Plugins
`$GITHUB_TOKEN`             | Publish the example of API document to GitHub Pages
