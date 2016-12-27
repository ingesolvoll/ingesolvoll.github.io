---
layout: post
title:  "How boot solved our FTP upload problem"
date:   2016-12-26 21:30:02 +0100
---

### Sloooooow

My company uses Maven and Jenkins for most of our build needs. We are pretty happy with the setup, it gets the job done. Except for one thing: FTP uploads. The Wagon plugin with FTP extension is the best option we found for doing this. Check out our setup below:

{% highlight xml %}
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>wagon-maven-plugin</artifactId>
    <version>1.0</version>
    <executions>
        <execution>
            <id>deploy-to-some-ftp</id>
            <phase>install</phase>
            <goals>
                <goal>upload-single</goal>
            </goals>
            <configuration>
                <serverId>server-id-on-jenkins</serverId>
                <url>ftp://myserver.com</url>
                <fromFile>${project.build.directory}/${project.build.finalName}.war</fromFile>
                <toFile>path/on/server/something.war</toFile>
            </configuration>
        </execution>
    </executions>
</plugin>

<extensions>
    <extension>
        <groupId>org.apache.maven.wagon</groupId>
        <artifactId>wagon-ftp</artifactId>
        <version>2.10</version>
    </extension>
</extensions>
{% endhighlight %}

The FTP support in this setup is not great. It uses Apache Commons Net, which is fair enough. But the FTP part of Wagon seems to be a relatively dead project, and needs a quite particular setup of dependencies to work.
