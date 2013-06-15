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

Ansible默认使用SSH，而非SSL和守护进程，无需在远程服务器上安装任何软件。你可以使用任何语言编写插件，只要它能够返回JSON格式即可。Ansible的API深受Func的影响，但它和Func相较提供了配置管理和多节点统一化部署（Playbooks）等功能。

### Puppet？

首先我要强调的是，如果没有Puppet，就不会有Ansible。Puppet从cfengine中吸收了配置管理的概念，并更合理地加以实现。但是，我依旧认为它可以再简单一些。

Ansible的playbook是一套完整的配置管理系统。和Puppet不同，playbook在编写时就隐含了执行顺序（和Chef类似），但同时也提供了事件机制（和Puppet类似），可以说是结合了两者的优点。

Ansible没有中心节点的概念，从而避免了惊群效应。它一开始就是为多节点部署设计的，这点Puppet很难做到，因为它是一种“拉取”的架构。Ansible以“推送”为基础，从而能够定义执行顺序，同时只操作一部分服务器，无需关注它们的依赖关系。又因为Ansible可以用任何语言进行扩展，因此并不是只有专业的程序员才能为其开发插件。

Ansible中资源的概念深受Puppet的启发，甚至“state”这一关键字直接来自Puppet的“ensure”一词。和Puppet不同的是，Ansbile可以用任何语言进行扩展，甚至是Bash，只需返回JSON格式的输出即可。你不需要懂得Ruby。

和Puppet不同，Ansible若在配置某台服务器时发生错误，它会立即终止这台服务器的配置过程。它提倡的是“提前崩溃”，修正错误，而非最大化应用。这一点在我们需要配置包含依赖关系的服务器架构时尤为重要。

Ansible的学习曲线非常平滑，你不需要掌握编程技能，更不需要学习新的语言。Ansible内置的功能应该能够满足超过80%的用户需求，而且它不会遇到扩展性方面的瓶颈（因为没有中心节点）。

如果系统中安装了factor，Ansible同样支持从中获取系统信息。Ansible使用jinja2作为模板语言，类似于Puppet使用erb文件作为模板。Ansible可以使用自己的信息收集工具，因此factor并不是必需的。

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
