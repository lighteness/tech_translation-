# 教你如何在10分钟内用Prometheus和Grafana监控你的Django应用

在这篇10分钟的教程里，**我将教你如何用Prometheus和Grafana监控一个Django应用**。你将获得如下的**监控面板**：

。。。图1

**[Prometheus](https://prometheus.io/)是一个监控解决方案**，它能从目标端那边收集**监控数据（metrics）**。Prometheus是一个**开源项目**，最初从SoundCloud公司内部发起，这所公司是由几个前Google工程师所创建。Prometheus的初衷是为了能够监控高度动态的容器环境。自从2012年它问世以来，许多公司和组织就开始采纳它。

**[Grafana](https://grafana.com/)是一个能够展示统计图像的Web**，它有着非常优秀的展示能力，并且图表、面板都可配置化。当你有了数据，你可以通过Grafana来展示统计，它支持非常多的数据来源，当然也支持来源自Prometheus的数据。


## 为什么我选择用Prometheus搭配Grafana来监控我的Django应用？

2个月前，我在我们生产环境里的web应用上加了一条新的安全规则。这条安全规则在测试环境中通过测试，并未引起任何问题。之后部署到生产环境里，刚开始也一切正常。但是直到几个小时过后，有一个用户反馈无法访问我们这个web，我们也发现正是由于这个新加的规则，一些API返回了“503 Server Unavailable”错误。
那时，我们还没有一个监控系统去监控我们的应用，生产上发生事故，直到有用户告诉我们，我们才知道。
像是在Google这样级别的大公司，每小时营收大约在1千5百万美元左右，你可以想象一下，产线上发生这种小事故，将会给公司带来多大的损失！

## Prometheus是如何工作的？
作为监控服务，Prometheus监控每一项具体的事，可以是整个Linux服务器，可以是单独的一个Apache服务器，可以单单是一个进程，一个数据库，亦或是其他任何你想监控的系统单元。
在Prometheus世界里，有两件事要明确，一是Prometheus服务本身，二是监控目标。


。。。图2


为了能够使用Prometheus监控你的服务，你的服务需要**通过endpoint暴露给Prometheus**，暴露的endpoint里，会有一组监控数据名字和其当前的值。

为了方便用户设置endpoint和暴露监控数据，Prometheus还提供了了**一组客户端库**。

Prometheus官方支持的客户端库有这4种语言：**Go，Java/Scala，Python和Ruby**。

Prometheus本身提供一个简单的UI界面，你可以通过Prometheus支持的查询语言PromQL，亲自手写一个，去查询你想要的监控数据。

在这里，我们使用上面提及的**Grafana**，Grafana是来自第三方的组件，作为展现层，**将存储在Prometheus时序数据库里的数据可视化**。

你不需要编写PromQL查询语句，直接与Prometheus打交道，你可以使用Grafana UI面板，从Prometheus服务器那边查询到监控数据，并且把它们在Grafana上展现出来。我们马上动手试一试。

现在，我们开始吧。


## 第1步: 把你的Prometheus和Grafana跑起来

在此之前，你需要先安装[Docker](https://docs.docker.com/engine/docker-overview/)。在这个教程里，我会使用Docker镜像，但是你也可以用其他方式。

Docker容器镜像是一种轻量、独立、可执行的软件打包方式，它将所需要的一切：代码，运行时，系统工具，系统库和设置，都打包在一起。想要活得更多关于Docker的信息，你可以点击连接：https://docs.docker.com/engine/docker-overview/

最简单的方式去运行一个Prometheus服务就是用Prometheus的Docker镜像。
这个镜像，由Prometheus社区维护：https://hub.docker.com/r/prom/prometheus/，你可以直接从docker hub上拉取这个镜像

为了能够让Grafana跑起来，你可以用下面的这个镜像：[grafana/grafana:6.5.2](https://hub.docker.com/r/grafana/grafana)

接下来，你得写一个docker-compose文件来包含对这两个镜像的配置

### docker-compose.monitoring.yml


```
version: ‘3.6’
services:
prometheus:
 image: prom/prometheus:v2.14.0
 volumes:
 — ./prometheus/:/etc/prometheus/
 command:
 — ‘ — config.file=/etc/prometheus/prometheus.yml’
 ports:
 — 9090:9090
grafana:
 image: grafana/grafana:6.5.2
 ports:
 — 3060:3000
```

## 第2步：配置你的Prometheus

为了能够让Prometheus跑起来，你需要创建一份prometheus.yml文件。

我把这个文件放在了./prometheus/目录下，同时这个目录里也放了刚刚创建的docker-compose.monitoring.yml文件

### prometheus.yml
```
global:
 scrape_interval: 15s
 evaluation_interval: 15s
rule_files:
 # — “first.rules”
 # — “second.rules”
scrape_configs:
 — job_name: monitoring
 static_configs:
 — targets:
    — host.docker.internal
```

`host.docker.internal`用来获取目标监控对象的IP地址。你可以从这里找到更多的信息：https://docs.docker.com/docker-for-windows/networking。
这里的url是仅用作开发目的。如果你面对的是在生产环境中部署好的应用，那么在生产环境中，你也应该拥有一个prometheus服务器。并且你需要修改相应配置文件prometheus.yml，将这个`host.docker.internal`改成生产环境中目标机器的域名地址，比如`your-prod-application.com`


为了能够让Prometheus和Grafana跑起来，你需要在你的命令行终端，敲上这个命令：

`$ docker-compose -f docker-compose.monitoring.yml up -d`

那么你将在浏览器里输入localhost:9090后，看到下面的东西


。。。。图片4


此时此刻，endpoint/metrics还不存在，这也是为什么目标返回404错误。我们将在下一步看到，如何在我们的Django里，创建一个endpoint/metrics


## 第3步: 添点东西，暴露你Django应用的监控数据

Prometheus社区已经为许多技术栈（Nodejs，Spring-Boot，Django等等。。）准备好了许多的开源的第三方库，你可以直接拿来使用。

其中之一就有[Django-prometheus](https://github.com/korfuri/django-prometheus/)，它可以将Django的监控数据暴露给Prometheus。

在你的django项目里安装django-prometheus:

`pip install django-prometheus`

在你的配置项里，增加下面内容：

### settings.py

```
BASE_INSTALLED_APPS = [
 ...
 “django_prometheus”，
]
MIDDLEWARE = [
 “django_prometheus.middleware.PrometheusBeforeMiddleware”，
...
 “django_prometheus.middleware.PrometheusAfterMiddleware”，
]
```

那么暴露给Prometheus的endpoints设置完成，这个url地址会是localhost/metrics。


### urls.py

`url(“”， include(“django_prometheus.urls”))，`

现在你可以到，你的prometheus targets对象会是"UP"状态：http://localhost:9090/targets

。。。图5


## 第4步：创建你的Grafana面板

现在你已经有了一个活着的Prometheus targets对象，你可以用Grafana创建让人兴奋的监控面板。

输入http://localhost:3060，你可以看到Grafana。

添加你的Prometheus实例作为一个数据源（如下GIF动图显示）

请点击左边的Configuration配置项，然后点击Data Sources数据源，然后点击添加一个数据源，然后输入名字，输入你数据源的url地址。

。。。。图6


一旦上面的完成，你就可以创建一个新的监控面板：

1. 点击graph title，然后点击“Edit”
2. 在“Metrics”tab页，选择你的Prometheus数据源（页面右下角）
3. 在“Query”一栏，输入任何的Prometheus查询表达式，并且使用“Metric”选项去完成补全

可以的话，你可以用一些label标签去完成进一步的过滤

。。。。图7

Grafana社区非常的活跃，提供了非常多的dashboards面板供你下载，其中许多就是基于Prometheus数据源。如果你的被监控组件是常见的那种，那么你可以非常快的搭建起来。


## 结论

你可以看到，**只要把Grafana的数据源设定为Prometheus，Grafana就能使得开发者在Grafana里添加上他们想要的统计数据，获得对他们而言有价值的数据**：比如http响应码，每页的延迟，程序部署次数等。。。

为了做到这些，你还得使用第三方的库来设置一个endpoint，用来暴露你想要的统计数据。配置文件prometheus.yml是为了告诉你的Prometheus实例，endpoint在哪里，如何连接。

这部分做完后，把Prometheus实例的url地址告诉给Grafana，以完成数据源的添加。

现在一切都已设置完成，你应该有了Prometheus和Grafana，它们正在监控你的Django应用。

## 后续
你已经注意到，我们已经在本地把我们的Grafana和Prometheus跑起来了。一旦你做到这点，你也准备好用你自己的方式去[部署你的Grafana和Prometheus docker镜像](https://www.katacoda.com/courses/docker/deploying-first-container)

我希望这10分钟的教程可以帮到你，让你方便、快速地使用Prometheus和Grafana去监控一个Django应用。这边有一些资源，你可看看：

- [CLIENT LIBRARIES](https://prometheus.io/docs/instrumenting/clientlibs/)
- [Deploying Your First Docker Container | Docker | Katacoda](https://www.katacoda.com/courses/docker/deploying-first-container)
- [Grafana: The open observability platform](https://www.docker.com/resources/what-container)
