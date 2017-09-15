---
layout: post
title: "开源软件架构 - 卷1：第14章 Python 打包工具"
date: 2012-11-18 19:20
comments: true
categories: Programming
tags: [aosa, python, translation]
---

作者：[Tarek Ziadé](http://www.aosabook.org/en/intro1.html#ziade-tarek)，翻译：[张吉](mailto:zhangji87@gmail.com)

原文：[http://www.aosabook.org/en/packaging.html](http://www.aosabook.org/en/packaging.html)

14.1 简介
---------

对于如何安装软件，目前有两种思想流派。第一种是说软件应该自给自足，不依赖于其它任何部件，这点在Windows和Mac OS X系统中很流行。这种方式简化了软件的管理：每个软件都有自己独立的“领域”，安装和卸载它们不会对操作系统产生影响。如果软件依赖一项不常见的类库，那么这个类库一定是包含在软件安装包之中的。

第二种流派，主要在类Linux的操作系统中盛行，即软件应该是由一个个独立的、小型的软件包组成的。类库被包含在软件包中，包与包之间可以有依赖关系。安装软件时需要查找和安装它所依赖的其他特定版本的软件包。这些依赖包通常是从一个包含所有软件包的中央仓库中获取的。这种理念也催生了Linux发行版中那些复杂的依赖管理工具，如`dpkg`和`RPM`。它们会跟踪软件包的依赖关系，并防止两个软件使用了版本相冲突的第三方包。

以上两种流派各有优劣。高度模块化的系统可以使得更新和替换某个软件包变的非常方便，因为每个类库都只有一份，所有依赖于它的应用程序都能因此受益。比如，修复某个类库的安全漏洞可以立刻应用到所有程序中，而如果应用程序使用了自带的类库，那安全更新就很难应用进去了，特别是在类库版本不一致的情况下更难处理。

不过这种“模块化”也被一些开发者视为缺点，因为他们无法控制应用程序的依赖关系。他们希望提供一个独立和稳定的软件运行环境，这样就不会在系统升级后遭遇各种依赖方面的问题。

<!-- more -->

在安装程序中包含所有依赖包还有一个优点：便于跨平台。有些项目在这点上做到了极致，它们将所有和操作系统的交互都封装了起来，在一个独立的目录中运行，甚至包括日志文件的记录位置。

Python的打包系统使用的是第二种设计思想，并尽可能地方便开发者、管理员、用户对软件的管理。不幸的是，这种方式导致了种种问题：错综复杂的版本结构、混乱的数据文件、难以重新打包等等。三年前，我和其他一些Python开发者决定研究解决这个问题，我们自称为“打包别动队”，本文就是讲述我们在这个问题上做出的努力和取得的成果。

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

### 14.3.2 元信息和PyPI

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

Python项目索引（Python Package Index，简称PyPI<sup>2</sup>），是一个类似CPAN的中央软件包仓库，可以调用`Distutils`的`register`和`upload`命令来注册和发布项目。`register`命令会构建`Metadata`文件并传送给PyPI，让访问者和安装工具能够浏览和搜索。

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

### 14.3.3 PyPI的架构设计

![图14.3 PyPI工作流](http://www.aosabook.org/images/packaging/pypi-workflow.png)

图14.3 PyPI工作流

如上文所述，PyPI是一个Python项目的中央仓库，人们可以通过不同的类别来搜索已有的项目，也可以创建自己的项目。人们可以上传项目源码和二进制文件，供其他人下载使用或研究。同时，PyPI还提供了相应的Web服务，让安装工具可以调用它来检索和下载文件。

#### 注册项目并上传发布包

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

#### 检索PyPI

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

### 14.3.4 Python安装目录的结构

在使用`python setup.py install`安装一个Python项目后，`Distutils`这一Python核心类库会负责将程序代码复制到目标系统的相应位置。

* *Python包* 和模块会被安装到Python解释器程序所在的目录中，并随解释器启动：Ubuntu系统中会安装到`/usr/local/lib/python2.6/dist-packages/`，Fedora则是`/usr/local/lib/python2.6/sites-packages/`。
* 项目中的 *数据文件* 可以被安装到任何位置。
* *可执行文件* 会被安装到系统的`bin`目录下，依平台类型而定，可能是`/usr/local/bin`，或是其它指定的目录。

从Python2.5开始，项目的元信息文件会随模块和包一起发布，名称为`project-version.egg-info`。比如，`virtualenv`项目会有一个`virtualenv-1.4.9.egg-info`文件。这些元信息文件可以被视为一个已安装项目的数据库，因为可以通过遍历其中的内容来获取已安装的项目和版本。但是，`Distutils`并没有记录项目所安装的文件列表，也就是说，我们无法彻底删除安装某个项目后产生的所有文件。可惜的是，`install`命令本身是提供了一个名为`--record`的参数的，可以将已安装的文件列表记录在文本文件中，但是这个参数并没有默认开启，而且`Distutils`的文档中几乎没有提及这个参数。

### 14.3.5 Setuptools、Pip等工具

正如介绍中所提到的，有些项目已经在尝试修复`Distutils`的问题，并取得了一些成功。

#### 依赖问题

PyPI允许开发者在发布的项目中包含多个模块，还允许项目通过定义`Require`属性来声明模块级别的依赖。这两种做法都是合理的，但是同时使用就会很糟糕。

正确的做法应该是定义项目级别的依赖，这也是`Setuptools`在`Distutils`之上附加的一个特性。它还提供了一个名为`easy_install`的脚本来从PyPI上自动获取和安装依赖项。在实际生产中，模块级别的依赖并没有真正被使用，更多人倾向于使用`Setuptools`。然而，这些特性只是针对`Setuptools`的，并没有被`Distutils`或PyPI所接受，所以`Setuptools`实质上是一个构建在错误设计上的仿冒品。

`easy_install`需要下载项目的压缩文档，执行`setup.py`来获取元信息，并对每个依赖项进行相同的操作。项目的依赖树会随着软件包的下载逐步勾画出来。

虽然PyPI上可以直接浏览项目元信息，但是`easy_install`还是需要下载所有的软件包，因为上文提到过，PyPI上的项目元信息很可能和上传时所使用的平台有关，从而和目标系统有所差异。但是这种一次性安装项目依赖的做法已经能够解决90%的问题了，的确是个很不错的特性。这也是为什么`Setuptools`被广泛采用的原因。然而，它还是有以下一些问题：

* 如果某一个依赖项安装失败，它并没有提供回滚的选项，因此系统会处于一个不可用的状态。
* 项目依赖树是在安装一个个软件包时构建出来的，因此当其中两个依赖项产生冲突时，系统也会变的不可用。

#### 卸载的问题

虽然`Setuptools`可以在元信息中记录已安装的文件，但它并没有提供卸载功能。另一个工具`Pip`，它通过扩展`Setuptools`的元信息来记录已安装的文件，从而能够进行卸载操作。但是，这组信息又是一种自定义的内容，因此一个Python项目很可能包含四种不同的元信息：

* `Distutils`的`egg-info`，一个单一的文件；
* `Setuptools`的`egg-info`，一个目录，记录了`Setuptools`特定的元信息；
* `Pip`的`egg-info`，是后者的扩展；
* 其它由打包系统产生的信息。

### 14.3.6 数据文件如何处理？

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

14.4 改进标准
-------------

所以最后我们得到的是这样一个打包系统：所有功能都由一个模块提供，项目的元信息不完整，无法描述清楚项目中包含的所有内容。现在就让我们来做些改进。

### 14.4.1 元信息

首先，我们要修正`Metadata`标准中的内容。PEP 345定义了一个新的标准，它包含以下内容：

* 更合理的版本定义方式
* 项目级别的依赖关系
* 使用一种静态的方式描述平台相关的属性

#### 版本

元信息标准的目标之一是能够让Python包管理工具使用相同的方式来对项目进行分类。对于版本号来说，应该让所有的工具都知道“1.1”是在“1.0”之后的。如果项目使用了自己定义的版本号命名方式，就无法做到这一点了。

要保证这种一致性，唯一的方法是让所有的项目都按照统一的方式来命名版本号。我们选择的方式是经典的序列版本号，在PEP 386中定义，它的格式是：

```
N.N[.N]+[{a|b|c|rc}N[.N]+][.postN][.devN]
```

其中：

* *N* 是一个整数。你可以使用任意数量的N，用点号将它们分隔开来。但至少要有两个N，即“主版本.次版本”。
* *a, b, c* 分别是 *alpha, beta, release candidate* 的简写，它们后面还有一个整数。预发布版本有两种标记，c和rc，主要是为了和过去兼容，但c更简单些。
* *dev* 加一个数字表示开发版本。
* *post* 加一个数字表示已发布版本。

根据项目发布周期的不同，开发版本和已发布版本可以作为两个最终版本之间的过渡版本号。大多数项目会使用开发版本。

按照这个形式，PEP 386定义了严格的顺序：

* alpha < beta < rc < final
* dev < non-dev < post, non-dev包括alpha, beta, rc或者final

以下是一个完整的示例：

```
1.0a1 < 1.0a2.dev456 < 1.0a2 < 1.0a2.1.dev456
  < 1.0a2.1 < 1.0b1.dev456 < 1.0b2 < 1.0b2.post345
    < 1.0c1.dev456 < 1.0c1 < 1.0.dev456 < 1.0
      < 1.0.post456.dev34 < 1.0.post456
```

这样定义的目标在于让其他打包系统能够将Python项目的版本号方便地转换成它们自己的版本命名规则。目前，如果上传的项目使用了PEP 345定义的元信息，PyPI会拒绝接受没有遵守PEP 386版本号命名规范的项目。

#### 依赖

PEP 345定义了三个新的元信息属性，用来替换PEP 314中的`Requires`，`Provides`，和`Obsoletes`，它们是`Requires-Dist`，`Provides-Dist`，`Obsoletes-Dist`。这些属性可以在元信息中出现多次。

`Requires-Dist`中定义了项目所依赖的软件包，使用依赖项目的`Name`元信息，并可以跟上一个版本号。这些依赖项目的名称必须能在PyPI中找到，且版本号命名规则要遵守PEP 386中的定义。以下是一些示例：

```
Requires-Dist: pkginfo
Requires-Dist: PasteDeploy
Requires-Dist: zope.interface (>3.5.0)
```

`Provides-Dist`用来定义项目中包含的其他项目，常用于合并两个项目的情形。比如，ZODB项目可以包含名为`transaction`的项目，并声明：

```
Provides-Dist: transaction
```

`Obsoletes-Dist`主要用于将其它项目标记为本项目的过期版本。

```
ObsoletesDist: OldName
```

#### 环境标识

环境标识可以添加在上述三个属性的后面，使用分号分隔，用来标识该属性在什么样的目标环境中生效。以下是一些示例：

```
Requires-Dist: pywin32 (>1.0); sys.platform == 'win32'
Obsoletes-Dist: pywin31; sys.platform == 'win32'
Requires-Dist: foo (1,!=1.3); platform.machine == 'i386'
Requires-Dist: bar; python_version == '2.4' or python_version == '2.5'
Requires-External: libxslt; 'linux' in sys.platform
```

这种简易的语法足以让非Python程序员看懂：它使用`==`或`in`运算符（含`!=`和`not in`），且可以通过逻辑运算符连接。PEP 345中规定以下属性可以使用这种语法：

* `Requires-Python`
* `Requires-External`
* `Requires-Dist`
* `Provides-Dist`
* `Obsoletes-Dist`
* `Classifier`

### 14.4.2 用户安装了什么？

出于互通性的考虑，Python项目的安装格式必须一致。要让安装工具A能够检测到工具B安装的项目，它们就必须共享和更新相同的项目列表。

当然，理想中用户会在系统中只使用一种安装工具，但是他们也许会需要迁移到另一种工具以获得一些新的特性。比如，Mac OS X操作系统自带了`Setuptools`，因而装有`easy_install`工具。当他们想要切换到新的工具时，该工具就必须兼容现有的环境。

如果系统使用类似RPM这样的工具管理Python软件包，那么其它安装工具在安装新项目时是无法通知到系统的。更糟糕的是，即便Python安装工具能够通知到中央打包系统，我们也必须在Python元信息和系统元信息之间做一个映射。比如，项目的名称在两个系统中可能是不一致的。造成这种问题的原因也多种多样，比较常见的原因是命名冲突，即RPM源中已经有一个同名的项目了。另一个原因是项目名称中包含了`python`这个前缀，从而破坏了RPM系统的规范。比如，你的项目名称是`foo-python`，那在RPM源中很可能被表示为`python-foo`。

一种解决办法是不去触碰全局的Python环境，而是使用一个隔离的环境，如`Virtualenv`。

但不管怎样，采用统一的Python安装格式还是有必要的，因为其它一些打包系统在为自己安装Python项目时还是需要考虑互通性。当第三方打包系统新安装了一个项目，并在自身的数据库中注册后，它还需要为Python安装环境生成一个正确的元信息，从而让项目在这个环境中变得可见，或能通过该Python环境提供的API检索到。

元信息的映射问题可以这样描述：因为RPM系统知道自己安装了哪些Python项目，它就能生成合适的Python元信息。例如，它知道`python26-webob`项目在PyPI中的名字是`WebOb`。

回到我们的规范：PEP 376定义的项目安装规范和`Seteptools`以及`Pip`的格式很相似，它是一个以`dist-info`结尾的目录，包含以下内容：

* `METADATA`：元信息，其格式在PEP 345、PEP 314和PEP 241中描述。
* `RECORD`：项目安装的文件列表，以类似csv的格式保存。
* `INSTALLER`：安装项目所使用的工具。
* `REQUESTED`：如果这个文件存在，则表明这个项目是被显式安装的，即并不是作为依赖项而安装。

如果所有的安装工具都能识别这种格式，我们在管理Python项目时就不需要依赖特定的安装工具和它提供的特性了。此外，PEP 376将元信息设计为一个目录，这样就能方便地扩展。事实上，下一章要描述的`RESOURCES`文件很可能会在不久的将来添加到元信息中，而不用改变PEP 376标准。当事实证明这个文件能被所有的安装工具使用，则会将它修订到PEP中。

### 14.4.3 数据文件的结构

前面已经提到，我们需要能够让打包者来决定项目的数据文件安装在哪个位置，而不用修改代码。同样，也要能够让开发者在开发时不用去考虑数据文件的存放位置。我们的解决方案很普通：重定向。

#### 使用数据文件

假设你的`MPTools`项目需要使用一个配置文件。开发者会将改文件放到Python包安装目录中，并使用`__file__`去引用：

```python
import os

here = os.path.dirname(__file__)
cfg = open(os.path.join(here, 'config', 'mopy.cfg'))
```

这样编写代码意味着该配置文件必须和代码放在相同的位置，一个名为`config`的子目录下。

我们设计的新的数据文件结构以项目为根节点，开发者可以定义任意的文件目录结构，而不用关心根目录是存放在软件安装目录中或是其它目录。开发者可以使用`pkgutil.open`来访问这些数据文件：

```python
import os
import pkgutil

# Open the file located in config/mopy.cfg in the MPTools project
cfg = pkgutil.open('MPTools', 'config/mopy.cfg')
```

`pkgutil.open`命令会检索项目元信息中的`RESOURCES`文件，该文件保存的是一个简单的映射信息——文件名称和它所存放的位置：

```
config/mopy.cfg {confdir}/{distribution.name}
```

其中，`{confdir}`变量指向系统的配置文件目录，`{distribution.name}`变量表示的是Python项目名称。

![图14.4：定位一个文件](http://www.aosabook.org/images/packaging/find-file.png)

图14.4：定位一个文件

只要安装过程中生成了`RESOURCES`文件，这个API就能帮助开发者找到`mopy.cfg`文件。又因为`config/mopy.cfg`是一个相对于项目的路径，我们就能在开发模式下提供一个本地的路径，让`pkgutil`能够找到它。

#### 声明数据文件

实际使用中，我们可以在`setup.cfg`文件中用映射关系来定义数据文件的存放位置。映射关系的形式是`(glob-style pattern, target)`，每个“模式”指向项目中的一个或一组文件，“目标”则表示实际安装位置，可以包含变量名，用花括号括起。例如，`MPTools`的`setup.cfg`文件可以是以下内容：

```
[files]
resources =
        config/mopy.cfg {confdir}/{application.name}/
        images/*.jpg    {datadir}/{application.name}/
```

`sysconfig`模块提供了一组可用的变量，并为不同的操作系统提供了默认值。例如，`{confdir}`在Linux下是`/etc`。安装工具就能结合`sysconfig`模块来决定数据文件的存放位置。最后，它会生成一个`RESOURCES`文件，这样`pkgutil`就能找到这些文件了：

![图14.5：安装工具](http://www.aosabook.org/images/packaging/installer.png)

图14.5：安装工具

### 14.4.4 改进PypI

上文提到过，PyPI目前是一个单点故障源。PEP 380中正式提出了这个问题，并定义了一个镜像协议，使得用户可以在PyPI出现问题时连接到其他源。这个协议的目的是让社区成员可以在世界各地搭建起PyPI镜像。

![图14.6：镜像](http://www.aosabook.org/images/packaging/mirroring.png)

图14.6：镜像

镜像列表的格式是`X.pypi.python.org`，其中`X`是一个字母序列，如`a,b,c,…,aa,ab,….`，`a.pypi.python.org`是主服务器，b字母开始是从服务器。域名`last.pypi.python.org`的A记录指向这个列表中的最后一个服务器，这样PyPI的使用者就能够根据DNS记录来获取完整的服务器镜像列表了。

比如，以下代码获取到的最后一个镜像地址是`h.pypi.python.org`，表示当前PyPI有7个镜像服务器（b至h）：

```python
>>> import socket
>>> socket.gethostbyname_ex('last.pypi.python.org')[0]
'h.pypi.python.org'
```

这样一来，客户端还可以根据域名的IP地址来决定连接最近的镜像服务器，或者在服务器发生故障时自动重连到新的地址。镜像协议本身要比rsync更复杂一些，因为我们需要保证下载统计量的准确性，并提供最基本的安全性保障。

#### 同步

镜像必须尽可能降低和主服务器之间的数据交换，要达到这个目的，就必须在PyPI的XML-RPC接口中加入`changelog`信息，以保证只获取变化的内容。对于每个软件包“P”，镜像必须复制`/simple/P/`和`/serversig/P`这两组信息。

如果中央服务器中删除了一个软件包，它就必须删除所有和它有关的数据。为了检测软件包文件的变动，可以缓存文件的ETag信息，并通过`If-None-Match`头来判断是否可以跳过传输过程。当同步完成后，镜像就将`/last-modified`文件设置为当前的时间。

#### 统计信息

当用户在镜像中下载一个软件包时，镜像就需要将这个事件报告给中央服务器，继而广播给其他镜像服务器。这样就能保证下载工具在任意镜像都能获得正确的下载量统计信息。

统计信息以CSV文件的格式保存在中央服务器的`stats`目录中，按照日和周分隔。每个镜像服务器需要提供一个`local-stats`目录来存放它自己的统计信息。文件中保存了每个软件包的下载数量，以及它们的下载工具。中央服务器每天都会从镜像服务器中获取这些信息，将其合并到全局的`stats`目录，这样就能保证镜像服务器中的`local-stats`目录中的数据至少是每日更新的。

#### 镜像服务器的合法性

在分布式的镜像系统中，客户端需要能够验证镜像服务器的合法性。如果不这样做，就可能产生以下威胁：

* 中央索引发生错误
* 镜像服务被篡改
* 服务器和客户端之间遭到拦截攻击

对于第一种攻击，软件包的作者就需要使用自己的PGP密钥来对软件包进行加密，这样其他用户就能判断他所下载的软件包是来自可信任的作者的。镜像服务协议中只对第二种攻击做了预防，不过有些措施也可以预防拦截攻击。

中央服务器会在`/serverkey`这个URL下提供一个DSA密钥，它是用`opensll dsa-pubout`<sup>3</sup>生成的PEM格式的密钥。这个URL不能被镜像服务器收录，客户端必须从主服务器中获取这个serverkey密钥，或者使用PyPI客户端本身自带的密钥。镜像服务器也是需要下载这个密钥的，用来检测密钥是否有更新。

对于每个软件包，`/serversig/package`中存放了它们的镜像签名。这是一个DSA签名，和URL`/simple/package`包含的内容对等，采用DER格式，是SHA-1和DSA的结合<sup>4</sup>。

客户端从镜像服务器下载软件包时必须经过以下验证：

1. 下载`/simple`页面，计算它的`SHA-1`哈希值。
2. 计算这个哈希值的DSA签名。
3. 下载对应的`/serversig`，将它和第二步中生成的签名进行比对。
4. 计算并验证所有下载文件的MD5值（和`/simple`页面中的内容对比）。

在从中央服务器下载软件包时不需要进行上述验证，客户端也不应该进行验证，以减少计算量。

这些密钥大约每隔一年会被更新一次。镜像服务器需要重新获取所有的`/serversig`页面内容，使用镜像服务的客户端也需要通过可靠的方式获取新密钥。一种做法是从`https://pypi.python.org/serverkey`下载。为了检测拦截攻击，客户端需要通过CAC认证中心验证服务端的SSL证书。

14.5 实施细节
-------------

上文提到的大多数改进方案都在`Distutils2`中实现了。`setup.py`文件已经退出历史舞台，取而代之的是`setup.cfg`，一个类似`.ini`类型的文件，它描述了项目的所有信息。这样做可以让打包人员方便地改变软件包的安装方式，而不需要接触Python语言。以下是一个配置文件的示例：

```
[metadata]
name = MPTools
version = 0.1
author = Tarek Ziade
author-email = tarek@mozilla.com
summary = Set of tools to build Mozilla Services apps
description-file = README
home-page = http://bitbucket.org/tarek/pypi2rpm
project-url: Repository, http://hg.mozilla.org/services/server-devtools
classifier = Development Status :: 3 - Alpha
    License :: OSI Approved :: Mozilla Public License 1.1 (MPL 1.1)

[files]
packages =
        mopytools
        mopytools.tests

extra_files =
        setup.py
        README
        build.py
        _build.py

resources =
    etc/mopytools.cfg {confdir}/mopytools
```

`Distutils2`会将这个文件用作于：

* 生成`META-1.2`格式的元信息，可以用作多种用途，如在PyPI上注册项目。
* 执行任何打包管理命令，如`sdist`。
* 安装一个以`Distutils2`为基础的项目。

`Distutils2`还通过`version`模块实现了`VERSION`元信息。

对`INSTALL-DB`元信息的实现会被包含在Python3.3的`pkgutil`模块中。在过度版本中，它的功能会由`Distutils2`完成。它所提供的API可以让我们浏览系统中已安装的项目。

以下是`Distutils2`提供的核心功能：

* 安装、卸载
* 依赖树

14.6 经验教训
-------------

### 14.6.1 PEP的重要性

要改变像Python打包系统这样庞大和复杂的架构必须通过谨慎地修改PEP标准来进行。据我所知，任何对PEP的修改和添加都要历经一年左右的时间。

社区中一直以来有个错误的做法：为了改善某个问题，就肆意扩展项目元信息，或是修改Python程序的安装方式，而不去尝试修订它所违背的PEP标准。

换句话说，根据你所使用的安装工具的不同，如`Distutils`和`Setuptools`，它们安装应用程序的方式就是不同的。这些工具的确解决了一些问题，但却会引发一连串的新问题。以操作系统的打包工具为例，管理员必须面对多个Python标准：官方文档所描述的标准，`Setuptools`强加给大家的标准。

但是，`Setuptools`能够有机会在实际环境中大范围地（在整个社区中）进行实验，创新的进度很快，得到的反馈信息也是无价的。我们可以据此撰写出更切合实际的PEP新标准。所以，很多时候我们需要能够察觉到某个第三方工具在为Python社区做出贡献，并应该起草一个新的PEP标准来解决它所提出的问题。

### 14.6.2 一个被纳入标准库的项目就已经死亡了一半

这个标题是援引Guido van Rossum的话，而事实上，Python的这种战争式的哲学也的确冲击了我们的努力成果。

`Distutils`是Python标准库之一，将来`Distutils2`也会成为标准库。一个被纳入标准库的项目很难再对其进行改造。虽然我们有正常的项目更新流程，即经过两个Python次版本就可以对某个API进行删改，但一旦某个API被发布，它必定会持续存在多年。

因此，对标准库中某个项目的一次修改并不是简单的bug修复，而很有可能影响整个生态系统。所以，当你需要进行重大更新时，就必须创建一个新的项目。

我之所以深有体会，就是因为在我对`Distutils`进行了超过一年的修改后，还是不得不回滚所有的代码，开启一个新的`Distutils2`项目。将来，如果我们的标准又一次发生了重大改变，很有可能会产生`Distutils3`项目，除非未来某一天标准库会作为独立的项目发行。

### 14.6.3 向前兼容

要改变Python项目的打包方式，其过程是非常漫长的：Python的生态系统中包含了那么多的项目，它们都采用旧的打包工具管理，一定会遇到诸多阻力。（文中一些章节描述的问题，我们花费了好几年才达成共识，而不是我之前预想的几个月。）对于Python3，可能会花费数年的时间才能将所有的项目都迁移到新的标准中去。

这也是为什么我们做的任何修改都必须兼容旧的打包工具，这是`Distutils2`编写过程中非常棘手的问题。

例如，一个以新标准进行打包的项目可能会依赖一个尚未采用新标准的其它项目，我们不能因此中断安装过程，并告知用户这是一个无法识别的依赖项。

举例来说，`INSTALL-DB`元信息的实现中会包含那些用`Distutils`、`Pip`、`Distribution`、或`Setuptools`安装的项目。`Distutils2`也会为那些使用`Distutils`安装的项目生成新的元信息。

14.7 参考和贡献者
-----------------

本文的部分章节直接摘自PEP文档，你可以在`http://python.org`中找到原文：

* PEP 241: Metadata for Python Software Packages 1.0: http://python.org/peps/pep-0214.html
* PEP 314: Metadata for Python Software Packages 1.1: http://python.org/peps/pep-0314.html
* PEP 345: Metadata for Python Software Packages 1.2: http://python.org/peps/pep-0345.html
* PEP 376: Database of Installed Python Distributions: http://python.org/peps/pep-0376.html
* PEP 381: Mirroring infrastructure for PyPI: http://python.org/peps/pep-0381.html
* PEP 386: Changing the version comparison module in Distutils: http://python.org/peps/pep-0386.html

在这里我想感谢所有为打包标准的制定做出贡献的人们，你可以在PEP中找到他们的名字。我还要特别感谢“打包别动队”的成员们。还要谢谢Alexis Metaireau、Toshio Kuratomi、Holger Krekel、以及Stefane Fermigier，感谢他们对本文提供的反馈。

本章中讨论的项目有：

* Distutils: http://docs.python.org/distutils
* Distutils2: http://packages.python.org/Distutils2
* Distribute: http://packages.python.org/distribute
* Setuptools: http://pypi.python.org/pypi/setuptools
* Pip: http://pypi.python.org/pypi/pip
* Virtualenv: http://pypi.python.org/pypi/virtualenv

脚注
---
1. 文中引用的Python改进提案（Python Enhancement Proposals，简称PEP）会在本文最后一节整理。
2. 过去被命名为CheeseShop
3. 即RFC 3280 SubjectPublishKeyInfo中定义的1.3.14.3.2.12算法。
4. 即RFC 3279 Dsa-Sig-Value中定义的1.2.840.10040.4.3算法。
