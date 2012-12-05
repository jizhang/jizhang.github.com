---
layout: post
title: "开源软件架构 - 卷2：第14章 Python打包工具"
date: 2012-11-18 19:20
comments: true
categories: Translation
tags: [aosa, python]
published: false
---

作者：[Tarek Ziadé](http://www.aosabook.org/en/intro1.html#ziade-tarek)，翻译：[张吉](mailto:zhangji87@gmail.com)

原文：[http://www.aosabook.org/en/packaging.html](http://www.aosabook.org/en/packaging.html)

14.1 简介
---------

对于如何安装软件，目前有两种思想流派。第一种是说软件应该自给自足，不依赖于其它任何部件，这点在Windows和Mac OS X系统中很流行。这种方式简化了软件的管理：每个软件都有自己独立的“领域”，安装和卸载它们不会对操作系统产生影响。如果软件依赖一项不常见的类库，那么这个类库一定是包含在软件安装包之中的。

第二种流派，主要在类Linux的操作系统中盛行，即软件应该是由一个个独立的、小型的软件包组成的。类库被包含在软件包中，包与包之间可以有依赖关系。安装软件时需要查找和安装它所依赖的其他特定版本的软件包。这些依赖包通常是从一个包含所有软件包的中央仓库中获取的。这种理念也催生了Linux发行版中那些复杂的依赖管理工具，如`dpkg`和`RPM`。它们会跟踪软件包的依赖关系，并防止两个软件使用了版本相冲突的第三方包。

以上两种流派各有优劣。高度模块化的系统可以使得更新和替换某个软件包变的非常方便，因为每个类库都只有一份，所有依赖于它的应用程序都能因此受益。比如，修复某个类库的安全漏洞可以立刻应用到所有程序中，而如果应用程序使用了自带的类库，那安全更新就很难应用进去了，特别是在类库版本不一致的情况下更难处理。

不过这种“模块化”也被一些开发者视为缺点，因为他们无法控制应用程序的依赖关系。他们希望提供一个独立和稳定的软件运行环境，这样就不会在系统升级后遭遇各种依赖方面的问题。

在安装程序中包含所有依赖包还有一个优点：便于跨平台。有些项目在这点上做到了极致，它们将所有和操作系统的交互都封装了起来，在一个独立的目录中运行，甚至包括日志文件的记录位置。

Python的打包系统使用的是第二种设计思想，并尽可能地方便开发者、管理员、用户对软件的管理。不幸的是，这种方式导致了种种问题：错综复杂的版本结构、混乱的数据文件、难以重新打包等等。三年前，我和其他一些Python开发者决定研究解决这个问题，我们自称为“Python打包工具小分队”，本文就是讲述我们在这个问题上做出的努力和取得的成果。

### 术语

在Python中， *包* 表示一个包含Python文件的目录。Python文件被称为 *模块* ，这样一来，使用“包”这个单词就显得有些模糊了，因为它常常用来表示某个项目的 *发行版本* 。

Python开发者有时也对此表示不能理解。为了更清晰地进行表述，我们用“Python包（package）”来表示一个包含Python文件的目录，用“发行版本（release）”来表示某个项目的特定版本，用“发布包（distribution）”来表示某个发行版本的源码或二进制文件，通常是Tar包或Zip文件的形式。

14.2 Python开发者的困境
---------------------

大多数Python开发者希望自己的程序能够在任何环境中运行。他们还希望自己的软件既能使用标准的Python类库，又能使用依赖于特定系统类型的类库。但除非开发者使用现有的各种打包工具生成不同的软件包，否则他们打出的软件安装包就必须在一个安装有Python环境的系统中运行。这样的软件包还希望做到以下几点：

* 其他人可以针对不同的目标系统对这个软件重新打包；
* 软件所依赖的包也能够针对不同的目标系统进行重新打包；
* 系统依赖项能够被清晰地描述出来。

要做到以上几点往往是不可能的。举例来说，Plone这一功能全面的CMS系统，使用了上百个纯Python语言编写的类库，而这些类库并不一定在所有的打包系统中提供。这就意味着Plone必须将它所依赖的软件包都集成到自己的安装包中。要做到这一点，他们选择使用`zc.buildout`这一工具，它能够将所有的依赖包都收集起来，生成一个完整的应用程序文件，在独立的目录中运行。它事实上是一个二进制的软件包，因为所有C语言代码都已经编译好了。

这对开发者来说是福音：他们只需要描述好依赖关系，然后借助`zc.buildout`来发布自己的程序即可。但正如上文所言，这种发布方式在系统层面构筑了一层屏障，这让大多数Linux系统管理员非常恼火。Windows管理员不会在乎这些，但CentOS和Debian管理员则会，因为按照他们的管理原则，系统中的所有文件都应该被注册和归类到现有的管理工具中。

这些管理员会想要将你的软件按照他们自己的标准重新打包。问题在于：Python有没有这样的打包工具，能够自动地按照新的标准重新打包？如果有，那么Python的任何软件和类库就能够针对不同的目标系统进行打包，而不需要额外的工作。这里，“自动”一词并不是说打包过程可以完全由脚本来完成——这点上`RPM`和`dpkg`的使用者已经证实是不可能的了，因为他们总会需要增加额外的信息来重新打包。他们还会告诉你，在重新打包的过程中会遇到一些开发者没有遵守基本打包原则的情况。

我们来举一个实际例子，如何通过使用现有的Python打包工具来惹恼那些想要重新打包的管理员：在发布一个名为“MathUtils”的软件包时使用“Fumanchu”这样的版本号名字。撰写这个类库的数学家想用自家猫咪的名字来作为版本号，但是管理员怎么可能知道“Fumanchu”是他家第二只猫的名字，第一只猫叫做“Phil”，所以“Fumanchu”版本要比“Phil”版本来得高？

可能这个例子有些极端，但是在现有的打包工具和规范中是可能发生的。最坏的情况是`easy_install`和`pip`使用自己的一套标准来追踪已安装的文件，并使用字母顺序来比较“Fumanchu”和“Phil”的版本高低。

另一个问题是如何处理数据文件。比如，如果你的软件使用了SQLite数据库，安装时被放置在包目录中，那么在程序运行时，系统会阻止你对其进行读写操作。这样做还会破坏Linux系统的一项惯例，即`/var`目录下的数据文件是需要进行备份的。

在现实环境中，系统管理员需要能够将你的文件放置到他们想要的地方，并且不破坏程序的完整性，这就需要你来告诉他们各类文件都是做什么用的。让我们换一种方式来表述刚才的问题：Python是否有这样一种打包工具，它可以提供各类信息，足以让第三方打包工具能据此重新进行打包，而不需要阅读软件的源码？

14.3 现有的打包管理架构
--------------------

Python标准库中提供的`Distutils`打包工具充斥了上述的种种问题，但由于它是一种标准，所以人们要么继续忍受并使用它，或者转向更先进的工具`Setuptools`，它在Distutils之上提供了一些高级特性。另外还有`Distribute`，它是`Setuptools`的衍生版本。`Pip`则是一种更为高级的安装工具，它依赖于`Setuptools`。

但是，这些工具都源自于`Distutils`，并继承了它的种种问题。有人也想过要改进`Distutils`本身，但是由于它的使用范围已经很广很广，任何小的改动都会对Python软件包的整个生态系统造成冲击。

所以，我们决定冻结`Distutils`的代码，并开始研发`Distutils2`，不去考虑向前兼容的问题。为了解释我们所做的改动，首先让我们近距离观察一下`Distutils`。

### 14.3.1 Distutils基础及设计缺陷

`Distutils`由一些命令组成，每条命令都是一个包含了`run`方法的类，可以附加若干参数进行调用。`Distutils`还提供了一个名为`Distribution`的类，它包含了一些全局变量，可供其他命令使用。

当要使用`Distutils`时，Python开发者需要在项目中添加一个模块，通常命名为`setup.py`。这个模块会调用`Distutils`的入口函数：`setup`。这个函数有很多参数，这些参数会被`Distribution`实例保存起来，供后续使用。下面这个例子中我们指定了一些常用的参数，如项目名称和版本，它所包含的模块等：

```python
from distutils.core import setup

setup(name='MyProject', version='1.0', py_modules=['mycode.py'])
```

这个模块可以用来执行`Distutils`的各种命令，如`sdist`。这条命令会在`dist`目录中创建一个源代码发布包：

```bash
$ python setup.py sdist
```

这个模块还可以执行`install`命令：

```bash
$ python setup.py install
```

`Distutils`还提供了一些其他命令：

* `upload` 将发布包上传至在线仓库
* `register` 向在线仓库注册项目的基本信息，而不上传发布包
* `bdist` 创建二进制发布包
* `bdist_msi` 创建`.msi`安装包，供Windows系统使用

我们还可以使用其他一些命令来获取项目的基本信息。

所以在安装或获取应用程序信息时都是通过这个文件调用`Distutils`实现的，如获取项目名称：

```bash
$ python setup.py --name
MyProject
```

`setup.py`是一个项目的入口，可以通过它对项目进行构建、打包、发布、安装等操作。开发者通过这个函数的参数信息来描述自己的项目，并使用它进行各种打包任务。这个文件同样用于在目标系统中安装软件。

![图14.1 安装](http://www.aosabook.org/images/packaging/setup-py.png)

图14.1 安装

然而，使用同一个文件来对项目进行打包、发布、以及安装，是`Distutils`的主要缺点。例如，你需要查看`lxml`项目的名称属性，`setup.py`会执行很多其他无关的操作，而不是简单返回一个字符串：

```bash
$ python setup.py --name
Building lxml version 2.2.
NOTE: Trying to build without Cython, pre-generated 'src/lxml/lxml.etree.c'
needs to be available.
Using build configuration of libxslt 1.1.26
Building against libxml2/libxslt in the following directory: /usr/lib/lxml
```

在有些项目中它甚至会执行失败，因为开发者默认为`setup.py`只是用来安装软件的，而其他一些`Distutils`功能只在开发过程中使用。因此，`setup.py`的角色太多，容易引起他人的困惑。

14.3.2 元信息和PyPI
------------------

`Distutils`在构建发布包时会创建一个`Metadata`文件。这个文件是按照PEP314<sup>1</sup>编写的，包含了一些常见的项目信息，包括名称、版本等，主要有以下几项：

* `Name`：项目名称
* `Version`：发布版本号
* `Summary`：项目简介
* `Description`：项目详情
* `Home-Page`：项目主页
* `Author`：作者
* `Classifers`：项目类别。Python为不同的发行协议、发布版本（beta，alpha，final）等提供了不同的类别。
* `Requires`，`Provides`，`Obsoletes`：描述项目依赖信息

这些信息一般都能移植到其他打包系统中。

Python包目录（Python Package Index，简称PyPI<sup>2</sup>），是一个类似CPAN的中央软件包仓库，可以调用`Distutils`的`register`和`upload`命令来注册和发布项目。`register`命令会构建`Metadata`文件并传送给PyPI，让访问者和安装工具能够浏览和搜索。

![图14.2：PyPI仓库](http://www.aosabook.org/images/packaging/pypi.png)

图14.2：PyPI仓库

你可以通过`Classifies`（类别）来浏览，获取项目作者的名字和主页。同时，`Requires`可以用来定义Python模块的依赖关系。`requires`选项可以向元信息文件的`Requires`字段添加信息：

```python
from distutils.core import setup

setup(name='foo', version='1.0', requires=['ldap'])
```

这里声明了对`ldap`模块的依赖，这种依赖并没有实际效力，因为没有安装工具会保证这个模块真实存在。如果说Python代码中会使用类似Perl的`require`关键字来定义依赖关系，那还有些作用，因为这时安装工具会检索PyPI上的信息并进行安装，其实这也就是CPAN的做法。但是对于Python来说，`ldap`模块可以存在于任何项目之中，因为`Distutils`是允许开发者发布一个包含多个模块的软件的，所以这里的元信息字段并无太大作用。

`Metadata`的另一个缺点是，因为它是由Python脚本创建的，所以会根据脚本执行环境的不同而产生特定信息。比如，运行在Windows环境下的一个项目会在`setup.py`文件中有以下描述：

```python
from distutils.core import setup

setup(name='foo', version='1.0', requires=['win32com'])
```

这样配置相当于是默认该项目只会运行在Windows环境下，即使它可能提供了跨平台的方案。一种解决方法是根据不同的平台来指定`requires`参数：

```python
from distutils.core import setup
import sys

if sys.platform == 'win32':
    setup(name='foo', version='1.0', requires=['win32com'])
else:
    setup(name='foo', version='1.0')
```

但这种做法往往会让事情更糟。要注意，这个脚本是用来将项目的源码包发布到PyPI上的，这样写就说明它向PyPI上传的`Metadata`文件会因为该脚本运行环境的不同而不同。换句话说，这使得我们无法在元信息文件中看出这个项目依赖于特定的平台。

14.3.3 PyPI的架构设计
---------------------

![图14.3 PyPI工作流](http://www.aosabook.org/images/packaging/pypi-workflow.png)

图14.3 PyPI工作流

如上文所述，PyPI是一个Python项目的中央仓库，人们可以通过不同的类别来搜索已有的项目，也可以创建自己的项目。人们可以上传项目源码和二进制文件，供其他人下载使用或研究。同时，PyPI还提供了相应的Web服务，让安装工具可以调用它来检索和下载文件。
### 注册项目并上传发布包

我们可以使用`Distutils`的`register`命令在PyPI中注册一个项目。这个命令会根据项目的元信息生成一个POST请求。该请求会包含验证信息，PyPI使用HTTP基本验证来确保所有的项目都和一个注册用户相关联。验证信息保存在`Distutils`的配置文件中，或在每次执行`register`命令时提示用户输入。以下是一个使用示例：

```bash
$ python setup.py register
running register
Registering MPTools to http://pypi.python.org/pypi
Server response (200): OK
```

每个注册项目都会产生一个HTML页面，上面包含了它的元信息。开发者可以使用`upload`命令将发布包上传至PyPI：

```bash
$ python setup.py sdist upload
running sdist
…
running upload
Submitting dist/mopytools-0.1.tar.gz to http://pypi.python.org/pypi
Server response (200): OK
```

如果开发者不想将代码上传至PyPI，可以使用元信息中的`Download-URL`属性来指定一个外部链接，供用户下载。

### 检索PyPI

除了在页面中检索项目，PyPI还提供了两个接口供程序调用：简单索引协议和XML-PRC API。

简单索引协议的地址是`http://pypi.python.org/simple/`，它包含了一个链接列表，指向所有的注册项目：

```html
<html><head><title>Simple Index</title></head><body>
⋮    ⋮    ⋮
<a href='MontyLingua/'>MontyLingua</a><br/>
<a href='mootiro_web/'>mootiro_web</a><br/>
<a href='Mopidy/'>Mopidy</a><br/>
<a href='mopowg/'>mopowg</a><br/>
<a href='MOPPY/'>MOPPY</a><br/>
<a href='MPTools/'>MPTools</a><br/>
<a href='morbid/'>morbid</a><br/>
<a href='Morelia/'>Morelia</a><br/>
<a href='morse/'>morse</a><br/>
⋮    ⋮    ⋮
</body></html>
```

如MPTools项目对应的`MPTools/`目录，它所指向的路径会包含以下内容：

* 所有发布包的地址
* 在`Metadata`中定义的项目网站地址，包含所有版本
* 下载地址（`Download-URL`），同样包含所有版本

以MPTools项目为例：

```html
<html><head><title>Links for MPTools</title></head>
<body><h1>Links for MPTools</h1>
<a href="../../packages/source/M/MPTools/MPTools-0.1.tar.gz">MPTools-0.1.tar.gz</a><br/>
<a href="http://bitbucket.org/tarek/mopytools" rel="homepage">0.1 home_page</a><br/>
</body></html>
```

安装工具可以通过访问这个索引来查找项目的发布包，或者检查`http://pypi.python.org/simple/PROJECT_NAME/`是否存在。

但是，这个协议主要有两个缺陷。首先，PyPI目前还是单台服务器。虽然很多用户会自己搭建镜像，但过去两年中曾发生过几次PyPI无法访问的情况，用户无法下载依赖包，导致项目构建出现问题。比如说，在构建一个Plone项目时，需要向PyPI发送近百次请求。所以PyPI在这里成为了单点故障。

其次，当项目的发布包没有保存在PyPI中，而是通过`Download-URL`指向了其他地址，安装工具就需要重定向到这个地址下载发布包。这种情况也会增加安装过程的不稳定性。

简单索引协议只是提供给安装工具一个项目列表，并不包含项目元信息。可以通过PyPI的XML-RPC API来获取项目元信息：

```python
>>> import xmlrpclib
>>> import pprint
>>> client = xmlrpclib.ServerProxy('http://pypi.python.org/pypi')
>>> client.package_releases('MPTools')
['0.1']
>>> pprint.pprint(client.release_urls('MPTools', '0.1'))
[{'comment_text': &rquot;,
'downloads': 28,
'filename': 'MPTools-0.1.tar.gz',
'has_sig': False,
'md5_digest': '6b06752d62c4bffe1fb65cd5c9b7111a',
'packagetype': 'sdist',
'python_version': 'source',
'size': 3684,
'upload_time': <DateTime '20110204T09:37:12' at f4da28>,
'url': 'http://pypi.python.org/packages/source/M/MPTools/MPTools-0.1.tar.gz'}]
>>> pprint.pprint(client.release_data('MPTools', '0.1'))
{'author': 'Tarek Ziade',
'author_email': 'tarek@mozilla.com',
'classifiers': [],
'description': 'UNKNOWN',
'download_url': 'UNKNOWN',
'home_page': 'http://bitbucket.org/tarek/mopytools',
'keywords': None,
'license': 'UNKNOWN',
'maintainer': None,
'maintainer_email': None,
'name': 'MPTools',
'package_url': 'http://pypi.python.org/pypi/MPTools',
'platform': 'UNKNOWN',
'release_url': 'http://pypi.python.org/pypi/MPTools/0.1',
'requires_python': None,
'stable_version': None,
'summary': 'Set of tools to build Mozilla Services apps',
'version': '0.1'}
```

这种方式的问题在于，项目元信息原本就能以静态文件的方式在简单索引协议中提供，这样可以简化安装工具的复杂性，也可以减少PyPI服务的请求数。对于诸如下载数量这样的动态数据，可以在其他接口中提供。用两种服务来获取所有的静态内容，显然不太合理。

14.3.4 Python安装目录的结构
---------------------------

在使用`python setup.py install`安装一个Python项目后，`Distutils`这一Python核心类库会负责将程序代码复制到目标系统的相应位置。

* *Python包* 和模块会被安装到Python解释器程序所在的目录中，并随解释器启动：Ubuntu系统中会安装到`/usr/local/lib/python2.6/dist-packages/`，Fedora则是`/usr/local/lib/python2.6/sites-packages/`。
* 项目中的 *数据文件* 可以被安装到任何位置。
* *可执行文件* 会被安装到系统的`bin`目录下，依平台类型而定，可能是`/usr/local/bin`，或是其它指定的目录。

从Python2.5开始，项目的元信息文件会随模块和包一起发布，名称为`project-version.egg-info`。比如，`virtualenv`项目会有一个`virtualenv-1.4.9.egg-info`文件。这些元信息文件可以被视为一个已安装项目的数据库，因为可以通过遍历其中的内容来获取已安装的项目和版本。但是，`Distutils`并没有记录项目所安装的文件列表，也就是说，我们无法彻底删除安装某个项目后产生的所有文件。可惜的是，`install`命令本身是提供了一个名为`--record`的参数的，可以将已安装的文件列表记录在文本文件中，但是这个参数并没有默认开启，而且`Distutils`的文档中几乎没有提及这个参数。

14.3.5 Setuptools、Pip等工具
----------------------------

正如介绍中所提到的，有些项目已经在尝试修复`Distutils`的问题，并取得了一些成功。

### 依赖问题

PyPI允许开发者在发布的项目中包含多个模块，还允许项目通过定义`Require`属性来声明模块级别的依赖。这两种做法都是合理的，但是同时使用就会很糟糕。

正确的做法应该是定义项目级别的依赖，这也是`Setuptools`在`Distutils`之上附加的一个特性。它还提供了一个名为`easy_install`的脚本来从PyPI上自动获取和安装依赖项。在实际生产中，模块级别的依赖并没有真正被使用，更多人倾向于使用`Setuptools`。然而，这些特性只是针对`Setuptools`的，并没有被`Distutils`或PyPI所接受，所以`Setuptools`实质上是一个构建在错误设计上的仿冒品。

`easy_install`需要下载项目的压缩文档，执行`setup.py`来获取元信息，并对每个依赖项进行相同的操作。项目的依赖树会随着软件包的下载逐步勾画出来。

虽然PyPI上可以直接浏览项目元信息，但是`easy_install`还是需要下载所有的软件包，因为上文提到过，PyPI上的项目元信息很可能和上传时所使用的平台有关，从而和目标系统有所差异。但是这种一次性安装项目依赖的做法已经能够解决90%的问题了，的确是个很不错的特性。这也是为什么`Setuptools`被广泛采用的原因。然而，它还是有以下一些问题：

* 如果某一个依赖项安装失败，它并没有提供回滚的选项，因此系统会处于一个不可用的状态。
* 项目依赖树是在安装一个个软件包时构建出来的，因此当其中两个依赖项产生冲突时，系统也会变的不可用。

### 卸载的问题

虽然`Setuptools`可以在元信息中记录已安装的文件，但它并没有提供卸载功能。另一个工具`Pip`，它通过扩展`Setuptools`的元信息来记录已安装的文件，从而能够进行卸载操作。但是，这组信息又是一种自定义的内容，因此一个Python项目很可能包含四种不同的元信息：

* `Distutils`的`egg-info`，一个单一的文件；
* `Setuptools`的`egg-info`，一个目录，记录了`Setuptools`特定的元信息；
* `Pip`的`egg-info`，是后者的扩展；
* 其它由打包系统产生的信息。

14.3.6 数据文件如何处理？
-------------------------

在`Distutils`中，数据文件可以被安装在任意位置。你可以像这样在`setup.py`中定义一个项目的数据文件：

```python
setup(…,
  packages=['mypkg'],
  package_dir={'mypkg': 'src/mypkg'},
  package_data={'mypkg': ['data/*.dat']},
  )
```

那么，`mypkg`项目中所有以`.dat`为扩展名的文件都会被包含在发布包中，并随Python代码安装到目标系统。

对于需要安装到项目目录之外的数据文件，可以进行如下配置。他们随项目一起打包，并安装到指定的目录中：

```python
setup(…,
    data_files=[('bitmaps', ['bm/b1.gif', 'bm/b2.gif']),
                ('config', ['cfg/data.cfg']),
                ('/etc/init.d', ['init-script'])]
    )
```

这对系统打包人员来说简直是噩梦：

* 元信息中并不包含数据文件的信息，因此打包人员需要阅读`setup.py`文件，甚至是研究项目源码来获取这些信息。
* 不应该由开发人员来决定项目数据文件应该安装到目标系统的哪个位置。
* 数据文件没有区分类型，图片、帮助文件等都被视为同等来处理。

打包人员在对项目进行打包时只能去根据目标系统的情况来修改`setup.py`文件，从而让软件包能够顺利安装。要做到这一点，他就需要阅读程序代码，修改所有用到这些文件的地方。`Setuptools`和`Pip`并没有解决这一问题。

脚注
---
1. 文中引用的Python改进提案（Python Enhancement Proposals，简称PEP）会在本文最后一节整理。
2. 过去被命名为CheeseShop