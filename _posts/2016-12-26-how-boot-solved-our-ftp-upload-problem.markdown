---
layout: post
title:  "How boot solved our FTP problem"
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
            <id>ftp-deploy</id>
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

There are several problems with this approach:
* That's a huge amount of typing for a simple file upload
* FTP project for Wagon seems pretty dead, hard to get dependencies right
* Not much can be configured

We would have been ok with most of these if it all worked. But it doesn't. Our 50MB artifact upload takes 5-10 minutes. With desktop FTP software it takes 8 seconds, so clearly this can be improved.

### Why so slow?
It took me less than 2 minutes to find the solution on stackoverflow. It turns out that FTP upload is extremely inefficient when using default buffer size. A buffer size of 1MB does the trick.

I considered rolling my own Maven plugin for this, but life is too short for that.

### Boot to the rescue
[Boot] is a relatively young build tool in a Clojure world dominated by Leiningen. It has a rather fresh approach by being less declarative and more script-centric than Maven and Leiningen.

With boot you just pull in the libraries you need. No special plugins or xml/json. You organize your code as tasks that operate on an immutable file system abstraction. Now I'm free to use Apache Commons Net directly to get the job done, but I'd much rather use an idiomatic clojure wrapper. Turns out there are several of those, I chose [clj-ftp]. With clj-ftp, all I need to do is this:

 {% highlight clojure %}
 (with-ftp [client "ftp://myserver.com"]
        (.setBufferSize client 1024000)
        (client-put client local-file-path remote-path))
 {% endhighlight %}

Getting it into my boot file is as simple as:

{% highlight clojure %}
(set-env!
  :dependencies '[[com.velisco/clj-ftp "0.3.8"]])

(require '[miner.ftp :refer [with-ftp client-put]])
{% endhighlight %}

{% highlight clojure %}
(set-env!
  :dependencies '[[com.velisco/clj-ftp "0.3.8"]])

(require '[miner.ftp :refer [with-ftp client-put]])

(defn find-war [fileset]
  (let [{:keys [dir path]} (->> fileset
                                output-files
                                (by-ext [".war"])
                                first)]
         (str (.toString dir) "\\" path)))

(defn ftp-upload! [url local-path remote-path]
  (with-ftp [client url]
            (.setBufferSize client 1024000)
            (client-put client local-path remote-path))

(deftask upload []
   (with-pass-thru fileset
     (ftp-upload! "ftp://myserver.com" (find-war fileset) "path/on/server/something.war"))))

(deftask build
         "Build my project."
         []
         (comp (compile) (jar) (uber) (war) (upload)))

{% endhighlight %}

[Boot]: http://boot-clj.com/
[clj-ftp]: https://github.com/miner/clj-ftp
