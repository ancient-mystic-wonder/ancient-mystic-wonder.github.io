---
layout: post
title:  "Learning about Spring Boot externalized configs...the hard way"
date:   2018-04-17
categories: spring-boot
tags: [spring-boot, java, maven]
---

This past week I have been dealing with Maven's resource filtering feature for our Spring Boot properties files. To give a short description of resource filtering, it is a way of substituting values from any `application*.properties` files (or any file really) with values from the `pom.xml`

For example, I needed a property in my `application.properties` that grabs the current version of the project (which can be found at the `pom.xml`'s `<project><version>...</version><project>` aka `project.version` xml tag. Resource filtering allows us to achieve this by doing the following:

##### application.properties
```
my.application.version=@project.version@
```

##### pom.xml
```
<project>
...
	<version>1.0-SNAPSHOT</version>
...
</project>
```

Note: we use `@...@` instead of `${...}` because Spring Boot. More on that [here][spring-boot-properties-docs]

## The Problem

The problem was, our project exclusively used external configurations (all the `application*.properties` files are found on an outside `config/` directory instead of `src/main/resources/`).

At the time, I did not have an idea how external configurations worked, and foolishly tried to apply resource filtering on the external properties file. As expected, it did not go so well:

Despite adding this line to my `pom.xml` (which should trigger resource filtering for the files specified):
```
<build>
...
	<resource>
		<directory>${basedir}/config</directory>
		<filtering>true</filtering>
		<includes>
		    <include>**/application*.yml</include>
		    <include>**/application*.yaml</include>
		    <include>**/application*.properties</include>
		</includes>
	</resource>
...
</build>
```

Resource filtering was not working for our properties files found on the `config/` directory. Instead, attempts to get the property value for the project version defined earlier will give the literal `@project.version@` string.

To add to the confusion, running `mvn resources:resources` will generate a PROPERLY FILTERED properties file in the `target/` directory. Even checking the generated jar file from `mvn package` will show a properly filtered property file. So why does my project STILL print the literal `'@project.version@'` string instead of giving me `'1.0-SNAPSHOT'` during runtime?

## Explanation

The answer, after looking for 3+ days, lies on the [docs][spring-boot-externalized-config-docs] that I oh so foolishly skimmed through during my first time reading. Turns out, even though I generated a filtered version of the external properties file in the `target/` directory, the since I was running the project on the same directory as `config/`, Spring Boot's search order will prioritize the `config/` directory outside the `target/`, which possesses the unfiltered string.

Also, the idea of attempting resource filtering on externalized configs is flawed in the first place, as resource filtering requires running the project through the build process, and the purpose of externalized configs is the have overriding properties outside of the `jar` file.

Anyways, the moral of the story is: read the docs.

[spring-boot-properties-docs]: https://docs.spring.io/spring-boot/docs/current/reference/html/howto-properties-and-configuration.html#howto-automatic-expansion-maven
[spring-boot-externalized-config-docs]: https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html
