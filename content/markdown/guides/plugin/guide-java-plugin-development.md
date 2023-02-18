title: Guide to Developing Java Plugins
author: Bob Allison, Vincent Siveton, Olivier Lamy
date: 2013-01-02

<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

# Introduction

This guide is intended to assist users in developing Java plugins for Maven.

<!-- TODO replace by TOC macro after https://issues.apache.org/jira/browse/DOXIA-696 -->

- [Important Notice: Plugin Naming Convention and Apache Maven Trademark](#important-notice-plugin-naming-convention-and-apache-maven-trade)
- [Your First Plugin](#your-first-plugin)
  - [Your First Mojo](#your-first-mojo)
  - [A Simple Mojo](#a-simple-mojo)
  - [Project Definition](#project-definition)
  - [Building a Plugin](#building-a-plugin)
- [Executing Your First Mojo](#executing-your-first-mojo)
  - [Shortening the Command Line](#shortening-the-command-line)
  - [Attaching the Mojo to the Build Lifecycle](#attaching-the-mojo-to-the-build-lifecycle)
- [Mojo archetype](#mojo-archetype)
- [Parameters](#parameters)
  - [Defining Parameters Within a Mojo](#defining-parameters-within-a-mojo)
  - [Configuring Parameters in a Project](#configuring-parameters-in-a-project)
- [Using Setters](#using-setters)
- [Resources](#resources)

## Important Notice: [Plugin Naming Convention and Apache Maven Trademark](../mini/guide-naming-conventions.html)

You will typically name your plugin `<yourplugin>-maven-plugin`.

Calling it `maven-<yourplugin>-plugin` (note "Maven" is at the beginning of the plugin name)
is **strongly discouraged** since it's a **reserved naming pattern for official Apache Maven plugins maintained by the Apache Maven team** 
with groupId `org.apache.maven.plugins`.

Using this naming pattern is an infringement of the Apache Maven Trademark.

## Your First Plugin

In this section we will build a simple plugin with one goal which takes no parameters and simply displays a message on the screen when run.
Along the way, we will cover the basics of setting up a project to create a plugin, the minimal contents of a Java mojo which will define goal code,
and a couple ways to execute the mojo.

### Your First Mojo

At its simplest, a Java mojo consists simply of a single class representing one plugin's goal. 
There is no requirement for multiple classes like EJBs, although a plugin which contains a number of similar mojos is likely to use an abstract superclass 
for the mojos to consolidate code common to all mojos.

When processing the source tree to find mojos, [`plugin-tools`](/plugin-tools/) looks for classes with either `@Mojo` annotation. 
Any class with this annotation are included in the plugin configuration file.

### A Simple Mojo

Listed below is a simple mojo class which has no parameters. This is about as simple as a mojo can be. 
After the listing is a description of the various parts of the source.

```
package sample.plugin;

import org.apache.maven.plugin.AbstractMojo;
import org.apache.maven.plugin.MojoExecutionException;
import org.apache.maven.plugins.annotations.Mojo;

/**
 * Says "Hi" to the user.
 *
 */
@Mojo(name = "sayhi")
public class GreetingMojo extends AbstractMojo {

    public void execute() throws MojoExecutionException {
        getLog().info( "Hello, world." );
    }
}
```

- The class `org.apache.maven.plugin.AbstractMojo` provides most of the infrastructure required to implement a mojo except for the `execute` method.
- The annotation "`@Mojo`" is required and control how and when the mojo is executed.
- The `execute` method can throw an exception:
  - `org.apache.maven.plugin.MojoExecutionException` if an unexpected problem occurs. Throwing this exception causes a "BUILD FAILURE" message to be displayed.
- The `getLog` method (defined in `AbstractMojo`) returns a log4j-like logger object which allows plugins to create messages at levels of "debug", "info", "warn", and "error".
  This logger is the accepted means to display information to the user. Please have a look at the section [Retrieving the Mojo Logger](../../plugin-developers/common-bugs.html#retrieving-the-mojo-logger) for a hint on its proper usage.

All Mojo annotations are described by the [Mojo API Specification](../../developers/mojo-api-specification.html#The_Descriptor_and_Annotations).

### Project Definition

Once the mojos have been written for the plugin, it is time to build the plugin. To do this properly, the project's descriptor needs to have a number of settings set properly:

|                |                                                                                                             |
|----------------|-------------------------------------------------------------------------------------------------------------|
| `groupId`      | This is the group ID for the plugin, and should match the common prefix to the packages used by the mojos   |
| `artifactId`   | This is the name of the plugin                                                                              |
| `version`      | This is the version of the plugin                                                                           |
| `packaging`    | This must be set to "`maven-plugin`"                                                                        |
| `dependencies` | A dependency must be declared to the Maven Plugin Tools API to resolve "`AbstractMojo`" and related classes |

Listed below is an illustration of the sample mojo project's pom with the parameters set as described in the above table:

```
<project>
  <modelVersion>4.0.0</modelVersion>

  <groupId>sample.plugin</groupId>
  <artifactId>hello-maven-plugin</artifactId>
  <version>1.0-SNAPSHOT</version>
  <packaging>maven-plugin</packaging>

  <name>Sample Parameter-less Maven Plugin</name>
  
  <properties>
    <maven-plugin-tools.version>3.7.1</maven-plugin-tools.version>
  </properties>
  
  <dependencies>
    <dependency>
      <groupId>org.apache.maven</groupId>
      <artifactId>maven-plugin-api</artifactId>
      <version>3.0</version>
      <scope>provided</scope>
    </dependency>

    <!-- dependencies to annotations -->
    <dependency>
      <groupId>org.apache.maven.plugin-tools</groupId>
      <artifactId>maven-plugin-annotations</artifactId>
      <version>${maven-plugin-tools.version}</version>
      <scope>provided</scope>
    </dependency>
  </dependencies>
  
  <build>
    <pluginManagement>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-plugin-plugin</artifactId>
        <version>${maven-plugin-tools.version}</version>
        <executions>
          <execution>
            <id>help-mojo</id>
            <goals>
              <!-- good practice is to generate help mojo for plugin -->
              <goal>helpmojo</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
    </pluginManagement>
  </build>
</project>
```

### Building a Plugin

There are few plugins goals bound to the standard build lifecycle defined with the `maven-plugin` packaging:

|                   |                                                                                           |
|-------------------|-------------------------------------------------------------------------------------------|
| `compile`         | Compiles the Java code for the plugin                                                     |
| `process-classes` | Extracts data to build the [plugin descriptor](/ref/current/maven-plugin-api/plugin.html) |
| `test`            | Runs the plugin's unit tests                                                              |
| `package`         | Builds the plugin jar                                                                     |
| `install`         | Installs the plugin jar in the local repository                                           |
| `deploy`          | Deploys the plugin jar to the remote repository                                           |

For more details, you can look at [detailed bindings for `maven-plugin` packaging](/ref/current/maven-core/default-bindings.html#Plugin_bindings_for_maven-plugin_packaging).

## Executing Your First Mojo

The most direct means of executing your new plugin is to specify the plugin goal directly on the command line. 
To do this, you need to configure the `hello-maven-plugin` plugin in you project:

```
<project>
...
  <build>
    <pluginManagement>
      <plugins>
        <plugin>
          <groupId>sample.plugin</groupId>
          <artifactId>hello-maven-plugin</artifactId>
          <version>1.0-SNAPSHOT</version>
        </plugin>
      </plugins>
    </pluginManagement>
  </build>
...
</project>
```

And, you need to specify a fully-qualified goal in the form of:

```
mvn groupId:artifactId:version:goal
```

For example, to run the simple mojo in the sample plugin, you would enter "`mvn sample.plugin:hello-maven-plugin:1.0-SNAPSHOT:sayhi`" on the command line.

**Tips**: `version` is not required to run a standalone goal.

### Shortening the Command Line

There are several ways to reduce the amount of required typing:

- If you need to run the latest version of a plugin installed in your local repository, you can omit its version number. 
  So just use "`mvn sample.plugin:hello-maven-plugin:sayhi`" to run your plugin.
- You can assign a shortened prefix to your plugin, such as `mvn hello:sayhi`. 
  This is done automatically if you follow the convention of using `${prefix}-maven-plugin` 
  (or `maven-${prefix}-plugin` if the plugin is part of the Apache Maven project).
  You may also assign one through additional configuration - for more information see [ Introduction to Plugin Prefix Mapping](../introduction/introduction-to-plugin-prefix-mapping.html).
- Finally, you can also add your plugin's groupId to the list of groupIds searched by default.
  To do this, you need to add the following to your `${user.home}/.m2/settings.xml` file:

```
<pluginGroups>
  <pluginGroup>sample.plugin</pluginGroup>
</pluginGroups>
```

At this point, you can run the mojo with `mvn hello:sayhi`.

### Attaching the Mojo to the Build Lifecycle

You can also configure your plugin to attach specific goals to a particular phase of the build lifecycle. Here is an example:

```
  <build>
    <pluginManagement>
      <plugins>
        <plugin>
          <groupId>sample.plugin</groupId>
          <artifactId>hello-maven-plugin</artifactId>
          <version>1.0-SNAPSHOT</version>
        </plugin>
      </plugins>
    </pluginManagement>  
    <plugins>
      <plugin>
        <groupId>sample.plugin</groupId>
        <artifactId>hello-maven-plugin</artifactId>
        <executions>
          <execution>
            <phase>compile</phase>
            <goals>
              <goal>sayhi</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
```

This causes the simple mojo to be executed whenever Java code is compiled. For more information on binding a mojo to phases in the lifecycle, please refer to the [Build Lifecycle](../introduction/introduction-to-the-lifecycle.html) document.

## Mojo archetype

To create a new plugin project, you could using the Mojo [archetype](../introduction/introduction-to-archetypes.html) with the following command line:

```
mvn archetype:generate \
  -DgroupId=sample.plugin \
  -DartifactId=hello-maven-plugin \
  -DarchetypeGroupId=org.apache.maven.archetypes \
  -DarchetypeArtifactId=maven-archetype-plugin
```

## Parameters

It is unlikely that a mojo will be very useful without parameters. Parameters provide a few very important functions:

- It provides hooks to allow the user to adjust the operation of the plugin to suit their needs.
- It provides a means to easily extract the value of elements from the POM without the need to navigate the objects.

### Defining Parameters Within a Mojo

Defining a parameter is as simple as creating an instance variable in the mojo and adding the proper annotations.
Listed below is an example of a parameter for the simple mojo:

```
    /**
     * The greeting to display.
     */
    @Parameter( property = "sayhi.greeting", defaultValue = "Hello World!" )
    private String greeting;
```

The portion before the annotations is the description of the parameter. 

The `@Parameter` annotation identifies the variable as a mojo parameter. 

The `defaultValue` parameter of the annotation defines the default value for the variable. 
This value can include expressions which reference the project, such as "`${project.version}`" 
(more can be found in the ["Parameter Expressions" document](/ref/current/maven-core/apidocs/org/apache/maven/plugin/PluginParameterExpressionEvaluator.html)).

The `property` parameter can be used to allow configuration of the mojo parameter from the command line by referencing a property that the user sets via the `-D` option.

### Configuring Parameters in a Project

Configuring the parameter values for a plugin is done in a Maven project within the `pom.xml` file as part of defining the plugin in the project.
An example of configuring a plugin:

```
<plugin>
  <groupId>sample.plugin</groupId>
  <artifactId>hello-maven-plugin</artifactId>
  <version>1.0-SNAPSHOT</version>
  <configuration>
    <greeting>Welcome</greeting>
  </configuration>
</plugin>
```

In the configuration section, the element name ("`greeting`") is the parameter name and the contents of the element ("`Welcome`") is the value to be assigned to the parameter.

**Note**: More details can be found in the [Guide to Configuring Plugins](../mini/guide-configuring-plugins.html).

## Using Setters

You are not restricted to using private field mapping which is good if you are trying to make you Mojos reusable outside the context of Maven.

Using the example above we could define public setters methods that the configuration mapping mechanism can use.

You can also add `@Parameter` annotation on setter method (from version 3.7.0 of `plugin-tools`)

So our Mojo would look like the following:

```

public class MyQueryMojo extends AbstractMojo {

    // provide name for non matching field and setter name
    @Parameter(name="url", property="url")
    private String _url;

    @Parameter(property="timeout")
    private int timeout;

    private String option0;
    private String option1;

    public void setUrl(String url) {
        _url = url;
    }

    public void setTimeout(int timeout) {
        this.timeout = timeout;
    }

    @Parameter(property="options")
    public void setOptions(String[] options) {
        // we can do something more with provided parameter
        this.option0 = options[0];
        this.option1 = options[1];
    }

    public void execute() throws MojoExecutionException {
        ...
    }
}

```

Note the specification of the property name for each parameter which tells Maven 
what setter and getter to use when the field's name does not match the intended name of the parameter in the plugin configuration.

## Resources

1. [Mojo Documentation](../../developers/mojo-api-specification.html): Mojo API, Mojo annotations
2. [Maven Plugin Testing Harness](/shared/maven-plugin-testing-harness/): Testing framework for your Mojos.
3. [Plexus](https://codehaus-plexus.github.io/): The IoC container used by Maven.
4. [Plexus Common Utilities](https://codehaus-plexus.github.io/plexus-utils/): Set of utilities classes useful for Mojo development.
5. [Commons IO](http://commons.apache.org/io/): Set of utilities classes useful for file/path handling.
6. [Common Bugs and Pitfalls](../../plugin-developers/common-bugs.html): Overview of problematic coding patterns.


