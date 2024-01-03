---
title: 开发人员必知的 5 种开源框架
date: 2016-03-13 16:25
categories: [Programming]
tags: [translation, bootstrap, angular, spring, spark, docker]
---

作者：[John Esposito](https://opensource.com/business/15/12/top-5-frameworks)

软件侵吞着世界[已经四年多了][1]，但开发人员看待软件的方式稍有不同。我们一直在致力于[解决实际问题][2]，而[很少思考软件开发的基石][3]。当问题变得更庞大、解决方案更复杂时，一些实用的、不怎么[产生泄漏][4]的抽象工具就显得越来越重要。

简单地来说，在那些追求生产效率的开发者眼中，*框架* 正在吞食着世界。那究竟是哪些框架、各自又在吞食着哪一部分呢？

开源界的开发框架实在太多了，多到近乎疯狂的地步。我从2015年各种领域的榜单中选取了最受欢迎的5种框架。对于前端框架（我所擅长的领域），我只选取那些真正的客户端框架，这是因为现今的浏览器和移动设备已经具备非常好的性能，越来越多的单页应用（SPA）正在避免和服务端交换数据。

1. 展现层：Bootstrap
---

我们从技术栈的顶端开始看——展现层，这一开发者和普通用户都会接触到的技术。展现层的赢家毫无疑问仍是[Bootstrap][101]。Bootstrap的[流行度][102]非常之惊人，[远远甩开][103]了它的老对手[Foundation][104]，以及新星[Material Design Lite][105]。在[BuiltWith][106]上，Bootstrap占据主导地位；而在GitHub上则长期保持[Star数][107]和[Fork数][108]最多的记录。

如今，Bootstrap仍然有着非常活跃的开发社区。8月，Bootstrap发布了[v4][109][内测版][110]，庆祝它的四岁生日。这个版本是对现有功能的[简化和扩充][111]，主要包括：增强可编程性；从Less迁移至Sass；将所有HTML重置代码集中到一个模块；大量自定义样式可直接通过Sass变量指定；所有JavaScript插件都改用ES6重写等。开发团队还开设了[官方主题市场][112]，进一步扩充现有的[主题生态][113]。

2. 网页MVC：AngularJS
---

随着网页平台技术越来越[成熟][201]，开发者们可以远离仍在使用标记语言进行着色的DOM对象，转而面对日渐完善的抽象层进行开发。这一趋势始于现代单页应用（SPA）对XMLHttpRequest的高度依赖，而其中[最][202][流行][203]的SPA框架当属[AngularJS][204]。

AngularJS有什么特别之处呢？一个词：指令（[directive][205]）。一个简单的`ng-`就能让标签“起死回生”（从静态的标记到动态的JS代码）。依赖注入也是很重要的功能，许多Angular特性都致力于简化维护成本，并进一步从DOM中抽象出来。其基本原则就是将声明式的展现层代码和命令式的领域逻辑充分隔离开来，这种做法对于使用过POM或ORM的人尤为熟悉（我们之中还有人体验过XAML）。这一思想令人振奋，解放了开发者，甚至让人第一眼看上去有些奇怪——因为它赋予了HTML所不该拥有的能力。

有些遗憾的是，AngualrJS的“杀手锏”双向绑定（让视图和模型数据保持一致）将在[Angular2][206]中移除，已经[临近公测][207]。虽然这一魔法般的特性即将消失，却带来了极大的性能提升，并降低了调试的难度（可以想象一下在悬崖边行走的感觉）。随着单页应用越来越庞大和复杂，这种权衡会变得更有价值。

<!-- more -->

3. 企业级Java：Spring Boot
---

Java的优点是什么？运行速度快，成熟，完善的类库，庞大的生态环境，一处编译处处执行，活跃的社区等等——除了痛苦的项目起始阶段。即便是最忠实的Java开发者也会转而使用Ruby或Python来快速编写一些只会用到一次的小型脚本（别不承认）。然而，鉴于以上种种原因，Java仍然是企业级应用的首选语言。

这时，Spring Boot出现了，它是模板代码的终结者，有了它，你就能在一条推文中写出一个Java应用程序来：

https://twitter.com/rob_winch/status/364871658483351552

```groovy
// spring run app.groovy
@Controller
class ThisWillActuallyRun {
  @RequestMapping("/")
  @ResponseBody
  String home() {
    "Hello World!"
  }
}
```

没有让人不快的XML配置，无需生成别扭的代码。这怎么可能？很简单，Spring Boot在背后做了很多工作，读上面的代码就能看出，框架会自动生成一个内嵌的servlet容器，监听8080端口，处理接收到的请求。这些都无需用户配置，而是遵从Spring Boot的约定。

Spring Boot有多流行？它是目前为止[fork数][301]和下载量最高的Spring应用（主框架除外）。2015年，[在谷歌上搜索Spring Boot的人数首次超过搜索Spring框架的人][302]。

4. 数据处理：Apache Spark
---

很久很久以前（2004年），谷歌研发出一种编程模型（[MapReduce][401]），将分布式批处理任务通用化了，并撰写了一篇[著名的论文][402]。之后，Yahoo的工程师用Java编写了一个框架（[Hadoop][403]）实现了MapReduce，以及一个[分布式文件系统][404]，使得MapReduce任务能够更简便地读写数据。

将近十年的时间，Hadoop主宰了大数据处理框架的生态系统，即使批处理只能解决有限的问题——也许大多数企业和科学工作者已经习惯了大数据量的批处理分析吧。然而，并不是所有大型数据集都适合使用批处理。特别地，流式数据（如传感器数据）和迭代式数据分析（机器学习算法中最为常见）都不适合使用批处理。因此，大数据领域诞生了很多[新的编程模型、应用架构][405]、以及各类[数据存储][406]也逐渐流行开来（甚至还从MapReduce中分离出了一种[新的集群管理系统][407]）。

在这些新兴的系统之中，[伯克利AMPLab][408]研发的[Apache Spark][409]在2015年脱颖而出。各类调研和报告（[DZone][410]、[Databricks][411]、[Typesafe][412]）都显示Spark的成长速度非常之快。GitHub提交数从2013年开始就呈[线性增长][413]，而谷歌趋势则在2015年呈现出[指数级的增长][414]。

Spark如此流行，它究竟是做什么的呢？答案很简单，非常快速的批处理，不过这点是构建在Spark的一个杀手级特性之上的，能够应用到比Hadoop多得多的编程模型中。Spark将数据表达为弹性分布式数据集（RDD），处理结果保存在多个节点的内存中，不进行复制，只是记录数据的计算过程（这点可以和[CQRS][415]、[实用主义][416]、[Kolmogorov复杂度][417]相较）。这一特点可以让迭代算法无需从底层（较慢的）分布式存储层读取数据。同时也意味着批处理流程无需再背负Nathan Marz在[Lambda架构][418]中所描述的“数据卡顿”之恶名。RDD还能让Spark模拟实时流数据处理，通过将数据切分成小块，降低延迟时间，达到大部分应用对“准实时”的要求。

5. 软件交付：Docker
---

严格意义上说，[Docker][501]并不是符合“框架”的定义：代码库，通用性好，使用一系列特殊约定来解决大型重复性问题。但是，如果框架指的是能够让程序员在一种舒适的抽象层之上进行编码，那Docker将是框架中的佼佼者（我们可以称它为“外壳型框架”，多制造一些命名上的混乱吧）。而且，如果将本文的标题改为“开发人员必知的5样东西”，又不把Docker包含进来，就显得太奇怪了。

为什么Docker很出色？首先我们应该问什么容器很受欢迎（FreeBSD Jail, Solaries Zones, OpenVZ, LXC）？很简单：无需使用一个完整的操作系统来实现隔离；或者说，在获得安全性和便利性的提升时，无需承担额外的开销。但是，隔离也有很多种形式（比如最先想到的`chroot`，以及各种虚拟内存技术），而且可以非常方便地用`systemd-nspawn`来启动程序，而不使用Docker。然而，仅仅隔离进程还不够，那Docker有什么[过人之处][502]呢？

两个原因：Dockefile（新型的tar包）增加了便携性；它的格式成为了现行的标准。第一个原因使得应用程序交付变得容易（之前人们是通过创建轻型虚拟机来实现的）。第二个原因则让容器更容易分享（不单是[DockerHub][503]）。我可以很容易地尝试你编写的应用程序，而不是先要做些不相关的事（想想`apt-get`给你的体验）。

关于作者
---

![John Esposito](https://opensource.com/sites/default/files/styles/profile_pictures/public/pictures/john-esposito.jpg?itok=xPVFPzr2)

John Esposito是DZone的主编，最近刚刚完成古典学博士学位的学习，养了两只猫。之前他是VBA和Force.com的开发者、DBA、网络管理员。（但说真的，Lisp是最棒的！）

[1]: http://www.wsj.com/articles/SB10001424053111903480904576512250915629460
[2]: http://www.dougengelbart.org/pubs/augment-3906.html
[3]: http://worrydream.com/refs/Brooks-NoSilverBullet.pdf
[4]: http://www.joelonsoftware.com/articles/LeakyAbstractions.html

[101]: http://getbootstrap.com/
[102]: https://www.google.com/trends/explore#q=%2Fm%2F0j671ln
[103]: https://www.google.com/trends/explore#q=%2Fm%2F0j671ln%2C%20%2Fm%2F0ll4n18%2C%20Material%20Design%20Lite&cmpt=q&tz=Etc%2FGMT%2B5
[104]: http://foundation.zurb.com/
[105]: http://www.getmdl.io/
[106]: http://trends.builtwith.com/docinfo/Twitter-Bootstrap
[107]: https://github.com/search?q=stars:%3E1&s=stars&type=Repositories
[108]: https://github.com/search?o=desc&q=stars:%3E1&s=forks&type=Repositories
[109]: http://v4-alpha.getbootstrap.com/
[110]: http://blog.getbootstrap.com/2015/08/19/bootstrap-4-alpha/
[111]: http://v4-alpha.getbootstrap.com/migration/
[112]: http://themes.getbootstrap.com/
[113]: https://www.google.com/search?q=bootstrap+theme+sites

[201]: https://www.w3.org/blog/news/
[202]: https://www.google.com/trends/explore#q=%2Fm%2F0j45p7w%2C%20EmberJS%2C%20MeteorJS%2C%20BackboneJS&cmpt=q&tz=Etc%2FGMT%2B5
[203]: https://www.pluralsight.com/browse#tab-courses-popular
[204]: https://angularjs.org/
[205]: https://docs.angularjs.org/guide/directive
[206]: https://www.quora.com/Why-is-the-two-way-data-binding-being-dropped-in-Angular-2
[207]: http://angularjs.blogspot.com/2015/11/highlights-from-angularconnect-2015.html

[301]: https://github.com/spring-projects
[302]: https://www.google.com/trends/explore#q=spring%20boot%2C%20spring%20framework&cmpt=q&tz=Etc%2FGMT%2B5

[401]: http://ayende.com/blog/4435/map-reduce-a-visual-explanation
[402]: http://static.googleusercontent.com/media/research.google.com/en//archive/mapreduce-osdi04.pdf
[403]: https://hadoop.apache.org/
[404]: https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsUserGuide.html
[405]: https://www.linkedin.com/pulse/100-open-source-big-data-architecture-papers-anil-madan
[406]: http://www.journalofbigdata.com/content/2/1/18
[407]: https://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-site/YARN.html
[408]: http://spark.apache.org/research.html
[409]: http://spark.apache.org/
[410]: https://dzone.com/guides/big-data-business-intelligence-and-analytics-2015
[411]: http://cdn2.hubspot.net/hubfs/438089/DataBricks_Surveys_-_Content/Spark-Survey-2015-Infographic.pdf
[412]: https://info.typesafe.com/COLL-20XX-Spark-Survey-Report_LP.html?lst=PR&lsd=COLL-20XX-Spark-Survey-Trends-Adoption-Report
[413]: https://github.com/apache/spark/graphs/contributors
[414]: https://www.google.com/trends/explore#q=%2Fm%2F0ndhxqz
[415]: http://martinfowler.com/bliki/CQRS.html
[416]: http://plato.stanford.edu/entries/peirce/
[417]: http://people.cs.uchicago.edu/~fortnow/papers/kaikoura.pdf
[418]: http://lambda-architecture.net/

[501]: https://www.docker.com/
[502]: http://techapostle.blogspot.com/2015/04/the-3-reasons-why-docker-got-it-right.html
[503]: https://hub.docker.com/
