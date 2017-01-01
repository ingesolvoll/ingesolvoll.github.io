---
layout: post
title:  "How boot solved our FTP problem"
date:   2016-12-26 21:30:02 +0100
---

### Limitations of Maven

My company uses Maven and Jenkins for most of our build needs. We are pretty happy with the setup, it gets the job done. Except for the couple of builds that need to do an FTP upload.

The Wagon plugin with FTP extension seems to be your best bet for doing this in Maven. This is our setup:
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

We would have been ok with this amount of XML bloat if it all worked. But it doesn't. Our 50MB artifact upload takes __5-10 minutes__. With desktop FTP software it takes 8 seconds, so clearly this can be improved. Unfortunately, the configuration options for Wagon FTP are nowhere to be found.

### Why so slow?
It took me less than 2 minutes to find the solution on [stackoverflow]. It turns out that FTP upload is extremely inefficient when using default buffer size. A buffer size of 1MB does the trick.

So what to do then, when Wagon has no options? I considered rolling my own Maven plugin for this, but life is too short for that.

### Boot to the rescue
[Boot] is a relatively young build tool in a Clojure world dominated by Leiningen. It has a rather fresh approach by being less declarative and more script-centric than Maven and Leiningen.

With boot you just pull in the plain java/clojure libraries you need. No special plugins or xml/json. You organize your code as tasks that operate on an immutable file system abstraction. Now I'm free to use Apache Commons Net directly to get the job done, but I'd much rather use an idiomatic clojure wrapper. Turns out there are several of those, one of them is [clj-ftp]. Quite a nice API, this should cover most use cases:
 {% highlight clojure %}
 (with-ftp [client "ftp://myserver.com"]
        (.setBufferSize client 1024000)
        (client-put client local-file-path remote-path))
 {% endhighlight %}

Making it available in my boot file is as simple as:
{% highlight clojure %}
(set-env!
  :dependencies '[[com.velisco/clj-ftp "0.3.8"]])

(require '[miner.ftp :refer [with-ftp client-put]])
{% endhighlight %}

As in all other code environments, it is good practice to make as much as possible of the code pure and decoupled from the framework. Since boot is just clojure code, this is easy:

{% highlight clojure %}
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
{% endhighlight %}

Finally, we need to connect our code to boot tasks, so they can be executed by Jenkins. The boot way is through the `deftask` macro. Boot has lots of built in tasks, like `jar` and `compile`, and our newly created upload task fits nicely in a chain with the built-in ones.
{% highlight clojure %}
(deftask upload []
   (with-pass-thru fileset
     (ftp-upload! "ftp://myserver.com" (find-war fileset) "path/on/server/something.war"))))

(deftask build
  (comp (compile) (jar) (uber) (war) (upload)))
{% endhighlight %}

### Full code listing

The full boot build file is listed below. I'm completely new to boot, so I probably made tons of mistakes. But I think this is as compact, powerful and readable as it gets for integrating an arbitrary Java/Clojure library into your build!

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
  (with-ftp [client url
              :file-type :binary]
            (.setBufferSize client 1024000)
            (client-put client local-path remote-path))

(deftask upload []
   (with-pass-thru fileset
     (ftp-upload! "ftp://myserver.com" (find-war fileset) "path/on/server/something.war"))))

(deftask build []
  (comp (compile) (jar) (uber) (war) (upload)))
{% endhighlight %}

[Boot]: http://boot-clj.com/
[clj-ftp]: https://github.com/miner/clj-ftp
[stackoverflow]: http://stackoverflow.com/questions/11572588/speed-up-apache-commons-ftpclient-transfer
