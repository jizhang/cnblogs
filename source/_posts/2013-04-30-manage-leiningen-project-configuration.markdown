---
layout: post
title: "Manage Leiningen Project Configuration"
date: 2013-04-30 16:16
comments: true
categories: Notes
tags: [clojure]
published: true
---

In Maven projects, we tend to use `.properties` files to store various configurations, and use Maven profiles to switch between development and production environments. Like the following example:

```text
# database.properties
mydb.jdbcUrl=${mydb.jdbcUrl}
```

```xml
<!-- pom.xml -->
<profiles>
    <profile>
        <id>development</id>
        <activation><activeByDefault>true</activeByDefault></activation>
        <properties>
            <mydb.jdbcUrl>jdbc:mysql://127.0.0.1:3306/mydb</mydb.jdbcUrl>
        </properties>
    </profile>
    <profile>
        <id>production</id>
        <!-- This profile could be moved to ~/.m2/settings.xml to increase security. -->
        <properties>
            <mydb.jdbcUrl>jdbc:mysql://10.0.2.15:3306/mydb</mydb.jdbcUrl>
        </properties>
    </profile>
</profiles>
```

As for Leiningen projects, there's no variable substitution in profile facility, and although in profiles we could use `:resources` to compact production-wise files into Jar, these files are actually replacing the original ones, instead of being merged. One solution is to strictly seperate environment specific configs from the others, so the replacement will be ok. But here I take another approach, to manually load files from difference locations, and then merge them.

<!-- more -->

## Read Configuration from `.clj` Files

Instead of using `.properties`, we'll use `.clj` files directly, since it's more expressive and Clojure makes it very easy to utilize them. 

```clojure
(defn read-config [section]
  (let [read (fn [res-path]
               (if-let [res (clojure.java.io/resource res-path)]
                 (read-string (slurp res))
                 {}))
        default-name (str (name section) ".clj")
        default (read default-name)
        override (read (str "override/" default-name))]
    (merge default override)))
```

This function assumes the following directory layout:

```text
test-project/
├── README.md
├── project.clj
├── resources
│   ├── database.clj
│   └── override
│       └── database.clj
└── src
    └── test_project
        └── core.clj
```

And the `database.clj`s are like:

```clojure
; resources/database.clj
{:charset "utf-8"
 :mydb {:host "127.0.0.1"}}

; resources/override/database.clj
{:mydb {:host "10.0.2.15"}}
```

The `.clj` files simply contains a `map` object, and we use `read-string` facility to parse the map. Since the latter map is merged into the former one, we can include some default settings without worrying about whether they'll be available.

## Places to Put 'Outter Configuration'

Here I use the word 'outter', which means those configs are related to environments, and will override the default settings. In this section, I'll introduce some typical places to put these outter configs and how to use them.

### A 'resources/override/' Directory

First of all, this directory should be removed from version control, such as `.gitignore`:

```text
/.project
/.settings

/resources/override
```

And then, developers can put production or local configuration files in this directory.

In production, there's typically a 'compiling server', which can be used to store production configs. After compiling, the Jar file will include the proper configs and are ready to be deployed.

### A Dedicated Directory on Every Server

We could simply replace the `override` directory with an absolute path, such as `/home/www/config`. The pros are that we don't need to recompile the jar files when config changes, and some of the configs could be shared between different projects. 

But in such approach, you'll need a provisioning tool like Puppet to manage those configs and notify the applications to restart. For something like Hadoop MapReduce job, it's probably not practical to have such a directory on every compute node.

Another thing I want to mention in this approach is that, I suggest using an environment variable to indicate the path to config directory, not hard-coded in application. As a matter of fact, you could even place all configs into env vars, as suggested by [12-factor apps](http://www.12factor.net/config).

### A Central Configuration Server

As for really big corporations, a central configuration server is necessary. One popular option is to use ZooKeeper. Or your companies have some service-discovery mechanism. These are really advanced topics, and I'll leave them to the readers.

## Manage Configs in Application

Lastly, I'll share a snippet that'll manage the configs, it's actually quite easy:

```clojure
(def ^:private config (atom {}))

(defn get-config

  ([section]
    (if-let [config-section (get @config section)]
      config-section
      (let [config-section (read-config section)]
        (swap! config assoc section config-section)
        config-section)))

  ([section item]
    (get (get-config section) item)))
```
