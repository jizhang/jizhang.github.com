---
layout: post
title: "How 'mysqladmin extended-status' Works?"
date: 2013-04-29 12:18
comments: true
categories: Notes
tags: [mysql]
published: false
---

When setting up the monitoring system for MySQL, it turns out the available scripts are repeatedly calling `mysqladmin extended-status` to get various metrics from the server. So a question comes up - Is this call expensive? How on earth MySQL Server store these statistics and respond the query? Then I dig a little in source code, and straight out a breif 'tour guide' for how this command works.
