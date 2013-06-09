---
layout: post
title: "Manage Leiningen Project Configuration"
date: 2013-04-30 16:16
comments: true
categories: Notes
tags: [clojure]
published: false
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
│   └── database.clj
└── src
    └── test_project
        └── core.clj
```

