---
layout: post
title: "Ansible FAQ"
date: 2013-06-11 21:18
comments: true
categories: Translation
tags: [ops]
published: false
---

本文是从原Ansible官网的FAQ页面翻译而来，网站改版后该页面已无法访问，但可以从[Github历史提交](https://github.com/ansible/ansible.github.com/blob/4a2bf7f60a020f0d0a7b042056fc3dd8716588f2/faq.html)中获得。翻译这篇原始FAQ文档是因为它陈述了Ansible这款工具诞生的原因，设计思路和特性，以及与Puppet、Fabric等同类软件的比较，可以让我们对Ansible有一个整体的了解，所以值得使用者一读。

## 目录

* 为什么命名为“Ansible”？
* Ansible受到了谁的启发？
* 与同类软件比较
    * Func？
    * Puppet？
    * Chef？
    * Capistrano/Fabric？
* 其它问题
    * Ansible的安全性如何？
    * Ansible如何扩展？
    * 是否支持SSH以外的协议？
    * Ansible的适用场景有哪些？

## 为什么命名为“Ansible”？

我最喜爱的书籍之一是奥森·斯科特·卡特的《安德的游戏》。在这本书中，“Ansible”是一种能够跨越时空的即时通讯工具。强烈推荐这本书！

## Ansible受到了谁的启发？

我在Red Hat任职期间主要开发Cobbler，很快我和几个同事就发现在部署工具（Cobbler）和配置管理工具（cfengine、Puppet等）之间有一个空缺，即如何更高效地执行临时性的任务。虽然当时有一些并行调用SSH脚本的方案，但并没有形成统一的API。所以我们（Adrian Likins、Seth Vidal、我）就开发了一个SSH分布式脚本框架——Func。

我一直想在Func的基础上开发一个配置管理工具，但因为忙于Cobbler和其他项目的开发，一直没有动手。在此期间，John Eckersberg开发了名为Taboot的自动化部署工具，它基于Func，采用YAML描述，和目前Ansible中的Playbooks很像。

近期我在一家新公司尝试引入Func，但遇到一些SSL和DNS方面的问题，所以想要开发一个更为简单的工具，吸收Func中优秀的理念，并与我在Puppet Labs的工作经验相结合。我希望这一工具能够易于学习，且不需要进行任何安装步骤。使用它不需要引入一整套新的理论，像Puppet和Chef那样，从而降低被某些运维团队排挤的可能。

我也曾参与过一些大型网站的应用部署，发觉现有的配置管理工具都太过复杂了，超过了这些公司的需求。程序发布的过程很繁复，需要一个简单的工具来帮助开发和运维人员。我不想教授他们Puppet或Chef，而且他们也不愿学习这些工具。

于是我便思考，应用程序的部署就应该那么复杂吗？答案是否定的。

我是否能开发一款工具，让运维人员能够在15分钟内学会使用，并用自己熟悉的语言来扩展它？这就是Ansible的由来。运维人员对自己的服务器设施最清楚，Ansible深知这一点，并将同类工具中最核心的功能提取出来，供我们使用。

Ansible不仅易于学习和扩展，它更是集配置管理、应用部署、临时任务等功能于一身。它非常强大，甚至前所未有。

我很想知道你对Ansible的看法，到邮件列表里发表一下意见吧。

## 与同类软件比较

### Func？

Ansible uses SSH by default instead of SSL and custom daemons, and requires no extra software to run on managed machines. You can also write modules in any language as long as they return JSON. Ansible’s API, of course, is heavily inspired by Func. Ansible also adds a configuration management and multinode orchestration layer (Playbooks) that Func didn’t have.

### Puppet？

First off, Ansible wouldn’t have happened without Puppet. Puppet took configuration management ideas from cfengine and made them sane. However, I still think they can be much simpler.

Ansible playbooks ARE a complete configuration management system. Unlike Puppet, playbooks are implicitly ordered (more like Chef), but still retain the ability to signal notification events (like Puppet). This is kind of a ‘best of both worlds’ thing.

There is no central server subject to thundering herd problems, and Ansible is also designed with multi-node deployment in mind from day-one – something that is difficult for Puppet because of the pull architecture. Ansible is push based, so you can do things in an ordered fashion, addressing batches of servers at one time, and you do not have to contend with the dependency graph. It’s also extensible in any language and the source is designed so that you don’t have to be an expert programmer to submit a patch.

Ansible’s resources are heavily inspired by Puppet, with the “state” keyword being a more or less direct port of “ensure” from Puppet. Unlike Puppet, Ansible can be extended in any language, even bash ... just return some output in JSON format. You don’t need to know Ruby.

Unlike Puppet, hosts are taken out of playbooks when they have a failure. It encourages ‘fail first’, so you can correct the error, instead of configuring as much of the system as it can. A system shouldn’t be half correct, especially if we’re planning on configuring other systems that depend on that system.

Ansible also has a VERY short learning curve – but it also has less language constructs and does not create its own programming language. What constructs Ansible does have should be enough to cover 80% or so of the cases of most Puppet users, and it should scale equally well (not having a server is almost like cheating).

Ansible does support gathering variables from ‘facter’, if installed, and Ansible templates in jinja2 in a way just like Puppet does with erb. Ansible also has it’s own facts though, so usage of facter is not required to get variables about the system.

### Chef？

Much in the ways Ansible is different from Puppet. Chef is notoriously hard to set up on the server, and requires that you know how to program in Ruby to use the language. As such, it seems to have a pretty good following mainly among Rails coders.

Like Chef (and unlike Puppet), Ansible executes configuration tasks in the order given, rather than having to manually specify a dependency graph. Ansible extends this though, by allowing triggered notifiers, so Apache can, be restarted if needed, only once, at the end of a configuration run.

Unlike Chef, Ansible’s playbooks are not a programming language. This means that you can parse Ansible’s playbooks and treat the instructions as data. It also means working on your infrastructure is not a development task and testing is easier.

Ansible can be used regardless of your programming language experience. Both Chef and Puppet are around 60k+ lines of code, while Ansible is a much simpler program. I believe this strongly leads to more reliable software and a richer open source community – the code is kept simple so it is easy for anyone to submit a patch or module.

Ansible does support gathering variables from ‘ohai’, if installed. Ansible also has it’s own facts so you do not need to use ohai unless you want to.

### Capistrano/Fabric？

These tools aren’t really well suited to doing idempotent configuration and are typically about pushing software out for web deployment and automating steps.

Meanwhile Ansible is designed for other types of configuration management, and contains some advanced scaling features.

The ansible playbook syntax is documented within one HTML page and also has a MUCH lower learning curve. And because Ansible is designed for more than pushing webapps, it’s more generally useful for sysadmins (not just web developers), and can also be used for firing off ad-hoc tasks.

## 其它问题

### Ansible的安全性如何？

Ansible aims to not develop custom daemon or PKI code but rely heavily on OpenSSH, which is extremely well peer reviewed and the most widely used security subsystem in the industry. As a result, Ansible has a lower attack surface than any configuration management tool featuring daemons that run as root, and you do not have to worry about network security vulnerabilities in the tool itself.

If your central server is taken over (or even logged into by a malicious employee), provided you were using SSH-agent and encrypted keys (and/or sudo with a password), your keys are still locked and no one can take control of your nodes.

Compared with something like Chef/Puppet/other, compromised manifests would lead to a loss of the whole network, with your network turning into an easily controllable botnet. Further by not running daemon infrastructure, you have more free RAM and compute resources, which should be relevant to users wanting to maximize their computing investments.

### Ansible如何扩展？

Whether in single-execution mode or using ansible playbooks, ansible can run multiple commands in seperate parallel forks, thanks to the magic behind Python’s multiprocessing module.

You can decide if you want to try to manage 5 hosts at a time, or 50 at a time. It’s up to you and how much power you can throw at it and how fast you want to go.

There are no daemons so it’s entirely up to you. When you are aren’t using Ansible, it is not consuming any resources, and you don’t have to contend with a herd of machines all knocking at the door of your management server all at once.

The SSH connection type (paramiko is the default, binary openssh is an option) can also make use of “ControlMaster” features in SSH, which reuses network connections.

If you have 10,000 systems, running a single ansible playbook against all of them probably isn’t appropriate, which is why ansible-pull exists. This tool is designed for running out of git and cron, and can scale to any number of hosts. Ansible-pull uses local connections versus SSH, but can be easily bootstrapped or reconfigured just using SSH. There is more information available about this in the Advanced Playbooks section. The self-bootstrapping and ease of use are ansible are still retained, even when switching to the pull model.

If you’d like to discuss scaling strategies further, please hop on the mailing list.

### 是否支持SSH以外的协议？

Currently SSH (you can choose between paramiko or the openssh binaries) and local connections are supported. The interface is actually pluggable so a small patch could bring transport over message bus or XMPP as an option.

Stop by the mailing list if you have ideas. The connection-specific parts of Ansible are all abstracted away from the core implementation so it is very easy to extend.

### Ansible的适用场景有哪些？

One of the best use cases? Complex multi-node cloud deployments using playbooks. Another good example is for configuration management where you are starting from a clean OS with no extra software installed, adopting systems that are already deployed.

Ansible is also great for running ad-hoc tasks across a wide variety of Linux, Unix, and BSDs. Because it just uses the basic tools available on the system, it is exceptionally cross platform without needing to install management packages on each node.

It also excels for writing distributed scripts and ad-hoc applications that need to gather data or perform arbitrary tasks – whether for a QA sytem, build system, or anything you can think of.
