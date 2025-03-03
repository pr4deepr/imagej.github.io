---
title: Maven
section: Extend:Development:Tools
---

{% include notice icon="info" content='If Maven is completely new to you, read:

-   [What is Maven?](https://maven.apache.org/what-is-maven.html)
-   [Maven in 5 Minutes](https://maven.apache.org/guides/getting-started/maven-in-five-minutes.html)' %}


{% capture maven-caption %}
**Apache Maven** is a {% include wikipedia title='Convention over configuration' text='convention over configuration' %} build automation tool.
{% endcapture %}
{% include img src='icons/maven' align='right' width=150 caption=maven-caption %}

[ImageJ](/software/imagej), [ImageJ2](/software/imagej2), [Fiji](/software/fiji), and other [SciJava](/libs/scijava)-based projects use [Maven](https://maven.apache.org/) for their project infrastructure.

Maven artifacts are published to [Maven Central](https://search.maven.org/) or to the [SciJava Maven repository](/develop/project-management#maven).

# Why do we use Maven?

-   We need something to take source code and package it into a useable format, *i.e.* jar files.
-   Maven organizes dependencies for us, declaring those dependencies... as opposed to manually tracking each individual piece-of-the-puzzle.
-   Maven is a central storage location for all developers, providing the same tools to many folks.
-   Maven has an established concept of immutable release and development versioning – both of which are essential for reproducible science.

# Introduction

Maven is a powerful tool to build Java projects and to manage their dependencies. It can build dependencies from sources, but if the sources are not available, it will look into Maven repositories from which to download the dependencies.

Example: let's assume that you want to build a new plugin for [ImageJ](/software/imagej) that builds on, say, the [3D Viewer](/plugins/3d-viewer) and commons-math. You do not want to rebuild them from scratch unless you need to debug issues that are suspect bugs in said components. This is where Maven comes in: you tell it that the dependencies are ImageJ, 3D Viewer and commons-math and what version(s) you require. It is Maven's job to find and get them, no matter whether you just built them locally or not.

Many convenient [IDEs](/develop/ides) (integrated development environments) including [Eclipse](/develop/eclipse), [NetBeans](/develop/netbeans) and [IntelliJ](/develop/intellij) support Maven projects; therefore, using Maven is an excellent choice when trying to let every developer choose their preferred development environment.

# What does it take to make a new Maven project?

## POM and directory structure

All it really takes is a `pom.xml` file and a certain directory structure:

```xml
pom.xml
src/
   main/
       java/
           <package>/
                    <name>.java
                    ...
       resources/
                    <other-files>
                    ...
```

Technically, you can override the default directory layout in the `pom.xml`, but why do so? It only breaks expectations and is more hassle than it is worth, really.

So the directory structure is: you put your .java files under `src/main/java/` and the other files you need to be included into `src/main/resources/`. Should you want to apply the best practices called "regression tests" or even "test-driven development": your tests' `.java` files go to `src/test/java/` and the non-`.java` files you might require unsurprisingly go into `src/test/resources/`.

So what does a `pom.xml` look like? This is a very simple example:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
	http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>org.mywebsite</groupId>
  <artifactId>my-uber-library</artifactId>
  <version>2.0.0-SNAPSHOT</version>
</project>
```
The first 6 lines are of course just a way to say "Hi, Maven? How are you today? This is what I would like you to do...".

The only relevant parts are the `groupId`, which by convention is something like the inverted domain name (similar to the Java package convention), the name of the artifact to build (it will be put into `target/`, under the name `<artifactId>-<version>.jar`). And of course the version.

## Dependencies

Maven is not only a build tool, but also a dependency management tool.

To depend on another library, you must declare the dependencies in your project's `pom.xml` file. For example, every [ImageJ](/software/imagej) plugin will depend on ImageJ. So let's add that (before the final `</project>` line):
```xml
<dependencies>
  <dependency>
	<groupId>net.imagej</groupId>
	<artifactId>ij</artifactId>
	<version>1.45b</version>
  </dependency>
</dependencies>
```
As you can see, dependencies are referenced using the same `groupId`, `artifactId` and `version` triplet (also known as *GAV parameters*) that you declared for your project itself.

## Repositories

Once your dependencies are declared, Maven will download them on demand from the Internet. However, for Maven to find the dependencies, it has to know where to look.

Out of the box, Maven will look in the so-called [Maven Central repository](https://search.maven.org/). Some ImageJ and SciJava components are deployed there, including the [pom-scijava parent POM](/develop/architecture#maven-component-structure) which declares important metadata, such as the [Bill of Materials](/develop/architecture#bill-of-materials): current artifact versions intended to work together.

However, many other SciJava and ImageJ components are not yet deployed to Maven Central, but instead to the [SciJava Maven repository](/develop/project-management#maven). To gain access to this repository from your project, add the following configuration block to your `pom.xml`:

```xml
<repositories>
  <repository>
	<id>scijava.public</id>
	<url>https://maven.scijava.org/content/groups/public</url>
  </repository>
</repositories>
```

As a rule of thumb: components [versioned at 0.x](/develop/versioning) are deployed to the SciJava Maven repository, while those at 1.x or later are deployed to Maven Central.

## Releases and snapshots

There are two different sorts of Maven artifacts (i.e., JAR files): releases and snapshots. The snapshot versions are "in-progress" versions. If you declare a dependency with a `-SNAPSHOT` suffix in the version, Maven will look once a day for new artifacts of the same versions; otherwise, Maven will look whether it has that version already and not bother re-downloading.

## Producing multiple JAR files

So what if you have multiple `.jar` files you want to build in the same project? Then these need to live in their own subdirectories and there needs to be a common parent POM, a so-called *aggregator* or *multi-module* POM (only this POM needs to have the SciJava POM as parent, of course). {% include github org='imagej' repo='tutorials' tag='577286474be8399eb38d30d66cf0c35ee50bd929' path='pom.xml\#L47-L62' label='Here is an example' %}. Basically, it is adding the `<packaging>pom</packaging>` entry at the top, as well as some subdirectory names to the `<modules>` section.

Note, however, that most projects of the [SciJava component collection](/develop/architecture) (e.g., [SciJava](/libs/scijava), [ImgLib2](/libs/imglib2), [SCIFIO](/libs/scifio), [ImageJ2](/software/imagej2) and [Fiji](/software/fiji)) now structure each component as its own single-module project in its own Git repository, since using multi-module projects can complicate versioning.

## Convention over configuration

There are many more things you can do with Maven, but chances are you will not need them.

The simplicity of the `pom.xml` you need comes from the fact that Maven defines implicit defaults. It calls that *convention over configuration*. For many reasons, it is strongly recommended to stay with the defaults as much as possible.

In the context of [SciJava](/libs/scijava), you will most likely never write a `pom.xml` from scratch. You will rather more likely edit an existing one, possibly after having copied it. We recommend using the [ImageJ "Load and Display a Dataset" tutorial](https://github.com/imagej/tutorials/tree/master/maven-projects/load-and-display-dataset) as a starting point.

# How to find a dependency's groupId/artifactId/version (GAV)?

Most popular open source libraries upon which you might want to depend are stored in the [Maven Central repository](https://search.maven.org/). However, the ImageJ and Fiji JARs are not yet stored there, but in the [SciJava Maven repository](/develop/project-management#maven). Fortunately, you can search both at once, by visiting:

[https://maven.scijava.org/](https://maven.scijava.org/)

For example, let's suppose you want to depend on the [snakeyaml](http://snakeyaml.org) library. Typing "snakeyaml" into the search box at [maven.scijava.org](https://maven.scijava.org) tells us to use a `groupId` of `org.yaml`, `artifactId` of `snakeyaml`, with available versions ranging from `1.4` to `1.10`. In the case of many results, you can click the "Drill down" link to view more details of that specific GAV combination. You can also click an entry to get a formatted `dependency` block for direct copy-pasting into your POM.

{% include notice icon="tip" content="If your dependencies are in Maven Central, you can use the [quickdeps](https://github.com/ingenieux/quickdeps) tool to quickly generate dependency blocks, by scanning your project's bytecode." %}

# Depending on libraries outside the core repositories

If you need to depend on a library that is not present in either Maven Central or the SciJava Maven repository, first double check the project's website for any documentation on using their library with Maven. They might provide their own public Maven repository which you could use instead (by [adding a `<repository>` to the `<repositories>` section of your POM](https://maven.apache.org/guides/mini/guide-multiple-repositories.html)).

If there are no public repositories containing your dependency, you have two options:

-   If the dependency is itself an ImageJ plugin, consider [contributing it to Fiji](/contribute/fiji). Plugins distributed with Fiji are [made available as Maven artifacts](/contribute/fiji#maven-artifacts), and thus will benefit both users and developers.

-   If the dependency is narrower in scope, you could [contact the ImageJ & Fiji maintainers](/discuss/mailing-lists) to get your needed dependency added to the SciJava Maven repository. Note that you will then be responsible for distributing the dependency with your code—so ensure it is [licensed appropriately](/licensing).

Finally, for local testing you can [install the dependency into your local Maven repository cache yourself](https://maven.apache.org/guides/mini/guide-3rd-party-jars-local.html). The command is `mvn install:install-file`. For example, if you have a library `foo.jar` to install, you could run:

```shell
mvn install:install-file -Dfile=/path/to/foo.jar -DgroupId=org.foo -DartifactId=foo -Dversion=1.0.0 -Dpackaging=jar
```

For the `groupId`, it is typically best to use the reversed domain name of the library's website. For libraries that are not explicitly versioned, you may want to use a datestamp such as "20120920" for the `version`, rather than inventing your own versioning scheme.

{% include notice icon='warning' content="If you use `install:install-file`, others will not be able to build your code unless they also use `install:install-file` to install the library on their systems." %}

When in doubt, [contact the community](/discuss) with your questions.

# Further reading

-   Our very own [Maven FAQ](/develop/maven-faq)
-   [Maven's Getting Started](https://maven.apache.org/guides/getting-started/)
-   [Maven: The Complete Reference](https://books.sonatype.com/mvnref-book/reference/index.html)
-   [Maven by Example](https://books.sonatype.com/mvnex-book/reference/index.html)
