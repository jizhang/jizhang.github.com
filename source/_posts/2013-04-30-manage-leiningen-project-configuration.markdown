---
layout: post
title: "Manage Leiningen Project Configuration"
date: 2013-04-30 16:16
comments: true
categories: Notes
tags: [clojure]
published: false
---

In Maven projects, we tend to use `.properties` files to store various configurations, and use Maven profiles to switch between development and production environments. 

As for Leiningen projects, we can use the same approach. But instead of using `.properties` files, we'll use `.clj` files directly, since it's more expressive and easy to use. 

This article will demonstrate how to utilize `.clj` files as configuration, and switch between different environemtns when testing and packaging Leiningen projects.
