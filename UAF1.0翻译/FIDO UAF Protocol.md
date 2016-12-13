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
通用认证框架（Universal Authentication Framework，UAF）提供了一个统一的可扩展的认证机制，以代替密码，同时避免当前选择的身份验证方法的缺点。

该方法被设计为允许依赖方为特定的终端用户或交互选择最适合的认证机制，同时保留了在未来利用新兴设备安全功能的可能性，而不用做额外的集成工作。

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
3.5.7.3 [FIDO Authenticator处理Authe 
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