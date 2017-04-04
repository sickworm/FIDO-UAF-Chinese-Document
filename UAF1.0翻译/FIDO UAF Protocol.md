# FIDO UAF  Authenticator Commands
FIDO联盟实施草案 2015年12月15日

**本版本**
[https://fidoalliance.org/specs/fido-uaf-v1.2-id-20151215/fido-uaf-protocol-v1.2-id-20151215.html](https://fidoalliance.org/specs/fido-uaf-v1.2-id-20151215/fido-uaf-protocol-v1.2-id-20151215.html)

**上一版本**
[https://fidoalliance.org/specs/fido-uaf-v1.0-ps-20141208/fido-uaf-protocol-v1.2-id-20151215.html](https://fidoalliance.org/specs/fido-uaf-v1.0-ps-20141208/fido-uaf-protocol-v1.2-id-20151215.html)

**编辑：**
Dr. Rolf Lindemann, Nok Nok Labs, Inc.  
Eric Tiffany, FIDO Alliance

**贡献者:**
Davit Baghdasaryan, Nok Nok Labs, Inc.
Dirk Balfanz, Google, Inc.
Brad Hill, PayPal, Inc.
Jeff Hodges, PayPal, Inc.

Copyright © 2013-2015 [FIDO Alliance](https://www.fidoalliance.org/) All Rights Reserved.

# 摘要
通用认证框架（Universal Authentication Framework，简称UAF）提供了一个统一的可扩展的认证机制，以代替密码，同时避免当前选择的身份验证方法的缺点。

该方法被设计为允许依赖方为特定的终端用户或交互选择最适合的认证机制，同时考虑到在未来利用新兴设备安全功能的可能性，而不用再做额外的集成工作。

本文档描述了FIDO架构的细节，FIDO架构定义了所有UAF协议message的处理流程和内容，并阐述了其设计背后的基本原理。

# 本文档状态
*本章节描述了本文档在发布时的状态。可能会有其他文档代替本文档。FIDO联盟发布过的以及最新的技术报告在[FIDO Alliance specifications index](https://fidoalliance.org/specifications/)的*https://www.fidoalliance.org/specifications/*都可以找到。

本文档作为实施草案由 [FIDO Alliance](https://www.fidoalliance.org/)发布。本文档旨在成为FIDO联盟的建议标准。如果你对本文档有任何意见或建议，请[联系我们](https://fidoalliance.org/contact/)。我们欢迎任何意见和建议。

**本实施草案已由FIDO Alliance, Inc.编制完成。**仅授权用于实现规范的目的。无权编制本规范的衍生作品。个人或集体想要获得复制本规范做其他用途的许可，必须联系FIDO联盟以确认相应的许可证是否允许这种用途。

本规范某些要素的实现可能需要第三方知识产权的许可，包括但不限于专利权。FIDO Alliance, Inc. 及其成员和其他所有贡献者都没有，且不得举行，负责以任何方式确认或无法确认任何或所有这样的第三方的知识产权。（译者注：大概是防FIDO联盟成员垄断利益的条款）

**本FIDO联盟规范为“按原样”提供，无任何，包括任何担保，包括但不限于任何明示或针对特定用途的非侵权，适销性或暗示的保证。**

# 目录
1. [注释](#1)  
1.1 [关键字](#1.1)  
2. [概述](#2)  
2.1 [内容范围](#2.1)  
2.2 [结构](#2.2)  
2.3 [协议会话](#2.3)  
2.3.1 [Registration 注册](#2.3.1)  
2.3.2 [Authentication 认证](#2.3.2)  
2.3.3 [Transcation Confirmation 交易确认](#2.3.3)  
2.3.4 [Deregistration 注销](#2.3.4)  
3. [协议细节](#3)  
3.1 [通用结构和类型](#3.1)  
3.1.1 [Version 接口](#3.1.1)  
3.1.1.1 [参数](#3.1.1.1)  
3.1.2 [Operation 枚举](#3.1.2)  
3.1.3 [OperationHeader 字典](#3.1.3)  
3.1.3.1 [字典 OperationHeader 成员](#3.1.3.1)  
3.1.4 [Authenticator Attestation ID (AAID) typedef](#3.1.4)  
3.1.5 [KeyID typedef](#3.1.5)  
3.1.6 [ServerChallenge 类型声明](#3.1.6)  
3.1.7 [FinalChallengeParams 字典](#3.1.7)  
3.1.7.1 [字典 FinalChallengeParams 成员](#3.1.7.1)  
3.1.8 [TLS ChannelBinding 字典](#3.1.8)  
3.1.8.1 [字典 ChannelBinding 成员](#3.1.8.1)  
3.1.9 [JwkKey 字典](#3.1.9)  
3.1.9.1 [字典 JwkKey 成员](#3.1.9.1)  
3.1.10 [Extension 字典](#3.1.10)  
3.1.10.1 [字典 Extension 成员](#3.1.10.1)  
3.1.11 [MatchCriteria 字典](#3.1.11)  
3.1.11.1 [字典 MatchCriteria 成员](#3.1.11.1)  
3.1.12 [Policy 字典](#3.1.12)  
3.1.12.1 [字典 Policy 成员](#3.1.12.1)  
3.2 [Server Policy的处理规则](#3.2)  
3.2.1 [Examples](#3.2.1)  
3.3 [版本协商](#3.3)  
3.4 [Registration 操作](#3.4)  
3.4.1 [Registration Request 消息](#3.4.1)  
3.4.2 [RegistrationRequest 字典](#3.4.2)  
3.4.2.1 [字典 RegistrationRequest 成员](#3.4.2.1)  
3.4.3 [AuthenticatorRegistrationAssertion 字典](#3.4.3)  
3.4.3.1 [字典 AuthenticatorRegistrationAssertion 成员](#3.4.3.1)  
3.4.4 [Registration Response 消息](#3.4.4)  
3.4.5 [RegistrationResponse 字典](#3.4.5)  
3.4.5.1 [字典 RegistrationResponse 成员](#3.4.5.1)  
3.4.6 [Registration 处理规则](#3.4.6)  
3.4.6.1 [FIDO Server生成Registration Request的规则](#3.4.6.1)  
3.4.6.2 [FIDO UAF Clients处理Registration Request的规则](#3.4.6.2)  
3.4.6.3 [FIDO Authenticator处理Registration Request的规则](#3.4.6.3)  
3.4.6.4 [FIDO UAF Client生成Registration Response的规则](#3.4.6.4)  
3.4.6.5 [FIDO Server处理Registration Response的规则](#3.4.6.5)  
3.5 [Authentication 操作](#3.5)  
3.5.1 [Transaction 字典](#3.5.1)  
3.5.1.1 [字典 Transaction 成员](#3.5.1.1)  
3.5.2 [Authentication Request 消息](#3.5.2)  
3.5.3 [AuthenticationRequest 字典](#3.5.3)  
3.5.3.1 [字典 AuthenticationRequest 成员](#3.5.3.1)  
3.5.4 [AuthenticatorSignAssertion 字典](#3.5.4)  
3.5.4.1 [字典 AuthenticatorSignAssertion 成员](#3.5.4.1)  
3.5.5 [AuthenticationResponse 字典](#3.5.5)  
3.5.5.1 [字典 AuthenticationResponse Members](#3.5.5.1)  
3.5.6 [Authentication Response 消息](#3.5.6)  
3.5.7 [Authentication 处理规则](#3.5.7)  
3.5.7.1 [FIDO Server生成Authentication Request的规则](#3.5.7.1)  
3.5.7.2 [FIDO UAF Client处理Authentication Request的规则](#3.5.7.2)  
3.5.7.3 [FIDO Authenticator处理Authentication Request的规则](#3.5.7.3)  
4.1.1 [Authenticator Metadata](#4.1.1)  
4.1.2 [Authenticator Attestation](#4.1.2)  
4.1.2.1 [Basic Attestation](#4.1.2.1)  
4.1.2.1.1 [Full Basic Attestation](#4.1.2.1.1)  
4.1.2.1.2 [Surrogate Basic Attestation](#4.1.2.1.2)  
4.1.3 [错误处理](#4.1.3)  
4.1.4 [Assertion Schemes](#4.1.4)  
4.1.5 [Authenticator的Username](#4.1.5)  
4.1.6 [Silent Authenticators](#4.1.6)  
4.1.7 [TLS通讯保护](#4.1.7)  
4.2 [实现注意事项](#4.2)  
4.2.1 [Server Challenge挑战码和随机数](#4.2.1)  
4.3 [安全性注意事项](#4.3)  
4.3.1 [FIDO Authenticator的安全性](#4.3.1)  
4.3.2 [加密算法](#4.3.2)  
4.3.3 [FIDO Client信任模块](#4.3.3)  
4.3.3.1 [隔离使用KHAccessToken](#4.3.3.1)  
4.3.4 [TLS绑定](#4.3.4)  
4.3.5 [Session管理](#4.3.5)  
4.3.6 [用户的多个身份](#4.3.6)  
4.3.7 [ServerData and KeyHandle](#4.3.7)  
4.3.8 [通过 UAF Application API 恢复 Authenticator Information vs. Metadata](#4.3.8)  
4.3.9 [Policy 验证](#4.3.9)  
4.3.10 [重放攻击保护](#4.3.10)  
4.3.11 [反克隆Authenticators保护](#4.3.11)  
4.3.12 [反欺骗信号](#4.3.12)  
4.4 [交互注意事项](#4.4)  
5. [UAF 支持的 Assertion Schemes](#5.)  
5.1 [Assertion Scheme "UAFV1TLV"](#5.1)  
5.1.1 [KeyRegistrationData](#5.1.1)  
5.1.2 [SignedData](#5.1.2)  
6. [定义](#6)  
7. [图表](#7)  
A. [参考文献](#A)  
A.1 [标准参考文献](#A.1)  
A.2 [参考性文献](#A.2)  

# 1. 符号解释
类型名，属性名和成员名将写成`code`。

字符串会被“”包起来，例如“UAF-TLV”。

在公式中使用“|”表示将两个byte数组串联（译者注：注意不是或）

符号base64url表示“Base64编码的URL和文件名安全的字符集”[RFC4648](#bib-RFC4648)且没有补位。（*译者注：即base64末尾的“=”号*）

遵循[WebIDL-ED](#bib-WebIDL-ED)，除非显式标注为`required`，则字典成员是可选的。

WebIDL字典成员**不可以**值为null （*译者注：1.2添加说明如下：“—— 也就是说，本规范中，所有的字典成员值都不可以为null。”*）（*译者注：json中，null和空{}是不同的概念*）

如果字典成员是DOMString，除非特别说明，否则不能为空（empty）

如果字典成员是List，除非特别说明，否则不能为空（empty）list

本文档中用到的UAF特定的术语在[FIDOGlossary](#bib-FIDOGlossary)中可以找到相关定义。

本规范中所有的图表，例子，notes都不属于规范。

> NOTE
>
> Note：某些字典成员必须出现，·以符合FIDO的要求。在本文档中这些成员在WebIDL定义中被标记为`required`。关键字`required`在[WebIDL-ED](#bib-WebIDL-ED)（还是半成品）中有相关介绍。如果你使用了实现了[WebIDL](#bib-WebIDL)的WebIDL解析器，则可能需要从你的WebIDL定义中去掉`required`关键字，并使用其他方式来保证这些域必须存在。

# 1.1 关键字
本文中出现的关键字**必须**，**不可以**，**需要**，**不应**，**应该（SHOULD）**，**建议**，**应该（MAY）**，**可选**，遵循[RFC2119](#bib-RFC2119)中的解释。（*译者注：原文为“must”, “must not”, “required”, “shall”, “shall not”, “should”, “should not”, “recommended”, “may”, and “optional”*）

# 2 概述
*本章节不是规范。*

通用认证框架（Universal Authentication Framework，UAF）的目标是提供一个统一的可扩展的认证机制，以代替密码，同时避免当前选择的身份验证方法的缺点。

设计的协议的目标是，允许依赖方（Relying Parties）通过一个统一的协议，来利用终端用户上多种多样的安全功能。

该方法被设计为允许依赖方为特定的终端用户或交互选择最适合的认证机制，同时考虑到在未来利用新兴设备安全功能的可能性，而不用再做额外的集成工作。

## 2.1 内容范围
本文档介绍了FIDO架构的细节，并定义了UAF协议作为网络协议。其定义了所有UAF messages的处理流程和内容，并阐述了其设计背后的基本原理。

特定平台的应用层的绑定方案不在此文档的范围之内。该文档不会解答比如这样的问题： 

* UAF的HTTP binding是什么样子的？
* web application如何与FIDO UAF Client进行通信？
* FIDO UAF Client如何与具有FIDO功能的Authenticators通信？

这些问题的答案可以在其他UAF规范中找到，如[[UAFAppAPIAndTransport](#bib-UAFAppAPIAndTransport)] [[UAFASM](#bib-UAFASM)] [[UAFAuthnrCommands](#bib-UAFAuthnrCommands)]。

## 2.2 FIDO结构
下面的图表描述了UAF协议的涉及的接口：

![](https://github.com/SickWorm/FIDO-UAF-Chinese-Document/raw/master/UAF1.0翻译/img/fido-uaf-architecture.png)
Fig. 1 UAF架构

这些接口里面，只有这3个会直接生成或处理UAF协议message：

* FIDO Server，在依赖方（服务提供商，如腾讯）的服务器上运行
* FIDO UAF Client（Web，App），在FIDO使用者的设备上运行，属于user agent的一部分
* FIDO 认证设备（Authenticator），和FIDO使用者的设备配合使用

本文档假设FIDO Server已经向UAF Authenticator Metadata [[UAFAuthnrMetadata](#bib-UAFAuthnrMetadata)]提交了所有可能会交互的Authenticator的信息。

### 2.3 协议会话：
FIDO UAF Client和FIDO Server之间的UAF协议核心共有4种会话：

* 注册（Registration）
UAF可以让依赖方使用用户账户注册FIDO Authenticator，依赖方可以制定具体策略支持各种不同的FIDO Authenticator类型。FIDO UAF Client只能根据policy注册依赖方提供的类型。

* 认证（Authentication）
UAF可以让依赖方的终端用户使用已经注册过的FIDO Authenticator进行认证。认证可以在任意时候执行，由依赖方自行判断。

* 交易确认（Transacation Confirmation）
通了提供一个通用的认证请求之外，UAF也可以让用户认证一次特定的交易。
该次认证会有额外的信息给client（*译者注：泛指终端，不是FIDO UAF Client*)，通过client交易确认显示模块显示给终端用户。这个额外的认证操作用于依赖方确保用户了解这个交易的信息细节（而不是只是简单的给user agent做一次认证session）（*译者注：比如说登录只需要Authentication，但做银行转账就需要Transcation Confirmation*）)

* 注销（Deregistration）
依赖方可以用于删除与帐户相关的认证key的数据

虽然该文档定义了FIDO Server是请求的发起者，在实际情况中第一个UAF操作请求总是user agent's（如HTTP）向依赖方服务器发起的（*译者注：client发起，然后server创建UAF message返回给client。[[UAFAppAPIAndTransport](#bib-UAFAppAPIAndTransport)]中有介绍。*）。

<<<<<<< HEAD
以下的章节提供了每一个UAF操作的简要的协议会话介绍 。更详细的描述在3.4 Registration Operation，3.5 Authentication Operation，3.6 Deregistration Operation中。

### 2.3.1 Registration
下图表为Registration的信息流：
![](https://github.com/SickWorm/FIDO-UAF-Chinese-Document/raw/master/UAF1.0翻译/img/fido-uaf-protocol-registration.png)
Fig. 2 Registration信息流

> NOTE
> 
> client application 应该使用适合的API去通知FIDO UAF Client操作的结果（详细查看2.3.1[[UAFAppAPIAndTransport](#bib-UAFAppAPIAndTransport)]，FIDO UAF Client可能会根据结果执行一些内部操作。

### 2.3.2 Authentication
下图表为Authentication的信息流：
![](https://github.com/SickWorm/FIDO-UAF-Chinese-Document/raw/master/UAF1.0翻译/img/fido-uaf-protocol-authentication.png)
Fig. 3 Authentication信息流

> NOTE
> 
> client application 应该使用适合的API去通知FIDO UAF Client操作的结果（详细查看2.3.1[[UAFAppAPIAndTransport](#bib-UAFAppAPIAndTransport)]，FIDO UAF Client可能会根据结果执行一些内部操作。

### 2.3.3 Transcation Confirmation
下图表为Transcation Confirmation的信息流：
![](https://github.com/SickWorm/FIDO-UAF-Chinese-Document/raw/master/UAF1.0翻译/img/fido-uaf-protocol-transaction-confirm.png)
Fig. 3 Transcation Confirmation信息流

> NOTE
> 
> client application 应该使用适合的API去通知FIDO UAF Client操作的结果（详细查看2.3.1[[UAFAppAPIAndTransport](#bib-UAFAppAPIAndTransport)]，FIDO UAF Client可能会根据结果执行一些内部操作。

### 2.3.4 Deregistration
下图表为Deregistration的信息流：
![](https://github.com/SickWorm/FIDO-UAF-Chinese-Document/raw/master/UAF1.0翻译/img/fido-uaf-protocol-deregistration.png)
Fig. 3 Deregistration信息流

> NOTE
> 
> client application 应该使用适合的API去通知FIDO UAF Client操作的结果（详细查看2.3.1[[UAFAppAPIAndTransport](#bib-UAFAppAPIAndTransport)]，FIDO UAF Client可能会根据结果执行一些内部操作。

# 3. 协议细节
*本章节属于规范。*

本章节包括了UAF Protocol支持的各个操作的细节描述。 

除非另外说明，规范强制性要求软件需要支持所有的协议元素。

本规范中所有字符值的范围是Unicode的`U+0000`..`U+007F`。

除非另外说明，协议message都用UTF-8编码。

> NOTE
> 
> 协议中FIDO UAF Client和依赖方之间用到的所有数据都必须使用安全传输协议进行传输（如TLS/HTTPS），以遵循[[FIDOSecRef](#bib-FIDOSecRef)]中的假设；详细内容在[4.1.7 TLS Protected Communication](#4.1.7)。

`base64url(byte[8..64])`表示8到64字节的base64url编码的数据，base64url表示“URL和文件名安全的Base64编码字符集”且没有padding[[RFC4648](#bib-RFC4648)]。（*译者注：即Base64.NO_WRAP | Base64.NO_PADDING | Base64.URL_SAFE*）

`string[5]`表示5个unicode字符，即UTF-8 [[RFC3629](#bib-RFC3629)]编码的声明中指定的类型的string，如WebIDL [[WebIDL-ED](#bib-WebIDL-ED)]的DOMString。

因为UTF-8是变长编码，所以`string[5]`的*最大*长度为`string[4*5]`。

除非另外说明，所有的string都是大小写敏感的。

本文档使用WebIDL [[WebIDL-ED](#bib-WebIDL-ED)]来定义UAF protocol messages。

在协议实现中，传输过程中**必须**使用UTF-8编码的JSON [[RFC4627](#bib-RFC4627)]来串行化UAF protocol messages。
=======
以下的章节提供了每一个UAF操作的简要的协议会话介绍 。更详细的描述在3.4 Registration Operation，3.5 Authentication Operation，3.6 Deregistration Operation中。
>>>>>>> parent of 241557f... 增加协议2.3.1 - 3章节
