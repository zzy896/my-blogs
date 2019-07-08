---
title: 如何在Maven项目模块A引用子模块B的war项目的类文件?
date: 2019-07-08 17:36:19
tags:
---
**1.首先在war模块B添加一个maven-jar-plugin，并设置其classifier为jar. **
```bash
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <version>2.4</version>
    <configuration>
        <classifier>jar</classifier>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>jar</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```
**2.然后模块A引用模块B的jar文件 **
```bash
<dependency>
    <groupId>${project.groupId}</groupId>
    <artifactId>B</artifactId>
    <version>${project.version}</version>
    <classifier>jar</classifier>
    <scope>runtime</scope>
</dependency>
```