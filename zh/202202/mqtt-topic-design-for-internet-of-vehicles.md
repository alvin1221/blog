> 本文作者：田桢，前上汽大众平台架构师，现为中科创达汽车云技术负责人

## 前言

在车联网生态中，TSP（Telematics Service Provider）平台在产业链中居于核心地位，上接汽车、车载设备制造商与网络运营商，下接内容提供商，是主机厂车辆与服务的核心数据连接平台。随着智能汽车的发展和车主用户对应用场景需求的不断提升，主机厂对 TSP 平台的设备与应用承载能力需求将不断增加。

在之前的文章《[车联网场景中的 MQTT 协议](https://www.emqx.com/zh/blog/mqtt-for-internet-of-vehicles)》我们提到，在车载设备与 TSP 平台数据交互协议选择上，[MQTT](https://www.emqx.com/zh/mqtt) 以其轻量化、易扩展、多种消息质量保证（QoS），以及通过发布订阅模式实现数据产生与数据消费系统解偶等优势成为了目前各大主机厂的新一代 TSP 平台的首选协议。

本文我们将介绍在车联网 TSP 平台搭建过程中，如何进行 [MQTT 消息主题](https://www.emqx.com/zh/blog/advanced-features-of-mqtt-topics)设计。

## 车联网 TSP 场景中对消息通道的需求

车联网 TSP 场景中，MQTT 协议作为「车-平台-应用」之间的业务消息通道，不仅要保证车与应用之间消息可以双向互通互联，而且需要通过一定规则将不同类型的消息识别与分发。而 MQTT 协议中的主题就是这些消息的标签，也可以看作是业务通道。

在车联网场景中，可以把消息分为从车-平台-应用的数据上行通道以及应用-平台-车的数据下行通道；对于车联网 TSP 平台，不同数据方向意味着不同的业务类型，需要通过 MQTT 主题进行明确的区分与隔离。

- 从车端角度看：

	在 TSP 平台中车辆数据上报是上行数据的主要业务类型。随着车联网业务的不断丰富，如 T-box 等车载系统计算能力与通讯能力不断增强，车辆数据上报的业务场景、数据量及频率也不断增加。基于业务隔离、实时性与安全等需求，从车联网早期的一车一主题逐渐向一车多消息通道发展。

- 从应用侧角度看：

	平台应用作为车辆数据接收与消费方，同时也会作为数据下发，指令下发的消息发送方。根据业务需求不同，消息发送类型也可以分为：

	1. 一对一消息：针对一些如车控㩐关键业务与高安全性要求的业务，需要针对每辆车提供一对一的消息通道。
	2. 一对多消息：对于某一类业务或者某一种车型，可以通过相同主题通道向车机设备进行指令与数据下发。
	3. 消息广播：针对大规模的消息通知，配置更新场景，可以向平台所连设备发送大规模的消息广播。 

## 什么是 MQTT 协议的主题

### 基础概念

在 MQTT 协议通信机制中有三个角色： 消息发布者（publisher）、代理服务器（broker）和消息订阅者（subscriber）。消息从发布者发送到代理服务器，然后被订阅者接收，而主题就是发布者与订阅者之间约定的消息通道。                                            

![车联网中的 MQTT](https://assets.emqx.com/images/27e6f4979969dcf62ce88e8fe9c46ba2.png)

发布者指定的主题发送消息，订阅者从指定的主题订阅接收消息，而 Broker 则起到按照主题接受并分发消息的代理人。在车联网 TSP 平台场景中，车载设备、移动终端与业务应用都可以被看作是 [MQTT 客户端](https://www.emqx.io/zh/mqtt-client)。根据业务不同与数据方向不同，车载设备、移动终端与业务应用的角色也会在发布者与订阅者之间切换。

### 主题的定义与规范

MQTT 协议中规定了主题是一段 UTF-8 编码的字符串，主题需要满足以下规则：

- 所有的主题名和主题过滤器必须至少包含一个字符。
- 主题名和主题过滤器是大小写敏感的。如：ACCOUNTS 和 Accounts 是不同的主题名。
- 主题名和主题过滤器可以包含空格字符。如：Accounts payable 是合法的主题名
- 主题名或主题过滤器以前置或后置斜杠 / 区分。如：/finance 和 finance 是不同的。
- 只包含斜杠 / 的主题名或主题过滤器是合法的。
- 主题名和主题过滤器不能包含 null 字符(Unicode U+0000)。
- 主题名和主题过滤器是 UTF-8 编码字符串，除了不能超过 UTF-8 编码字符串的长度限制之外，主题名或主题过滤器的层级数量没有其它限制。

### 主题层级

MQTT 协议主题可以通过斜杠（“/” U+002F）将主题分割成多个层级；作为消息通道，客户端可以通过定义主题层级来实现对消息类型的细分；

例如：一个主机厂有多个车型，每个车型下面有多个车联网业务，我们在定义车机向对某个车型业务系统发消息时可以向<车型A>/ <车辆唯一标识>/<业务X>主题发消息；

当然在 MQTT 世界中主题可以有很多层（MQTT 协议中没有限制层级数量），比如：<车型A>/<车辆唯一标识(车架号)>/<业务X>/<子业务1>

这样，我们在定义车联网分层级的业务通道的时候可以按主题层级来设计。

### 通配符

MQTT 协议中订阅者的订阅的主题过滤器可以包含特殊的通配符，允许客户端一次订阅多个主题。

- 多层通配符

	\#字符号（“#” U+0023）是用于匹配主题中任意层级的通配符。多层通配符表示它的父级和任意数量的子层级。如：订阅者可以通过订阅<车型A>/# 接收到：

	<车型A>

	<车型A>/<车架号1>

	<车型A>/<车架号1>/<业务X>

	这几类主题的消息。

- 单层通配符

	加号 (“+” U+002B) 用于单个主题层级匹配的通配符。如：订阅者可以通过订阅<车型A>/+ 来接收

	<车型A>/<车架号1>

	<车型A>/<车架号2>

	不同于多层通配符，使用单层通配符的时候无法匹配子层级的主题，比如：<车型A>/<车架号1>/<业务X>的主题消息就无法接收到。

## 车联网 TSP 平台主题设计原则最佳实践

前文中我们提到在车联网场景中 MQTT 主题定义了业务与数据的通道，主题定义的核心是区分业务场景。如何合理的定义主题，需要根据一定原则来设计。我们可以从以下几个维度来设计与定义主题：

### 根据业务数据方向区分

首先，数据的上下行方向不同决定了数据由谁产生，被谁消费。在车联网场景中，车载设备到平台的数据上行通道与平台应用到车的下行数据需要通过主题分开。通过对上行、下行主题的设计区分，可以帮助设计、运维及业务人员快速定位场景、问题及相关干系方。

有些业务可能会同时用到上下行主题，比如车辆申请数据下发后平台下发数据，以及平台请求车辆上班数据后车辆上报数据。这种情况下，由于 MQTT 协议的异步通讯机制，也需要对一个整体业务的上下行主题分别定义。 

### 根据车型区分

在车联网场景中，不同车型意味着车辆产生的数据不完全相同，车机能力不完全相同同，对接的业务应用也不尽相同。我们可以根据车型型号对差异化的车辆数据以及业务进行主题上的区分。当然，同一个主机厂下的不同车型也会有相同的业务和数据，这些业务可以通过跨车型的主题来定义。

### 根据车辆区分

在车联网场景中，如车控等安全等级较高的业务场景往往需要一对一的主题作为数据通通道。一方面通过主题来隔离车辆与车辆之间的业务信息，另一方面保证数据可以点对点的交互。在主题设计中，有时需要将车辆的唯一标识符作为主题的一部分来实现一对一的消息通道。常见的方案有使用车辆 VIN 码作为主题的一部分。

### 根据用户区分

在实际使用场景中，也存在需要根据用户（而不是车辆）实现车云的一对一的消息通道，此类需求经常发生在用户促销、运营、ToB 业务等场景中。在主题设计时，常见的方案有两种，一是使用用户 ID 作为主题的一部分；二是通过人-车关系转换成车辆级主题，但由于消息时效性、车内用户登录状态等原因，此方案下生产端及消费端均需要添加额外的设计及处理，相对复杂。

### 根据研发环境区分

从项目工程实施角度出发，一般在主题设计时同时会添加环境变量，通过配置实现不同研发环境下的资源复用以及正确性检查。

### 根据数据吞吐量区分

由于业务的不同，不管是上行数据还是下行数据，数据的发送频率与报文大小都不尽相同。不同的数据吞吐量会影响到消费端的处理以及架构设计，比如我们在处理高频的车辆数据上报业务时往往要考虑应用层的消费能力，这时候可能要借助类似 Kafka 之类的高吞吐消息队列来进行数据缓冲，防止应用消费不及时造成数据积压与数据丢失。所以在 MQTT 主题定义上，我们往往也需要对不同数据吞吐量的业务进行区分。

## MQTT 协议主题设计在车联网场景中的应用

### 车辆数据主动上报

车载设备（T-box，车机等）作为车辆运行数据的收集者，基于固定频率将车内各类控制器、传感器等数据打包发送到平台端。此类数据一般可以按照上报数据的车型、车架号、业务数据类型等多个层级进行设计。

例如在用户同意的前提下，车辆在行驶过程中会将位置、车速、电量等信息按照固定频率上报云平台，云端应用基于这些数据，提供位置查找、超速提醒、电量提醒、地理围栏服务给终端用户使用。

### 平台请求下发后车辆数据上报

当云平台需要获取车辆的最新状态及信息时，可以主动下发命令要求车辆上报数据。此类场景一般可以按照车架号、业务类型等层级进行主题设计。

例如在诊断场景下，平台通过 MQTT 下发诊断命令至车辆，当车内各设备完成诊断操作后，会将诊断数据打包后上报至云平台，车辆诊断工程师将根据采集到的诊断数据对于车况进行整体的分析及问题定位。

### 平台指令下发

车辆远程控制是车联网业务中最常见、最典型的场景，各主机厂均在手机 App 中提供各种远控功能，例如远程启动、远程开车门、远程闪灯鸣笛等等。此类场景下，手机 App 发送控制命令至云平台，平台应用经过权限检查、安全检查等一系列操作后，通过 MQTT 将命令下发至车辆执行，车辆端执行成功后，异步通知平台执行结果。

此类场景一般可以按照上行下行、车架号、业务类型、操作类型等多个层级进行主题设计。

### 车辆客户端请求后平台数据下发

在 SDV（软件定义汽车）的大背景下，车内很多配置是可以做到动态变化的，例如数据采集规则、安全访问规则，所以车辆在点火启动后，会主动请求平台最新的相关配置，若两侧配置不一致，平台侧会下发最新的配置信息至车辆，车辆侧实时生效。

此类场景一般可以按照上行下行、车架号、业务类型等多个层级进行主题设计。 

## 使用 EMQX 进行车联网 TSP 平台主题设计

[EMQX](https://www.emqx.io/zh) 作为全球领先的 MQTT 物联网消息中间件，基于分布式集群、大规模并发连接、快速低延时的消息路由等突出特性，能够有效处理车联网场景中高时效性业务需求，大幅度缩短端到端时延，为大规模车联网平台快速部署提供标准的 [MQTT 服务](https://www.emqx.com/zh/cloud)。

### EMQX 在车联网场景下的优势

#### 海量主题支持

随着车联网场景中的业务不断增加，承载业务通道的主题数量也不断增加，尤其是包括车控场景所需要的一车一主题、一车多主题需求越来越大。在这种背景下，[MQTT 服务器](https://www.emqx.com/zh/blog/mqtt-broker-server)的主题数承载能力就成为了 TSP 平台的重要评估指标。

EMQX 在一开始的底层设计中就规划了对海量设备连接与海量主题支持的能力。常见的 16 核 32G 内存的 3 节点 EMQX 集群可以支持百万级主题同时运行，为 TSP 平台主题设计提供了灵活的设计空间。

#### 强大规则引擎

EMQX 提供了内置的规则引擎， 基于规则引擎可以提供对不同主题数据的查找、过滤、数据分拆以及对消息重新路由。使用规则引擎，我们可以在已有车载设备与应用主题建立好的场景下，通过创建新的路由规则与数据预处理规则对已有主题中的数据进行再处理。在车辆上市后，通过在平台侧定义新规则实现对新业务应用的支持。

在 [EMQX 企业版](https://www.emqx.com/zh/products/emqx)中，规则引擎提供了数据持久化对接能力，可以通过规则引擎中的配置将不同主题中的数据直接对接不同持久化方案。比如对数据吞吐量比较高的数据可以通过规则引擎对接 Kafka、Apache Pulsar 等高吞吐消息队列进行数据缓冲；而车辆报警等小吞吐低时延主题数据可以直接对接应用，实现数据的快速路由消费。

#### 代理订阅功能

 EMQX 提供了代理订阅功能，客户端在连接建立时，不需要发送额外的 SUBSCRIBE 报文，便能自动建立用户预设的订阅关系。这样可以让平台侧直接管理车载设备的主题订阅关系，方便平台侧进行统一管理。    

#### 丰富的主题监控与慢订阅统计

EMQX 企业版提供了以主题为监控维度的运行数据监控，可以在 EMQX 可视化 Dashboard 中清晰看到主题下消息流入流出、丢弃的总数和当前速率。

自 4.4 版本起，EMQX 提供了对慢订阅的统计。该功能会追踪 QoS1 和 QoS2 消息到达 EMQX 后，完成消息传输全流程的时间消耗，然后采用指数移动平均算法，计算该订阅者的平均消息传输时延，之后按照时延高低对订阅者进行统计排名。

通过在 TSP 平台运营过程中不断监控各种主题的数据接收与消费情况，平台运营者就可以根据业务变化不断调整平台业务设计与应用设计，实现平台的不断优化扩展。

### 需要注意的事项

我们在使用 EMQX 作为车联网 TSP 平台 MQTT Broker 时，在设计主题的过程中需要注意以下几个问题：

- 通配符使用与主题数层级

	由于 EMQX 采用主题树的数据结构对主题进行过滤匹配。在使用通配符来匹配多个主题的场景下，如果主题层级非常多，就会对 EMQX 产生比较大的资源消耗。所以在主题设计时，不建议层级太多，一般不建议超过5层。

- 主题与内存的消耗

	由于在 EMQX 中主题数与主题长度主要与内存相关，我们在承载大量主题的同时也要重点监控 EMQX 集群内存的用量。

## 总结

随着 MQTT 协议在车联网业务中的广泛普及，车联网 TSP 平台的 MQTT 消息主题设计将是各主机厂与 TSP 平台方案供应商必须面对的课题。本文是我们结合多年 TSP 平台建设经验，针对车联网业务从多维度总结的 MQTT 主题设计思路，希望能够在平台前期设计与业务扩展阶段给行业同仁一些帮助与启发。


<section class="promotion">
    <div>
        免费试用 EMQX Cloud
        <div class="is-size-14 is-text-normal has-text-weight-normal">全托管的云原生 MQTT 消息服务</div>
    </div>
    <a href="https://accounts-zh.emqx.com/signup?continue=https://cloud.emqx.com/console/deployments/0?oper=new" class="button is-gradient px-5">开始试用 →</a >
</section>


## 本系列中的其它文章

- [车联网平台搭建从入门到精通 01 | 车联网场景中的 MQTT 协议](https://www.emqx.com/zh/blog/mqtt-for-internet-of-vehicles)
- [车联网平台搭建从入门到精通 02 | 千万级车联网 MQTT 消息平台架构设计](https://www.emqx.com/zh/blog/mqtt-messaging-platform-for-internet-of-vehicles)
- [车联网平台搭建从入门到精通 04 | MQTT QoS 设计：车联网平台消息传输质量保障](https://www.emqx.com/zh/blog/mqtt-qos-design-for-internet-of-vehicles)
- [车联网平台搭建从入门到精通 05 | 车联网平台百万级消息吞吐架构设计](https://www.emqx.com/zh/blog/million-level-message-throughput-architecture-design-for-internet-of-vehicles)
- [车联网平台搭建从入门到精通 06 | 车联网通信安全之 SSL/TLS 协议](https://www.emqx.com/zh/blog/ssl-tls-for-internet-of-vehicles-communication-security)
- [车联网平台搭建从入门到精通 07 | 国密在车联网安全认证场景中的应用](https://www.emqx.com/zh/blog/application-of-gmsm-in-internet-of-vehicles-security-authentication-scenario)
- [车联网平台搭建从入门到精通 08 | 云原生赋能智能网联汽车消息处理基础框架构建](https://www.emqx.com/zh/blog/cloud-native-smart-connected-car-messaging)
