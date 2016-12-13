
# FIDO UAF  Authenticator Commands v1.0
FIDO联盟实施草案 2014年12月08日

**本版本**
[https://fidoalliance.org/specs/fido-uaf-v1.0-ps-20141208/fido-uaf-authnr-cmds-v1.0-ps-20141208.html](https://fidoalliance.org/specs/fido-uaf-v1.0-ps-20141208/fido-uaf-authnr-cmds-v1.0-ps-20141208.html)

**上一版本**
[https://fidoalliance.org/specs/fido-uaf-authnr-cmds-v1.0-rd-20141008.pdf](https://fidoalliance.org/specs/fido-uaf-authnr-cmds-v1.0-rd-20141008.pdf)

**编辑：**
Davit Baghdasaryan, Nok Nok Labs, Inc.  
John Kemp, FIDO Alliance

**贡献者:**
Dr. Rolf Lindemann, Nok Nok Labs, Inc.  
Roni Sasson, Discretix  
Brad Hill, PayPal, Inc.  

本规范的英文版本是唯一的规范版本。也可能有非规范的[翻译](https://www.fidoalliance.org/specifications/translation/)版本。

Copyright © 2013-2014 [FIDO Alliance](https://www.fidoalliance.org/) All Rights Reserved.

# 摘要
UAF Authenticators可能会采取不同的形式实现。实现方式从在防篡改硬件中运行的安全应用，到在消费者设备上纯软件的解决方案都是可能的。

本文档定义了UAF Authenticators规范性方面，为authenticator实现人员提供安全性和实现指导。

# 本文档状态
*本章节描述了本文档在发布时的状态。可能会有其他文档代替本文档。FIDO联盟发布过的以及最新的技术报告在[FIDO Alliance specifications index](https://fidoalliance.org/specifications/)的*https://www.fidoalliance.org/specifications/*都可以找到。

本文档作为建议标准由 [FIDO Alliance](https://www.fidoalliance.org/)发布。本文档旨在成为FIDO联盟的建议标准。如果你对本文档有任何意见或建议，请[联系我们](https://fidoalliance.org/contact/)。我们欢迎任何意见和建议。

本规范某些要素的实现可能需要第三方知识产权的许可，包括但不限于专利权。FIDO Alliance, Inc. 及其成员和其他所有贡献者都没有，且不得举行，负责以任何方式确认或无法确认任何或所有这样的第三方的知识产权。（译者注：大概是防FIDO联盟成员垄断利益的条款）

**本FIDO联盟规范为“按原样”提供，无任何，包括任何担保，包括但不限于任何明示或针对特定用途的非侵权，适销性或暗示的保证。**

# 目录
1. [注释](#1)
1.1 [关键字](#1.1)
2. [概述](#2)
3. [额外说明](#3)
4. [UAF Authenticator](#4)
4.1 [Authenticators类型](#4.1)
5. [Tags](#5)
5.1 [Command Tags](#5.1)
5.2 [只用于Authenticator的Commands](#5.2)
5.3 [用于UAF Protocol的Tags](#5.3)
5.4 [Status Codes](#5.4)
6. [数据结构](#5)
6.1 [RawKeyHandle](#6.1)
6.1.1 [FIDO Server解析的结构](#6.2)
6.1.1.1 [TAG_UAFV1_REG_ASSERTION](#6.2.1)
6.1.1.2 [TAG_UAFV1_AUTH_ASSERTION](#6.2.2)
6.1.2 [UserVerificationToken](#6.3)
6.2 [Commands](#6.2)
6.2.1 [GetInfo Command](#6.1)
6.2.1.1 [Command描述](#6.1.1)
6.2.1.2 [Command结构](#6.1.2)
6.2.1.3 [Command Response](#6.1.3)
6.2.1.4 [Status Codes](#6.1.4)
6.2.2 [Register Command](#6.2)
6.2.2.1 [Command结构](#6.2.1)
6.2.2.2 [Command Response](#6.2.2)
6.2.2.3 [Status Codes](#6.2.3)
6.2.2.4 [Command描述](#6.2.4)
6.2.3 [Sign Command](#6.3)
6.2.3.1 [Command结构](#6.3.1)
6.2.3.2 [Command Response](#6.3.2)
6.2.3.3 [Status Codes](#6.3.3)
6.2.3.4 [Command描述](#6.3.4)
6.2.4 [Deregister Command](#6.4)
6.2.4.1 [Command结构](#6.4.1)
6.2.4.2 [Command Response](#6.4.2)
6.2.4.3 [Status Codes](#6.4.3)
6.2.4.4 [Command描述](#6.4.4)
6.2.5 [OpenSettings Command](#6.5)
6.2.5.1 [Command结构](#6.5.1)
6.2.5.2 [Command Response](#6.5.2)
6.2.5.3 [Status Codes](#6.5.3)
7. [KeyIDs and key handles](#7)
7.1 [first-factor Bound Authenticator](#7.1)
7.2 [2ndF Bound Authenticator](#7.2)
7.3 [first-factor Roaming Authenticator](#7.3)
7.4 [2ndF Roaming Authenticator](#7.4)
8. [Commands访问控制](#8.)
9. [与其他规范的关系](#9.)
9.1 [TEE](#9.1)
9.2 [Secure Elements](#9.2)
9.3 [TPM](#9.3)
9.4 [不可靠传输](#9.4)
A. [安全性指引](#A)
B. [图表](#B)
C. [参考文献](#C)
C.1 [标准参考文献](#C.1)
C.2 [参考性文献](#C.2)

# 1. 符号解释
类型名，属性名和成员名将写成`code`。

字符串会被“”包起来，例如“UAF-TLV”。

在公式中使用“|”表示将两个byte数组串联（译者注：注意不是或）

本文档用到的UAF专业术语在 [[FIDOGlossary](#bib-FIDOGlossary)]中定义。

本规范中所有图片，实例，小记（notes）不属于规范。

## 1.1 关键字

本文中出现的关键字**必须**，**不可以**，**需要**，**不应**，**应该（SHOULD）**，**建议**，**应该（MAY）**，**可选**，遵循[RFC2119](#bib-RFC2119)中的解释。

# 2 概述
*本章节不是规范。*

本文档制定了UAF Authenticators应该实现的用于支持UAF协议的底层的功能。文档有如下几个目标：
    * 定义UAF Authenticator的实现规范
    * 定义UAF功能的一系列指令实现，这些指令可以被不同类型的authenticator实现。
    * 定义`UAFV1TLV`断言方案结构，该断言可被FIDO Server解析。

>NOTE
>UAF Protocol支持不同的断言方案。本文档定义的指令和结构都假设authenticator支持`UAFV1TLV`断言方案。实现了不同断言方案的Authenticator不需要遵循本文档规定的要求。（译者注：目前UAF协议1.2版本只支持这一个断言方案）

整个UAF Protocol的架构和其操作定义在[[UAFProtocol](#bib-UAFProtocol)]中。下面这个简化的架构图说明了本文档涉及的交互和行为：
[img]
Fig. 1 UAF Authenticator Commands

# 3 额外说明

除非另外规定，本文档所有的数据**必须**以小端格式编码。

所有的TLV结构可以使用“递归-下降”解析，且tag的顺序不是一定的。在某些情况下一个tag**可能**在同一个结构中出现很多次，此时所有的value都必须保存下来。

除非另外明确提到，否则TLV结构所有的域都是*强制要求*的。

# 4 UAF Authenticator
*本章节不是规范。*

UAF Authenticator是一个满足了[[UAFProtocol](#bib-UAFProtocol)]中描述的协议要求的认证组件。UAF Authenticators具备的主要功能如下：

1. 【强制】使用authenticator内置的认证装置认证用户。认证技术可以是多样的，从生物识别到简单的在场证明，甚至无用户验证（see Slient Authenticator）。
1. 【强制】执行[[UAFProtocol](#bib-UAFProtocol)]中定义的加密操作。
1. 【强制】创建可被FIDO Server解析的数据结构。
1. 【强制】向FIDO Server证明自己拥有认证能力。
1. 【可选】向用户显示交易内容。

[img]
Fig. 2 FIDO Authenticator逻辑上的子组件

一些UAF Authenticators的代表：
* 移动设备内置的指纹传感器
* *secure element*内置的PIN码验证（译者注：*secure element*是一个标准硬件平台）
* 内置用户在场验证的USB令牌
* 设备内置的声音或脸部识别技术

## 4.1 Authenticators的种类
本文档一共定义了4中authenticator。这些定义不是规范的（除非另有说明），仅仅为了下文的简化描述。
>NOTE
>以下是仅考虑这4种authenticator的原理：
>* 绑定authenticator是典型的嵌入在用户计算设备中并使用宿主存储的设备。从经济角度上使用宿主的存储比使用（自己的）嵌入式存储更有意义。Trusted Execution Environments (TEE), Secure Elements and Trusted Platform Modules (TPM)就是典型的以此理念设计的设备。
>* 第一因素漫游设备（First-factor roaming authenticators）必须拥有内部存储来存储key handles。
>* 第二因素漫游设备（Second-factor roaming authenticators）可以把key handles存储在相关服务器上，而不用提供内部存储。
>* 定义这些约束使得在主要的用例中，规范描述将会更加简洁清晰
> 尽管如此，厂商（设计authenticator）并不受这些限制影响。例如一个用内部存储来存储key handles的绑定authenticator也是可以的。厂商可以自由的设计和实现这些authenticator，只要设计符合本文档描述的规范要求。

* First-factor Bound Authenticator
    * 这类authenticator拥有内部匹配器。匹配器可以认证一个已经登记了的用户。如果有多个用户登记——匹配器依然可以认证该用户。
    * authenticator和使用这个authenticator设备会有一个逻辑上的绑定（绑定通过KeyHandleAsscessToken来描述）。这个authenticator无法被多个设备绑定，只能绑定一个。
    * 这类authenticator不会存储key handles到自己的内部存储。它们会把key handle返回给ASM，然后存到ASM本地数据库。
    * 这类authenticator也可以工作在send factor。
    * 例子：
        * 笔记本，手机或平板的指纹识别传感器
        * 内嵌在移动设备的secure element
        * 设备内置的声音识别认证

* Second-factor (2ndF) Bound Authenticator
    * 这类authenticator和first-factor bound authenticator类似，除了它可以以second-factor的身份进行multi-factor认证。
    * 例子：
        * 使用内置电容触摸来认证用户在场的USB令牌
        * 运行在手机上的信任的执行环境中的“Trustlet”应用，利用安全键盘来验证用户在场。（译者注：Trustlet后文有提到）

* First-factor (1stF) Roaming Authenticator
    * 这类authenticator不会与其他设备绑定。用户可以在任意数量的设备上使用。
    * 这类authenticator被假定为拥有内部匹配器。匹配器可以认证一个已经登记了的用户。如果有多个用户登记——匹配器依然可以认证该用户。
    * 这类authenticato被假定为在自己的内部安全存储中存储key handles，不会暴露到外面。
    * 这些authenticator也可以作为second factor工作。
    * 例子：
        * 内置指纹识别传感器的基于BLE的硬件
        * PIN码保护的USB令牌
        * 一个以roaming authenticator身份工作的first-facor bound authenticator，其为另一台设备做认证。

* Second-factor Roaming Authenticator
    * 这类authenticator不会与其他设备绑定。用户可以在任意数量的设备上使用。、
    * 这类authenticator**可能**拥有内部匹配器。匹配器可以认证一个已经登记了的用户。如果有多个用户登记——匹配器依然可以认证该用户。
    * 这类authenticator被假定为不会在自己的内部存储中存储key handles。它们会把key handles上传到FIDO Server，等到认证操作的时候再由FIDO Server传回来。
    * 这类authenticator只能作为second factor。
    * 例子：
        * 内置电容触摸用于验证用户在场的USB加密器
        * 一个运行在手机上的信任的执行环境中的“Trustlet”应用，利用安全键盘来验证用户在场。

>译者注：
>下文有提到：支持用户名是first-和second- factor authenticator之间的关键区别。first-可以区分不同key handles分别属于哪个用户（key handle有对应的username），second-没有username这个概念，所有key handle对它来说都是一样的。
>我的理解：
>First-factor可以理解为只有本人才知道的信息。比如PIN，指纹等。该设备可以直接登录。
>
>Second-factor可以理解为本人持有的东西，比如预写好密钥或证书的便携式设备，你对这个设备的持有被看做是本人。如内置证书的USB令牌。该设备不可以直接登录，需要先通过其他途径进行登录，再使用该设备进行认证，签名。第二因素设备更像u2f设备。
>
>仅供参考。如有误烦请提出。

整篇文章中会有针对这几类型的authenticator指定的特殊条件。

>规范
>在某些部署中，ASM和bound authenticator的组合可以作为一个roaming authenticator使用（例如当一个移动设备上的ASM和一个内嵌的authenticator的组合作为另一台设备的roaming authenticator）。当这种情况发生时，这个authenticator**必须**遵循这个bound authenticator所绑定的系统的要求，并且遵循其连接的在另一系统中，作为roaming authenticator的要求。
>
> （译者注：1.2版本在此段新增了：为了符合authenticator的要求，authenticator**必须**至少实现一个[ [UAFRegistry](#bib-UAFRegistry)]中的认证类型，和[[UAFRegistry](#bib-UAFRegistry)]中的一个认证算法和一个密钥对格式。）

>NOTE
>书上所述，bound authenticator不存储key handles，而roaming authenticator存储key handles（译者注：这里在1.2版本中，roaming authenticators to store them被改为roaming authenticators do store them，猜测是笔误。这里按照1.2的版本翻译）。上面的例子中ASM会存储bound authenticator的key handles，以满足上面的假设。

# 5. Tags
*该章节是规范的。*

本文档中UAF Authenticators使用“Tag-Length-Value"（TLV）格式来与外界通讯。所有的请求和回复都必须是TLV格式的。

command和已存在的预定义TLV tags可以通过附加其他（自定义或预定义的）TLV tags来进行扩展。

参考[[UAFRegistry](#bib-UAFRegistry)]来了解更多有关TAG tags的信息。

TLV格式数据为下面这样的简单结构：

2 bytes|2 bytes|Length bytes
-----|-----|-----
Tag|Length in bytes|Data

>译者注：Length in bytes应该是Length in Data，即第三个成员Data的长度

所有长度的单位是bytes。如UINT32[4]为16个长度。

虽然tag被分配了2个byte，但为了适配某些硬件平台的限制，只可以使用前14个bits（到0x3FFF）。

数组在TLV中是隐藏的。一些数据结构的描述表明该结构允许多个值，在这种情况中，如果tag出现多于1次，则所有的值都是有意义的，它们应该被当做是一个数组。


为了方便解码TLV结构的信息，所有的复合tag——即必须递归下降解析的值（译者注，value是也是一个TLV结构数据）——其第13 bit（0x1000）是1。

第14 bit（0x2000）为1的tag表示这是致命的，接受者**必须**终端处理整个信息，如果无法处理这个tag。（译者注：目前全部TAG第14 bit都为1）

由于UAF Authenticator可能处在极为有限的处理环境，发送command时ASM**必须**遵循规范中定义的结构顺序。（译者注：写死了顺序，可以让解析程序更加简单，authenticator性能更容易满足解析需要）

ASM和Server被假定有充足的资源以任何的顺序解析tags，所以authenticator发送过来的数据结构**可能**是任何顺序的。（译者注：也是为了方便authenticator）

## 5.1 Command Tags
名称|值|描述
----|----|----|
TAG_UAFV1_GETINFO_CMD|0x3401|Tag for GetInfo command.
TAG_UAFV1_GETINFO_CMD_RESPONSE|0x3601|Tag for GetInfo command response.
TAG_UAFV1_REGISTER_CMD|0x3402|Tag for Register command.
TAG_UAFV1_REGISTER_CMD_RESPONSE|0x3602|Tag for Register command response.
TAG_UAFV1_SIGN_CMD|0x3403|Tag for Sign command.
TAG_UAFV1_SIGN_CMD_RESPONSE|0x3603|Tag for Sign command response.
TAG_UAFV1_DEREGISTER_CMD|0x3404|Tag for Deregister command.
TAG_UAFV1_DEREGISTER_CMD_RESPONSE|0x3604|Tag for Deregister command response.
TAG_UAFV1_OPEN_SETTINGS_CMD|0x3406|Tag for OpenSettings command.
TAG_UAFV1_OPEN_SETTINGS_CMD_RESPONSE|0x3606|Tag for OpenSettings command response.

*Table 4.1.1：UAF Authenticator Command TLV tags (0x3400 - 0x34FF, 0x3600-0x36FF)*

## 5.2 只在Authenticator Command中使用的Tag
名称|值|描述
----|----|----|
TAG_KEYHANDLE|0x2801|表示key handle。 参考[[FIDOGlossary](#bib-FIDOGlossary)]以获得有关key handle更详细的信息。
TAG_USERNAME_AND_KEYHANDLE|0x3802|表示 Username和相关的key handle。 这是一个包含TAG_USERNAME和TAG_KEYHANDLE的复合tag，以用于在authenticator上识别注册信息。参考[[FIDOGlossary](#bib-FIDOGlossary)]以获得有关username更详细的信息。
TAG_USERVERIFY_TOKEN|0x2803|表示用户认证token。参考[[FIDOGlossary](#bib-FIDOGlossary)]以获得有关user verification tokens更详细的信息。
TAG_APPID|0x2804|一个UTF-8编码的UINT8[] AppID string。参考[[FIDOGlossary](#bib-FIDOGlossary)]以获得有关AppID更详细的信息。
TAG_KEYHANDLE_ACCESS_TOKEN|0x2805|表示key handle访问token。
TAG_USERNAME|0x2806|一个UTF-8编码的UINT8[]用户名string。
TAG_ATTESTATION_TYPE|0x2807|表示认证类型。
TAG_STATUS_CODE|0x2808|表示Status Code。
TAG_AUTHENTICATOR_METADATA|0x2809|表示更详细的authenticator信息集合。
TAG_ASSERTION_SCHEME|0x280A|一个UTF-8编码的UINT8[]断言方案，可选方案定义在[[UAFRegistry](#bib-UAFRegistry)]。 (例如"UAFV1TLV")
TAG_TC_DISPLAY_PNG_CHARACTERISTICS|0x280B|如果authenticator包含PNG交易展示功能而且不是由上层实现的，该tag则描述这个展示的信息。参考[[UAFAuthnrMetadata](#bib-UAFAuthnrMetadata)] 以获得更多有关该tag的信息。
TAG_TC_DISPLAY_CONTENT_TYPE|0x280C|一个UTF-8编码的UINT8[]交易展示的内容类型，可选类型定义在[[UAFAuthnrMetadata](#bib-UAFAuthnrMetadata)]。 (例如"image/png")
TAG_AUTHENTICATOR_INDEX|0x280D|Authenticator Index
TAG_API_VERSION|0x280E|API Version（译者注：该版本即1.2）
TAG_AUTHENTICATOR_ASSERTION|0x280F|该tag的值是由authenticator生成的断言。因为authenticator生成的断言可能有好几种格式，不同authenticator之间该值的格式可能也不同。
TAG_TRANSACTION_CONTENT|0x2810|表示发给authenticator的交易内容。
TAG_AUTHENTICATOR_INFO|0x3811|内含authenticator的功能细节信息。
TAG_SUPPORTED_EXTENSION_ID|0x2812|表示authenticator支持的扩展ID。

*Table 4.2.1: Non-Command Tags (0x2800 - 0x28FF, 0x3800 - 0x38FF)*

## 5.3 UAF Protocol中用到的Tag
名称|值|描述
----|----|----
TAG_UAFV1_REG_ASSERTION|0x3E01|Authenticator对Register command的回复。
TAG_UAFV1_AUTH_ASSERTION|0x3E02|Authenticator对Sign command的回复。
TAG_UAFV1_KRD|0x3E03|Key Registration Data
TAG_UAFV1_SIGNED_DATA|0x3E04|authenticator使用UAuth。priv私钥签名的数据。
TAG_ATTESTATION_CERT|0x2E05|每个入口包含一个X。509 DER编码[[ITU-X690-2008](#bib-ITU-X690-2008)]证书。可多次出现并组成认证证书链。多次出现时必须按顺序出现。认证证书本身必须在最前面。每个后续出现的证书（如果存在）必须是前一个出现的证书的颁发证书。
TAG_SIGNATURE|0x2E06|加密的签名
TAG_ATTESTATION_BASIC_FULL|0x3E07|Full Basic Attestation，定义在[[UAFProtocol](#bib-UAFProtocol)]。
TAG_ATTESTATION_BASIC_SURROGATE|0x3E08|Surrogate Basic Attestation，定义在[[UAFProtocol](#bib-UAFProtocol)]。
TAG_KEYID|0x2E09|表示KeyID。
TAG_FINAL_CHALLENGE|0x2E0A|表示Final Challenge。参考[[UAFProtocol](#bib-UAFProtocol)]以获得有关the Final Challenge更详细的信息。
TAG_AAID|0x2E0B|表示authenticator的认证ID。参考[[UAFProtocol](#bib-UAFProtocol)]以获得有关the AAID更详细的信息。（译者注：1.2版本中该TAG被修改为TAG_FINAL_CHALLENGE_HASH）
TAG_PUB_KEY|0x2E0C|表示公钥。
TAG_COUNTERS|0x2E0D|表示authenticator的计数（译者注：有注册计数和签名计数）。
TAG_ASSERTION_INFO|0x2E0E|表示message处理时需要的断言信息。
TAG_AUTHENTICATOR_NONCE|0x2E0F|表示authenticator临时生成的值。
TAG_TRANSACTION_CONTENT_HASH|0x2E10|表示交易信息的hash值。
TAG_EXTENSION|0x3E11, 0x3E12|这是一个复合TAG，表示值为扩展内容。如果tag值为0x3E11，表示这是一个非常重要的扩展，如果接收方不能解读这个tag，则停止处理整个message。该tag内含两个tag —— TAG_EXTENSION_ID和TAG_EXTENSION_DATA。 参考[[UAFProtocol](#bib-UAFProtocol)]以获得有关Extension更详细的信息。**NOTE**：该tag可以存在于任何command和response中。使用tag 0x3E11(和tag 0x3E12相反)和[[UAFProtocol](#bib-UAFProtocol)]中的flag fail_if_unknown意思是一样的。
TAG_EXTENSION_ID|0x2E13|表示extension ID。值为UTF-8编码的UINT8[] string。
TAG_EXTENSION_DATA|0x2E14|表示extension数据。值为UINT8[] byte数组。

*Table 4.3.1: Tags used in the UAF Protocol (0x2E00 - 0x2EFF, 0x3E00 - 0x3EFF). Normatively defined in [[UAFRegistry(#bib-UAFRegistry)]]*

## 5.4 状态码（Status Code）
名称|值|描述
----|----|----
UAF_CMD_STATUS_OK|0x00|成功。
UAF_CMD_STATUS_ERR_UNKNOWN|0x01|未知错误。
UAF_CMD_STATUS_ACCESS_DENIED|0x02|该操作被拒绝。
UAF_CMD_STATUS_USER_NOT_ENROLLED|0x03|用户还没有在这个authenticator上登记，且authenticator无法自动触发登记流程。
UAF_CMD_STATUS_CANNOT_RENDER_TRANSACTION_CONTENT|0x04|无法展示交易信息。
UAF_CMD_STATUS_USER_CANCELLED|0x05|用户取消操作。
UAF_CMD_STATUS_CMD_NOT_SUPPORTED|0x06|命令不支持。
UAF_CMD_STATUS_ATTESTATION_NOT_SUPPORTED|0x07|不支持该请求的认证类型。
UAF_CMD_STATUS_PARAMS_INVALID|0x08|authenticator接收到的command的参数无效或格式错误。
UAF_CMD_STATUS_KEY_DISAPPEARED_PERMANENTLY|0x09|与该命令相关的UAuth key从authenticator找不到且无法恢复。对于某些authenticator，当用户认证参考数据被修改时会发生此错误。 (如新添加了的指纹模板)（译者注：意思是超过了指纹存储数量，把旧的冲掉/删掉了）。
UAF_CMD_STATUS_TIMEOUT|0x0a|该操作在authenticator中超时（因为技术问题），并因此中断了操作。
UAF_CMD_STATUS_USER_NOT_RESPONSIVE|0x0e|用户在指示中耗时过长，如没有在限定时间内验证指纹。
UAF_CMD_STATUS_INSUFFICIENT_RESOURCES|0x0f|authenticator资源不足以执行请求的任务。
UAF_CMD_STATUS_USER_LOCKOUT|0x10|操作失败，用户被锁定且authenticator无法自动触发恢复。典型情况用户需要输入一个替代的密码（或更正式的说法：执行一些其他替代的用户认证方法），来恢复主要的用户认证方法。（译者注：如iPhone指纹识别失败太多次，只能用密码来解锁屏幕）**NOTE**：任何用户可以用来恢复主要的用户认证方法都可以被认为是替代的用户认证方法，这些认证方法必须被正确的声明。例如，如果用户可以用替代的密码去恢复指纹或增加新指纹，则显然authenticator支持基于指纹或密码的用户认证。

*Table 4.4.1: UAF Authenticator Status Codes(0x00 - 0xFF)*

# 6. Structures
*本章节是规范的。*

## 6.1 RawKeyHandle
RawKeyHandle是一个由authenticator生成和解析的结构。authenticator**应该**定义不同的RawKeyHandle，RawKeyHandle内部结构应仅与指定的authenticator实现相关。
一个典型的**first-factor bound atuenticator**的RawKeyHandle有如下的结构：

和哈希算法有关（如32bytes）|和密钥类型有关（如32bytes）|用户名长度（1bytes）|最多128bytes
----|----|----|----
KHAccessToken|UAuth.priv|Size|Username

Table 5.1: RawKeyHandle Structure

first factor authenticator**必须**存储Username在RawKeyHandle中（译者注：1.2版本被修改为：first factor authenticator**必须**自己存储用户名且**必须**将key和用户名关联在一起。这**可以**通过存储用户名到RawKeyHandle来实现。），second factor authenticator**不可以**存储用户名。

支持用户名是first-和second- factor authenticator之间的关键区别。

在离开authenticator边界时（译者注：即authenticator要完成自己的操作，把数据返回到上层的时候），RawKeyHandle**必须**加密，因为其存放着敏感信息。如用户的私钥（UAuth.priv）。

## 6.1.1 由FIDO Server解析的结构
本章节定义的结构由UAF Authenticator创建，被FIDO Server解析。

如果authenticator实现了"UAFV1TLV"断言方案，则**必须**生成这些结构。

>NOTE
>"UAFV1TLV"断言方案假设authenticator拥有TAG_UAFV1_KRD 和TAG_UAFV1_SIGNED_DATA内部所有数据的独占控制权。

嵌套关系结构必须保留，但在一个复合tag中的tag的顺序不必遵循规范。FIDO Servers**必须**准备好处理任何顺序的tag。

### 6.1.1.1 TAG_UAFV1_REG_ASSERTION
下面的TLV结构由authenticator在处理Register command的时候生成。然后原封不动地发送到FIDO Server，并被server解析。该结构内嵌了TAG_UAFV1_KRD，该tag含有新生成的公钥和其他数据。

如果authenticator想要在TAG_UAFV1_KRD上附加自定义数据（然后用认证密钥签名）——这些数据必须包含在TAG_EXTENSION_DATA 中，而TAG_EXTENSION_DATA包含在TAG_UAFV1_KRD的TAG_EXTENSION 中。

如果authenticator想要发送额外的数据给FIDO Server而不签名——这些数据**必须**必须包含在TAG_EXTENSION_DATA 中，而TAG_EXTENSION_DATA包含在TAG_UAFV1_REG_ASSERTION 的TAG_EXTENSION 中。

当前本文档只指定了TAG_ATTESTATION_BASIC_FULL，TAG_ATTESTATION_BASIC_SURROGATE和TAG_ATTESTATION_ECDAA这三个认证类型。以防万一authenticator需要对TAG_UAFV1_KRD 执行“其他某种认证“，其**必须**使用TLV tag和内容定义”其他某种认证“（定义在[[UAFRegistry](#bib-UAFRegistry)]）。

序号|TLV结构|描述
----|----|----
1|UINT16 Tag|TAG_UAFV1_REG_ASSERTION
1.1|UINT16 Length|结构长度
1.2|UINT16 Tag|TAG_UAFV1_KRD
1.2.1|UINT16 Length|结构长度
1.2.2|UINT16 Tag|TAG_AAID
1.2.2.1|UINT16 Length|AAID长度
1.2.2.2|UINT8[] AAID|Authenticator认证ID
1.2.3|UINT16 Tag|TAG_ASSERTION_INFO
1.2.3.1|UINT16 Length|断言信息长度
1.2.3.2|UINT16 AuthenticatorVersion|厂商写入的authenticator版本
1.2.3.3|UINT8 AuthenticationMode|注册操作必须为0x01表明用户已经明确地验证了该action。
1.2.3.4|UINT16 SignatureAlgAndEncoding|签名算法和认证签名的编码。参考[[UAFRegistry](#bib-UAFRegistry)]以获得有关支持的算法和其值。
1.2.3.5|UINT16 PublicKeyAlgAndEncoding|公钥算法和新生成的UAuth.pub密钥的编码。参考[[UAFRegistry](#bib-UAFRegistry)]以获得有关支持的算法和其值。
1.2.4|UINT16 Tag|TAG_FINAL_CHALLENGE
1.2.4.1|UINT16 Length|最终挑战码长度
1.2.4.2|UINT8[] FinalChallenge|Command中提供的最终挑战码（字节码）。
1.2.5|UINT16 Tag|TAG_KEYID
1.2.5.1|UINT16 Length|KeyID长度
1.2.5.2|UINT8[] KeyID|Authenticator生成的KeyID字节码。
1.2.6|UINT16 Tag|TAG_COUNTERS
1.2.6.1|UINT16 Length|计数器长度
1.2.6.2|UINT32 SignCounter|签名计数。表示这个authenticator一共执行了的签名的次数。
1.2.6.3|UINT32 RegCounter|注册计数。表示这个authenticator一共执行了的注册的次数。
1.2.7|UINT16 Tag|TAG_PUB_KEY
1.2.7.1|UINT16 Length|UAuth.pub长度
1.2.7.2|UINT8[] PublicKey|authenticator刚生成的用户认证公钥（UAuth.pub）。
1.3 **(选择1)**|UINT16 Tag|TAG_ATTESTATION_BASIC_FULL
1.3.1|UINT16 Length|结构长度
1.3.2|UINT16 Tag|TAG_SIGNATURE
1.3.2.1|UINT16 Length|签名长度
1.3.2.2|UINT8[] Signature|Basic Attestation私钥对TAG_UAFV1_KRD进行签名的结果。整个TAG_UAFV1_KRD，包括tag和length都**必须**加入到签名数据。
1.3.3|UINT16 Tag|TAG_ATTESTATION_CERT (可能出现多次)。多次出现时必须按照顺序出现。认证证书必须出现在首位。每个子序列（如果存在）必须是前一个出现的证书颁发证书。最后出现的必须链接到的包含在对应Metadata Statement[[UAFAuthnrMetadata](#bib-UAFAuthnrMetadata)]中的attestationRootCertificate域中的其中一个证书。
1.3.3.1|UINT16 Length|认证证书长度
1.3.3.2|UINT8[] Certificate|单X.509 DER编码[[ITU-X690-2008](#bib-ITU-X690-2008)]的认证证书或认证证书链中的单证书（查看上面的描述）。
1.3 **(选择2)**|UINT16 Tag|TAG_ATTESTATION_BASIC_SURROGATE
1.3.1|UINT16 Length|结构长度
1.3.2|UINT16 Tag|TAG_SIGNATURE
1.3.2.1|UINT16 Length|signature长度
1.3.2.2|UINT8[] Signature|使用刚生成的UAuth.privTAG_UAFV1_KRD进行前面的结果。整个TAG_UAFV1_KRD，包括tag和length都**必须**加入到签名数据。

### 6.1.1.2 TAG_UAFV1_AUTH_ASSERTION
下面的TLV结构由authenticator在处理Sign command的时候生成。然后原封不动地发送到FIDO Server，并被server解析。该结构内嵌了TAG_UAFV1_SIGNED_DATA。

如果authenticator想要在TAG_UAFV1_SIGNED_DATA 上附加自定义数据（然后用认证密钥签名）——这些数据必须作为额外的tag包含在TAG_UAFV1_SIGNED_DATA。

如果authenticator想要发送额外的数据给FIDO Server而不签名——这些数据**必须**必须包含在TAG_UAFV1_AUTH_ASSERTION中，但不包含在TAG_UAFV1_SIGNED_DATA中。

序号|TLV结构|描述
----|----|----
1|UINT16 Tag|TAG_UAFV1_AUTH_ASSERTION
1.1|UINT16 Length|结构长度。
1.2|UINT16 Tag|TAG_UAFV1_SIGNED_DATA
1.2.1|UINT16 Length|结构长度。
1.2.2|UINT16 Tag|TAG_AAID
1.2.2.1|UINT16 Length|AAID长度
1.2.2.2|UINT8[] AAID|Authenticator Attestation ID
1.2.3|UINT16 Tag|TAG_ASSERTION_INFO
1.2.3.1|UINT16 Length|Assertion Information长度
1.2.3.2|UINT16 AuthenticatorVersion|厂商写入的authenticator版本
1.2.3.3|UINT8 AuthenticationMode|Authentication Mode表示用户是否明确地进行了确认，和表示是否有交易信息。0x01：表示用户已经明确确认了；0x02：表示交易已经展示给用户且用户已经使用authenticator明确确认了。
1.2.3.4|UINT16 SignatureAlgAndEncoding|签名算法和编码格式。参考[[UAFRegistry](#bib-UAFRegistry)]以获得有关支持的算法和其值。
1.2.4|UINT16 Tag|TAG_AUTHENTICATOR_NONCE
1.2.4.1|UINT16 Length|authenticator临时生成数——必须至少8 bytes的长度
1.2.4.2|UINT8[] AuthnrNonce|一个authenticator生成的临时随机数（字节码）。
1.2.5|UINT16 Tag|TAG_FINAL_CHALLENGE
1.2.5.1|UINT16 Length|最终挑战码长度
1.2.5.2|UINT8[] FinalChallenge|Command中提供的最终挑战码（字节码）
1.2.6|UINT16 Tag|TAG_TRANSACTION_CONTENT_HASH
1.2.6.1|UINT16 Length|交易内容哈希值。如果AuthenticationMode == 0x01，则该值长度为0。例如进行认证，而不是交易确认的时候。
1.2.6.2|UINT8[] TCHash|交易内容哈希(字节码)
1.2.7|UINT16 Tag|TAG_KEYID
1.2.7.1|UINT16 Length|KeyID长度
1.2.7.2|UINT8[] KeyID|(binary value of) KeyID
1.2.8|UINT16 Tag|TAG_COUNTERS
1.2.8.1|UINT16 Length|计数器长度
1.2.8.2|UINT32 SignCounter|签名计数器。表示这个authenticator一共执行了的签名的次数。
1.3|UINT16 Tag|TAG_SIGNATURE
1.3.1|UINT16 Length|签名长度
1.3.2|UINT8[] Signature|使用UAuth.priv对TAG_UAFV1_SIGNED_DATA结构进行签名的结果。整个TAG_UAFV1_SIGNED_DATA，包括tag和length都**必须**加入到签名数据。

## 6.1.2 UserVerificationToken
本规范没有指定究竟应该如何在authenticator中进行用户验证。验证被认为是一个authenticator，一个厂商，一个明确的操作。

本文档提供了“vendor_specific_UserVerify“命令（验证用户使用的authenticator内置技术）如何安全地绑定到UAF Register和Sign Command的一个例子。绑定通过一个叫`UserVerificationToken`的概念完成。这个绑定使得"vendor_specific_UserVerify"和"UAF Register/Sign"互相独立（解耦）。

UserVerificationToken的定义如下：
    * ASM调用“vendor_specific_UserVerify”命令。authenticator验证用户并返回`UserVerificationToken`。
    * ASM调用 UAF.Register/Sign命令并传入参数`UserVerificationToken`。authenticator验证`UserVerificationToken`的合法性，如果合法则执行FIDO操作。

UserVerificationToken这个概念不是规范的。authenticator可能以完全不同的方式实现这个绑定功能。例如一个authenticator厂商可能决定直接追加UAF Register请求到他们的"vendor_specific_UserVerify"命令，并当作一个单命令处理。（？？没理解）

如果`UserVerificationToken`绑定机制实现了，则应该满足下面其中一个标准，或实现一个机制来提供更简单或更好的安全性：
    * `UserVerificationToken`必须允许仅执行一个单UAF Register或UAF Sign操作。
    * `UserVerificationToken`必须和时间绑定，并允许在规定时间内执行多次UAF操作。

## 6.2 Commands
*该章节不是规范的。*

>规范
>可以和不同厂商的ASM交互的UAF Authenticators**必须**实现本章节的command接口。例如：
>* 核心功能由一个厂商生产的bound authenticator，但需要和另一个厂商开发的ASM交互
>* Roaming Authenticators

>规范
>和自定义ASM紧密耦合的UAF Authenticators**可以**实现不同的command接口。

所有的UAF Authenticators command和responses在语义上是相似的——它们都表现为TLV编码块。每个command前面的2bytes是command code。接收完一个command后，authenticator必须解析第一个TLV tag并了解是发过来的是哪个command。

### 6.2.1 GetInfo Command

#### 6.2.1.1 Command描述
该command返回内部authenticator的信息。可能会返回0或更多的authenticator。每个authenticator被赋值一个不同的`authenticatorIndex`，用于其他命令时引用该authenticator。
（译者注：一个UAF Authenticators可能包含多个internal authenticator。）

#### 6.2.1.2 Command结构
序号|TLV结构|描述
----|----|----
1|UINT16 Tag|TAG_UAFV1_GETINFO_CMD
1.1|UINT16 Length|整个Command的长度——此处必须为0

### 6.2.1.3 Command Response
序号|TLV结构|描述
----|----|----
1|UINT16 Tag|TAG_UAFV1_GETINFO_CMD_RESPONSE
1.1|UINT16 Length|Response长度
1.2|UINT16 Tag|TAG_STATUS_CODE
1.2.1|UINT16 Length|Status Code长度
1.2.2|UINT16 Value|Authenticator返回的Status Code Authenticator
1.3|UINT16 Tag|TAG_API_VERSION
1.3.1|UINT16 Length|API Version 长度（必须为0x0001）
1.3.2|UINT8 Version|Authenticator API Version （必须为0x01）。该版本表示authenticator支持这些command类型与其格式。
1.4|UINT16 Tag|TAG_AUTHENTICATOR_INFO （可能多次出现）
1.4.1|UINT16 Length|Authenticator Info长度
1.4.2|UINT16 Tag|TAG_AUTHENTICATOR_INDEX
1.4.2.1|UINT16 Length|AuthenticatorIndex 长度（必须为0x0001）
1.4.2.2|UINT8 AuthenticatorIndex|Authenticator Index
1.4.3|UINT16 Tag|TAG_AAID
1.4.3.1|UINT16 Length|AAID长度
1.4.3.2|UINT8[] AAID|由厂商设置的AAID
1.4.4|UINT16 Tag|TAG_AUTHENTICATOR_METADATA
1.4.4.1|UINT16 Length|Authenticator Metadata长度
1.4.4.2|UINT16 AuthenticatorType|表示authenticator是bound还是roaming，是first-还是second-factor only。ASM必须使用这个信息去明白authenticator是如何工作的。<br>预定义值：<br>0x0001 - 表示second-factor authenticator （不设置为first-factor）<br>0x0002 - 表示roaming authenticator （不设置为bound authenticator）<br>0x0004 - Key handles会在authenticator内部存储不会发给ASM<br>0x0008 - authenticator拥有内置UI界面，用于用户登录和登记。ASM不应该展示自己的UI。<br>0x0010 - authenticator拥有内置UI设置界面，并支持OpenSettings command。<br>0x0020 - authenticator期望在TAG_APPID被标为optional的command中，TAG_APPID会在command中出现。<br>0x0040 - 至少有一个用户已经登记到这个authenticator中。不支持用户登记的authenticator必须把设置这个位为1（如USER_VERIFY_NONE，USER_VERIFY_PRESENCE）。<br>（译者注：1.2版本新增 0x0080 - authenticator支持文档中提到的user verification token（UVTs）。参考[5.3 UserVerificationToken](#5.3)。）
1.4.4.3|UINT8 MaxKeyHandles|该authenticator一个command可以接收和处理的key handles数量最大值。该信息会被ASM在调用多个key handles的SIGN command的时候使用。
1.4.4.4|UINT32 UserVerification|User Verification method （定义在[[UAFRegistry](#bib-UAFRegistry)]）
1.4.4.5|UINT16 KeyProtection|Key Protection type （定义在[[UAFRegistry](#bib-UAFRegistry)]）。
1.4.4.6|UINT16 MatcherProtection|Matcher Protection type （定义在[[UAFRegistry](#bib-UAFRegistry)]）。
1.4.4.7|UINT16 TransactionConfirmationDisplay|Transaction Confirmation type （定义在[[UAFRegistry](#bib-UAFRegistry)]）。<br>NOTE<br>If Authenticator doesn't support Transaction Confirmation - this value must be set to 0.
1.4.4.8|UINT16 AuthenticationAlg|Authentication Algorithm （定义在[[UAFRegistry](#bib-UAFRegistry)]）。
1.4.5|UINT16 Tag|TAG_TC_DISPLAY_CONTENT_TYPE （optional）
1.4.5.1|UINT16 Length|content type长度。
1.4.5.2|UINT8[] ContentType|Transaction Confirmation Display Content Type。浏览[[UAFAuthnrMetadata](#bib-UAFAuthnrMetadata)]以获得更多的信息 on the format of this field。
1.4.6|UINT16 Tag|TAG_TC_DISPLAY_PNG_CHARACTERISTICS （optional,multiple occurrences permitted）
1.4.6.1|UINT16 Length|display characteristics information长度。
1.4.6.2|UINT32 Width|浏览[[UAFAuthnrMetadata](#bib-UAFAuthnrMetadata)]以获得更多的信息。
1.4.6.3|UINT32 Height|浏览[[UAFAuthnrMetadata](#bib-UAFAuthnrMetadata)]以获得更多的信息。
1.4.6.4|UINT8 BitDepth|浏览[[UAFAuthnrMetadata](#bib-UAFAuthnrMetadata)]以获得更多的信息。
1.4.6.5|UINT8 ColorType|浏览[[UAFAuthnrMetadata](#bib-UAFAuthnrMetadata)]以获得更多的信息。
1.4.6.6|UINT8 Compression|浏览[[UAFAuthnrMetadata](#bib-UAFAuthnrMetadata)]以获得更多的信息。
1.4.6.7|UINT8 Filter|浏览[[UAFAuthnrMetadata](#bib-UAFAuthnrMetadata)]以获得更多的信息。
1.4.6.8|UINT8 Interlace|浏览[[UAFAuthnrMetadata](#bib-UAFAuthnrMetadata)]以获得更多的信息。
1.4.6.9|UINT8[] PLTE|浏览[[UAFAuthnrMetadata](#bib-UAFAuthnrMetadata)]以获得更多的信息。
1.4.7|UINT16 Tag|TAG_ASSERTION_SCHEME
1.4.7.1|UINT16 Length|Assertion Scheme长度
1.4.7.2|UINT8[] AssertionScheme|Assertion Scheme （定义在[[UAFRegistry](#bib-UAFRegistry)]）
1.4.8|UINT16 Tag|TAG_ATTESTATION_TYPE （可能多次出现）
1.4.8.1|UINT16 Length|AttestationType长度
1.4.8.2|UINT16 AttestationType|Attestation Type值定义在[[UAFRegistry](#bib-UAFRegistry)]
1.4.9|UINT16 Tag|TAG_SUPPORTED_EXTENSION_ID （optional, 可能多次出现）
1.4.9.1|UINT16 Length|SupportedExtensionID长度
1.4.9.2|UINT8[] SupportedExtensionID|SupportedExtensionID，UTF-8编码的UINT8[]字符串

### 6.2.1.4 Status Codes
* `UAF_CMD_STATUS_OK`
* `UAF_CMD_STATUS_ERR_UNKNOWN`

## 6.2.2 Register Command
该command生成UAF registration断言。该断言可用于向FIDO Server注册authenticator。

### 6.2.2.1 Command结构
序号|TLV结构|描述
----|----|----
1|UINT16 Tag|TAG_UAFV1_REGISTER_CMD
1.1|UINT16 Length|Command长度
1.2|UINT16 Tag|TAG_AUTHENTICATOR_INDEX
1.2.1|UINT16 Length|AuthenticatorIndex长度（必须为0x0001）
1.2.2|UINT8 AuthenticatorIndex|Authenticator Index
1.3|UINT16 Tag|TAG_APPID（可选）
1.3.1|UINT16 Length|AppID长度
1.3.2|UINT8[] AppID|AppID（最多512字节）
1.4|UINT16 Tag|TAG_FINAL_CHALLENGE
1.4.1|UINT16 Length|Final Challenge长度
1.4.2|UINT8[] FinalChallenge|ASM提供的Final Challenge（最多32字节）
1.5|UINT16 Tag|TAG_USERNAME
1.5.1|UINT16 Length|Username长度
1.5.2|UINT8[] Username|ASM提供的Username（最多128字节）
1.6|UINT16 Tag|TAG_ATTESTATION_TYPE
1.6.1|UINT16 Length|AttestationType长度
1.6.2|UINT16 AttestationType|将要使用的Attestation Type
1.7|UINT16 Tag|TAG_KEYHANDLE_ACCESS_TOKEN
1.7.1|UINT16 Length|KHAccessToken长度
1.7.2|UINT8[] KHAccessToken|ASM提供的KHAccessToken（最多32字节）
1.8|UINT16 Tag|TAG_USERVERIFY_TOKEN（可选）
1.8.1|UINT16 Length|VerificationToken长度
1.8.2|UINT8[] VerificationToken|用户认证token

### 6.2.2.2 Command Response
序号|TLV结构|描述
----|----|----
1|UINT16 Tag|TAG_UAFV1_REGISTER_CMD_RESPONSE
1.1|UINT16 Length|Command长度
1.2|UINT16 Tag|TAG_STATUS_CODE
1.2.1|UINT16 Length|Status Code长度
1.2.2|UINT16 Value|Authenticator返回的Status code
1.3|UINT16 Tag|TAG_AUTHENTICATOR_ASSERTION
1.3.1|UINT16 Length|Assertion长度
1.3.2|UINT8[] Assertion|Registration Assertion（详情查看5.2.1 TAG_UAFV1_REG_ASSERTION）
1.4|UINT16 Tag|TAG_KEYHANDLE（可选）
1.4.1|UINT16 Length|key handle长度
1.4.2|UINT8[] Value|key handle（字节码）

### 6.2.2.3 Status Codes
* `UAF_CMD_STATUS_OK`
* `UAF_CMD_STATUS_ACCESS_DENIED`
* `UAF_CMD_STATUS_USER_CANCELLED`
* `UAF_CMD_STATUS_ATTESTATION_NOT_SUPPORTED`
* `UAF_CMD_STATUS_ERR_UNKNOWN`

### 6.2.2.4 指令介绍
authenticator必须遵循以下步骤（浏览下面的command结构的表格）（？？上面）

1. 如果authenticator有交易确认展示界面可以展示AppID，则需要保证（ASM）提供了`Command.TAG_APPID`，并在用户确认的时候进行展示。使用`TAG_APPID`更新`Command.KHAccessToken`：
    * 把Command.KHAccessToken和Command.TAG_APPID混合，结果赋值给Command.KHAccessToken。举个混合例子（使用加密哈希）：
        * 例：Command.KHAccessToken=hash(Command.KHAccessToken | Command.TAG_APPID)
    
    > NOTE
    
    > 这个方法允许我们避免在RawKeyHandle分开存储AppID。

2. 如果用户已经使用此authenticator进行了登记（通过生物识别验证，PIN验证或者类似的机制）—— 验证该用户。如果验证已经在之前的command完成了 —— 保证`Command.TAG_USERVERIFY_TOKEN`合法。
    1. 如果验证失败 —— 返回`UAF_CMD_STATUS_ACCESS_DENIED`
3. 如果用户还没有使用该authenticator进行登记，则引导用户执行登记流程。（如指纹录入）
    1. 如果登记失败 —— 返回`UAF_CMD_STATUS_ACCESS_DENIED`
    2. 如果用户明确的取消了该操作 —— 返回`UAF_CMD_STATUS_USER_CANCELLED `

4. 确认支持`Command.TAG_ATTESTATION_TYPE `。如果不支持该type —— 返回`UAF_CMD_STATUS_ATTESTATION_NOT_SUPPORTED`
5. 生成一个新的密钥对（UAuth.pub/UAuth.priv）
6. 创建RawKeyHandle
    1. 添加UAuth.priv
    2. 添加Command.KHAccessToken
    3. 如果是first-factor authenticator，添加Command.Username
7. 用Wrap.sym密钥打包RawKeyHandle
8. 创建TAG_UAFV1_KRD结构
    1.  如果是second-factor roaming authenticator - 把key handle作为TAG_KEYID的值。否则生成一个随机KeyID作为TAG_KEYID的值。
    2. 拷贝其它所有要求的数据域（参考 [TAG_UAFV1_REG_ASSERTION](#5.2.1)）
9. 根据提供的Command.AttestationType认证TAG_UAFV1_KRD。
10. 创建TAG_AUTHENTICATOR_ASSERTION
    1. 创建 TAG_UAFV1_REG_ASSERTION
        1. 拷贝其它所有要求的数据域（参考 [TAG_UAFV1_REG_ASSERTION](#5.2.1)）
        2. 如果这是first-factor authenticator —— 内部存储KeyID和key handle
        3. 如果是second-factor roaming authenticator —— 通过TAG_KEYHANDLE返回key handle
    2. 将整个TAG_UAFV1_REG_ASSERTION的TLV结构当作TAG_AUTHENTICATOR_ASSERTION的值
11. 返回TAG_UAFV1_REGISTER_CMD_RESPONSE
    1. 设置状态为`UAF_CMD_STATUS_OK`
    2. 添加TAG_AUTHENTICATOR_ASSERTION
    3. 如果需要外部存储key handle，添加TAG_KEY_HANDLE

> 规范
> authenticator**不可以**在验证用户（或登记用户（用户第一次使用authenticator））前处理`Register` command。
> authenticator**必须**在每次调用Register command时都必须生成唯一的UAuth密钥对。
> authenticator**应该**存储key handle在内部安全存储中，或加密传送给ASM。
> 对于silent authenticators，key handle**不可以**存储在FIDO Server，否则会使用户可能被跟踪但又没有为用户提供从本地设备上清除key handle的能力。
> 如果KeyID不是key handle（如second-factor roaming authenticator）——则其必须是一个唯一的无规律的byte数组且最大长度为32。其**必须**是同样AAID的设备中唯一的。
> > NOTE
> > 如果KeyID是随机生成的（而不是根据key handle计算出来的（比如hash））——则其应该存储KeyID到RawKeyHandle里面，使得authenticator在处理Sign command的时候可以根据RawKeyHandle获取到keyID。
> 如果authenticator不支持`SignCount`或`RegCounter`，则**必须**在TAG_UAFV1_KRD中设置它们为0。authenticator恢复出厂设置时，`RegCount`和`SignCount`**必须**设置为0。

## 6.2.3 Sign Command
这个command会生成UAF断言。断言稍后将会发送到之前进行注册的FIDO Server并被验证。

### 6.2.3.1 Command结构
序号|TLV结构|描述
----|----|----
1|UINT16 Tag|TAG_UAFV1_SIGN_CMD
1.1|UINT16 Length|Command长度
1.2|UINT16 Tag|TAG_AUTHENTICATOR_INDEX
1.2.1|UINT16 Length|AuthenticatorIndex长度（必须为0x0001）
1.2.2|UINT8 AuthenticatorIndex|Authenticator Index
1.3|UINT16 Tag|TAG_APPID（可选）
1.3.1|UINT16 Length|AppID长度
1.3.2|UINT8[] AppID|AppID(最多512字节)
1.4|UINT16 Tag|TAG_FINAL_CHALLENGE
1.4.1|UINT16 Length|Final Challenge长度
1.4.2|UINT8[] FinalChallenge|由ASM提供的Final Challenge（字节码）（最多32字节）
1.5|UINT16 Tag|TAG_TRANSACTION_CONTENT（可选）
1.5.1|UINT16 Length|Length of Transaction Content
1.5.2|UINT8[] TransactionContent|由ASM提供的Transaction Content（字节码）
1.6|UINT16 Tag|TAG_KEYHANDLE_ACCESS_TOKEN
1.6.1|UINT16 Length|KHAccessToken长度
1.6.2|UINT8[] KHAccessToken|由ASM提供的KHAccessToken（最多32字节）
1.7|UINT16 Tag|TAG_USERVERIFY_TOKEN（可选）
1.7.1|UINT16 Length|User Verification Token长度
1.7.2|UINT8[] VerificationToken|User Verification Token
1.8|UINT16 Tag|TAG_KEYHANDLE（可选，可能多次出现）
1.8.1|UINT16 Length|KeyHandle长度
1.8.2|UINT8[] KeyHandle|key handle（字节码）

### 6.2.3.2 Command Response
序号|TLV结构|描述
----|----|----
1|UINT16 Tag|TAG_UAFV1_SIGN_CMD_RESPONSE
1.1|UINT16 Length|整个Command Response的长度
1.2|UINT16 Tag|TAG_STATUS_CODE
1.2.1|UINT16 Length|Status Code长度
1.2.2|UINT16 Value|authenticator返回的Status code
1.3**（选择1）**|UINT16 Tag|TAG_USERNAME_AND_KEYHANDLE（可选，可能多次出现）<br>这个TLV tag可以用于表示多个（>=1）{Username, KeyHandle}组合。<br>如果这个tag出现，则TAG_AUTHENTICATOR_ASSERTION不可以出现
1.3.1|UINT16 Length|structure长度
1.3.2|UINT16 Tag|TAG_USERNAME
1.3.2.1|UINT16 Length|Username长度
1.3.2.2|UINT8[] Username|Username
1.3.3|UINT16 Tag|TAG_KEYHANDLE
1.3.3.1|UINT16 Length|KeyHandle长度
1.3.3.2|UINT8[] KeyHandle|key handle（字节码）
1.3**（选择2）**|UINT16 Tag|TAG_AUTHENTICATOR_ASSERTION（可选）<br>如果这个tag出现，则TAG_USERNAME_AND_KEYHANDLE不可以出现
1.3.1|UINT16 Length|Assertion长度
1.3.2|UINT8[] Assertion|Authentication assertion generated by the authenticator（查看[TAG_UAFV1_AUTH_ASSERTION](#5.2.2)）。

### 6.2.3.3 Status Codes

* `UAF_CMD_STATUS_OK`
* `UAF_CMD_STATUS_ACCESS_DENIED`
* `UAF_CMD_STATUS_USER_NOT_ENROLLED`
* `UAF_CMD_STATUS_USER_CANCELLED`
* `UAF_CMD_STATUS_CANNOT_RENDER_TRANSACTION_CONTENT`
* `UAF_CMD_STATUS_ERR_UNKNOWN`

# 6.2.3.4 Command描述
> NOTE
> First-factor authenticator应该将该命令实现分为两个阶段：
> 1. 第一阶段：验证KHAccessToken后，authenticator发现存在多个key handles。该阶段中，authenticator必须返回所有key handles的username+key handles组合给ASM。
> 2. 第二阶段：用户选择了一个username后，ASM再次调用Sign Command，此时command中只有一个key handle，authenticator根据此key handle返回UAF断言。
>
> 如果second-factor authenticator收到多与一个合法的key handle，则必须选择第一个，并忽略其他。
> 
> 该命令实现分为两个阶段，可以保证一次command调用只会生成一个断言。

authenticator必须遵循以下步骤：

1. 如果authenticator有交易确认展示界面可以展示AppID，则需要保证（ASM）提供了`Command.TAG_APPID`，并在用户确认的时候进行展示。使用`TAG_APPID`更新`Command.KHAccessToken`：
    * 把Command.KHAccessToken和Command.TAG_APPID混合，结果赋值给Command.KHAccessToken。举个混合例子（使用加密哈希）：
        * 例：Command.KHAccessToken=hash(Command.KHAccessToken | Command.TAG_APPID)
    
    > NOTE
    
    > 这个方法允许我们避免在RawKeyHandle分开存储AppID。
2.  如果用户已经在此authenticator登记过（如生物识别录入，PIN初始化等），则验证该用户。如果在之前的command中已经验证过，则保证`Command.TAG_USERVERIFY_TOKEN`合法。（译者注：可以做成：如果`Command.TAG_USERVERIFY_TOKEN`存在，则验证是否合法。如果不合法或者过期，则重新进行验证）
    1.  如果认证失败——返回UAF_COD_STATUS_ACCESS_DENIED
    2.  如果用户取消了操作——返回UAF_COMD_STATUS_USER_CANCELLED
3. 如果用户还没在此authenticator登记过，则返回UAF_CMD_STATUS_USER_NOT_ENROLLED
4. 使用Wrap.sym解包所有Command.TAG_KEYHANDLE提供的key handles。
    1. 如果是first-factor roaming authenticator：
        1. 如果提供了Command.TAG_KEYHANDLE，则该列表位KeyIDs。使用这些KeyIDs去找到内部存储中的key handles。
        2. 如果没有提供Command.TAG_KEYHANDLE，解包内部存储的所有key handles
            >译者注：
            >虽然文档其他地方只是说存储在authenticator外部的key handles都要加密，但这个步骤中表明内部存储的key handles也要用Wrap.sym加密
5. 根据Command.KHAcessToken过滤RawKeyHandles（RawKeyHandle.KHAccessToken == Command.KHAccessToken）
6. 如果过滤后RawKeyHandles数量为0，则返回`UAF_CMD_STATUS_ACCESS_DENIED`
7. 如果过滤后RawKeyHandles数量 > 1
    1. 如果是second-factor authenticator，则选择第一个RawKeyHandle然后调到步骤8。
    2. 拷贝所有{Command.KeyHandle, RawKeyHandle.username}到TAG_USERNAME_AND_KEYHANDLE tag。
        * 如果是first-factor roaming authenticator，则返回的TAG_USERNAME_AND_KEYHANDLES必须按照key handle注册日期排序（最后注册的key handle必须排在最后）
        （译者注：1.2版本新增了提示如下）
        > NOTE
        > 如果发现多个拥有同样username的key handles，first-factor roaming authenticator应该只保留最近注册的，并删除其余key handles。这样可以避免authenticator持有无用的（旧的）私钥，这些私钥可能会在删除新生成的key handles时重新变成可用的。
    3. 拷贝TAG_USERNAME_AND_KEYHANDLE到TAG_UAFV1_SIGN_CMD_RESPONSE并返回
8. 如果RawKeyHandles数量为1
    1. 创建TAG_UAFV1_SIGNED_DATA并设置TAG_UAFV1_SIGNED_DATA.AuthenticationMode = 0x01
    2. 如果`TransactionContent`非空
        * 如果这是一个slient authenticator，则返回`UAF_CMD_STATUS_ACCESS_DENIED`
        * 如果这个authenticator不支持交易
        * 确认（在先前的GetInfo Command中TransactionConfirmationDisplay设置为0），则返回`UAF_CMD_STATUS_ACCESS_DENIED`
        * 如果authenticator拥有内置的交易显示确认模块，则显示`Command.TransactionContent`和`Command.TAG_APPID`（可选）并等待用户确认：
            * 如果用户取消了交易，返回`UAF_CMD)STATUS_USER_CANCELLED`
            * 如果交易内容无法显示，则返回`UAF_CMD_STATUS_CANNOT_RENDER_TRANSACTION_CONTENT`
        * 计算TranscationContent的hash值
            * TAG_UAFV1_SIGNED_DATA.TAG_TRANSACTION_CONTENT_HASH = hash(Command.TransactionContent)
            * 设置TAG_UAFV1_SIGNED_DATA.AuthenticationMode = 0x02
    3. 创建TAG_UAFV1_AUTH_ASSERTION
        1. 填充TAG_UAFV1_SIGNED_DATA 剩余的数据域
            * SignCounter增加1并放入TAG_UAFV1_SIGNED_DATA
            * 复制所有强制要求的数据域（参考[TAG_UAFV1_AUTH_ASSERTION](#5.2.2)
            * 如果 TAG_UAFV1_SIGNED_DATA.AuthenticationMode == 0x01 —— 设置TAG_UAFV1_SIGNED_DATA.TAG_TRANSACTION_CONTENT_HASH.Length = 0
            * 使用UAuth.priv签名TAG_UAFV1_SIGNED_DATA
    4. 把整个TAG_UAFV1_AUTH_ASSERTION作为TAG_AUTHENTICATOR_ASSERTION的值放入
    5. 拷贝TAG_AUTHENTICATOR_ASSERTION到TAG_UAFV1_SIGN_CMD_RESPONSE并返回结果
    
    > 规范
    > Authenticator**不可以**不先验证用户而去处理Sign command
    > Authenticator**不可以**不先验证用户而展示username
    > Bound authenticators**不可以**不先验证KHAccessToken而先去处理Sign command
    > UAuth.priv密钥在解析为明文的时候，**必须**留存在Authenticator的安全区域。UAuth.priv保护边界声明在[[UAFAuthnrMetadata](#bib-UAFAuthnrMetadata)]的`Metadata.keyProtection`中。
    > 如果Authenticator的Metadata表示其支持“交易确认显示”（Transaction Confirmation display）——则其**必须**显示提供的交易信息并在TAG_UAFV1_SIGNED_DATA加入交易信息的哈希值
    > Silent Authenticators**不可以**工作在first-factor模式，以遵循[[FIDOSecRef](#bib-FIDOSecRef)]中的假设
    > 如果Authenticator不支持`SignCounter`，则在TAG_UAFV1_SIGNED_DAT中**必须**设置`SignCounter`为0。当Authenticator恢复出厂设置时`SignCounter`必须设置为0，以遵循[[FIDOSecRef](#bib-FIDOSecRef)]中的假设
    >
    > 一些Authenticators可能支持“交易确认显示”功能，但不是在Authenticator内，而是在ASM的边界中。典型例子是基于软件的“交易确认显示”。当处理提供了交易的Sign command的时候，这些Authenticators应该假设他们的确有一个内置的“交易确认显示”，并讲交易信息的哈希值包括在最后的断言中，但并不显示东西给用户。不仅如此，这些Authenticator的Metadata文件**必须**明确显示交易确认显示”的类型。典型例子是“交易确认显示”的flag为TRANSACTION_CONFIRMATION_DISPLAY_ANY或TRANSACTION_CONFIRMATION_DISPLAY_PRIVILEGED_SOFTWARE。查看[[UAFRegistry](#bib-UAFRegistry)]了解“交易确认显示”的flag类型。

## 6.2.4 Deregister Command
该command删除Authenticator上已注册的UAF凭据。

### 6.2.4.1 Command Structure
序号|TLV结构|描述
----|----|----
1|UINT16 Tag|TAG_UAFV1_DEREGISTER_CMD
1.1|UINT16 Length|整个Command的长度
1.2|UINT16 Tag|TAG_AUTHENTICATOR_INDEX
1.2.1|UINT16 Length|AuthenticatorIndex长度（必须为0x0001)
1.2.2|UINT8 AuthenticatorIndex|Authenticator Index
1.3|UINT16 Tag|TAG_APPID（可选）
1.3.1|UINT16 Length|AppID长度
1.3.2|UINT8[] AppID|AppID（最大512字节）
1.4|UINT16 Tag|TAG_KEYID
1.4.1|UINT16 Length|KeyID长度
1.4.2|UINT8[] KeyID|ASM提供的KeyID（字节码）
1.5|UINT16 Tag|TAG_KEYHANDLE_ACCESS_TOKEN
1.5.1|UINT16 Length|KeyHandle Access Token长度
1.5.2|UINT8[] KHAccessToken|ASM提供的KeyHandle Access Token（字节码）（最大32字节）

### 6.2.4.2 Command Response
序号|TLV结构|描述
----|----|----
1|UINT16 Tag|TAG_UAFV1_DEREGISTER_CMD_RESPONSE
1.1|UINT16 Length|整个Command Response的长度
1.2|UINT16 Tag|TAG_STATUS_CODE
1.2.1|UINT16 Length|Status Code长度
1.2.2|UINT16 StatusCode|Authenticator返回的StatusCode

### 6.2.4.3 Status Codes
* `UAF_CMD_STATUS_OK`
* `UAF_CMD_STATUS_ACCESS_DENIED`
* `UAF_CMD_STATUS_CMD_NOT_SUPPORTED`
* `UAF_CMD_STATUS_ERR_UNKNOWN`

### 6.2.4.4 Command描述
Authenticator必须遵循以下步骤：

1. 如果authenticator有交易确认展示界面可以展示AppID，则需要保证（ASM）提供了`Command.TAG_APPID`，并在用户确认的时候进行展示。使用`TAG_APPID`更新`Command.KHAccessToken`：
    * 把Command.KHAccessToken和Command.TAG_APPID混合，结果赋值给Command.KHAccessToken。举个混合例子（使用加密哈希）：
        * 例：Command.KHAccessToken=hash(Command.KHAccessToken | Command.TAG_APPID)
    
    > NOTE
    
    > 这个方法允许我们避免在RawKeyHandle分开存储AppID。
2. 如果该authenticator没有自行存储key handles，则返回`UAF_CMD_STATUS_CMD_NOT_SUPPORTED`
3. 寻找匹配Command.KeyID的KeyHandle
4. 使用Wrap.sym解包KeyHandle
5. 保证RawKeyHandle.KHAccessToken == Command.KHAccessToken
        * 如果不相等则抛出`UAF_CMD_STATUS_ACCESS_DENIED`
6.  删除该KeyHandle
7. 返回`UAF_CMD_STATUS_OK`

（译者注：1.2新增了没提供KEYID的情况：
 * 如果`TAG_KEYID`长度为0，则
    * 如果提供了`TAG_APPID`，则
        * 对于每个匹配`TAG_APPID`的KeyHandle：
            1. 如果RawKeyHandle.KHAccessToken == Command.KHAccessToken，则删除该KeyHandle，否则标记发生了错误（译者注：意思是还是要先删掉所有可以删的KeyHandle）
        * 如果触发了错误，则返回UAF_CMD_STATUS_ACCESS_DENIED
 
 ）

> 规范
> Bound authenticators**不可以**不验证KHAccessToken而先处理Deregister command。

## 6.2.5 OpenSettings Command
该command告诉Authenticator开启其内置的UI设置界面（如修改密码，记录新指纹等）。

如果不支持该功能，Authenticator必须返回`UAF_CMD_STATUS_CMD_NOT_SUPPOERTED`。

### 6.2.5.1 Command Structure
序号|TLV结构|描述
----|----|----
1|UINT16 Tag|TAG_UAFV1_OPEN_SETTINGS_CMD
1.1|UINT16 Length|整个Command的长度
1.2|UINT16 Tag|TAG_AUTHENTICATOR_INDEX
1.2.1|UINT16 Length|AuthenticatorIndex长度（必须为0x0001)
1.2.2|UINT8 AuthenticatorIndex|Authenticator Index

### 6.2.5.2 Command Response
序号|TLV结构|描述
----|----|----
1|UINT16 Tag|TAG_UAFV1_OPEN_SETTINGS_CMD_RESPONSE
1.1|UINT16 Length|整个Command Response的长度
1.2|UINT16 Tag|TAG_STATUS_CODE
1.2.1|UINT16 Length|Status CSignCounterode长度
1.2.2|UINT16 StatusCode|Authenticator返回的StatusCode

### 6.2.5.3 Status Codes
* `UAF_CMD_STATUS_OK`
* `UAF_CMD_STATUS_CMD_NOT_SUPPORTED`
* `UAF_CMD_STATUS_ERR_UNKNOWN`

# 7. KeyIDs与key handles

*该章节不是规范。*

该文档根据其在处理command时的特性和行为差异一共定义了4种Authenticator。其中一个主要的差别是它们存储和处理key handles的差异。该章节试图描述所有Authenticator在处理相关command的时候的行为。

## 7.1 first-factor Bound Authenticator
**Register Command：**

Authenticator不存储key handles。KeyHandles由ASM存储并提供。

KeyID为随机生成的32位随机数（或简单的就做成KeyHandle的hash值）

**Sign Command：**

当用户seesion不存在的时候（没有cookies，是一个刚清理的机器）。Server不会返回KeyID（因为Server不知道返回哪个KeyID）。这种情况下，**ASM**选择所有的key handles然后发给Authenticator。

在按步骤进行authentication的时候（此时有用户session），Server会提供相应的KeyID。**ASM**选择KeyID对应的key handles然后发给Authenticator。

**Deregister Command：**

因为Authenticator不存储任何key handles，所以Authenticator没什么需要删除的。

**ASM**找到KeyID对应的KeyHandle然后删除。

## 7.2 2ndF Bound Authenticator
**Register Command：**

Authenticator不存储key handles。KeyHandles由ASM存储并提供。

KeyID为随机生成的32位随机数（或简单的就做成KeyHandle的hash值）

**Sign Command：**

没有KeyID的时候Authenticator无法操作，即Authenticator无法在无用户seesion的环境中操作（没有cookies，是一个刚清理的机器）。

在按步骤进行authentication的时候（此时有用户session），Server会提供相应的KeyID。**ASM**选择KeyID对应的key handles然后发给Authenticator。

**Deregister Command：**

因为Authenticator不存储任何key handles，所以Authenticator没什么需要删除的。

**ASM**找到KeyID对应的KeyHandle然后删除。


## 7.3 first-factor Roaming Authenticator
**Register Command：**

Authenticator存储key handles到自己的内部存储空间。ASM不会发给Authenticator。

KeyID为随机生成的32位随机数（或简单的就做成KeyHandle的hash值）

**Sign Command：**

当用户seesion不存在的时候（没有cookies，是一个刚清理的机器）。Server不会返回KeyID（因为Server不知道返回哪个KeyID）。这种情况下，**Authenticator**选择匹配该AppID的所有key handles

在按步骤进行authentication的时候（此时有用户session），Server会提供相应的KeyID。**Authenticator**选择KeyID对应的key handles然后发给Authenticator。

**Deregister Command：**

**Authenticator**找到KeyID对应的KeyHandle然后删除。



## 7.4 2ndF Roaming Authenticator
**Register Command：**

Authenticator和ASM都不存储key handles。KeyHandles（将取代KeyID）将发送给Server，由Server存储到用户记录中。从Server的角度来说，KeyHandles被当作KeyID。实际上KeyID就是KeyHandles。

**Sign Command：**

没有KeyID的时候Authenticator无法操作，即Authenticator无法在无用户seesion的环境中操作（没有cookies，是一个刚清理的机器）。

在按步骤进行authentication的时候（此时有用户session），Server会提供相应的KeyID（实际上是KeyHandle）。**Authenticator**找到正确的key handles然后使用它（？？直接提供了还需要找吗？）

**Deregister Command：**

因为Authenticator和ASM不存储任何key handles，所以client端没什么需要删除的。

# 8. Command的访问控制
*该章节是规范。*

FIDO Authenticators可能会实现不同的机制去管理command访问控制。

下面的表格总结了各个command访问控制要求。

所有的UAF Authenticators**必须**满足下面的访问控制要求。

Authenticator厂商**可能**会提供额外的安全机制。

下面表格用到的词汇：
NoAuth - 不需要访问控制
UserVerify - 明确的用户验证
KHAccessToken - 调用者必须持有KHAccessToken
KeyHandleList - 调用者必须持有KeyHandleList
KeyID - 调用者必须持有KeyID

Command|First-factor Bound Authenticator|2ndF Bound Authenticator|First-factor Roaming Authenticator|2ndF Roaming Authenticator
---|---|---|---|---
GetInfo | NoAuth | NoAuth | NoAuth | NoAuth
OpenSettings | NoAuth | NoAuth | NoAuth | NoAuth
Register | UserVerify | UserVerify | UserVerify | UserVerify
Sign | UserVerify KHAccessToken KeyHandleList | UserVerify KHAccessToken KeyHandleList | UserVerify KHAccessToken | UserVerify KHAccessToken KeyHandleList
Deregister | KHAccessToken KeyID | KHAccessToken KeyID | KHAccessToken KeyID | KHAccessToken KeyID

*表格 1:Command访问控制*

# 9. 与其它标准的关系
*该章节不是规范。＊

与UAF authenticator有关的当前存在的标准说明文档有[[TPM](#bib-TPM)]，[[TEE](#bib-TEE)]和[[SecureElement](#bib-SecureElement)]。

实施这些标准的硬件模块可以通过它们的扩展机制来纳入UAF功能，如通过加载安全应用（trustlets，applets等）。不支持这样的扩展机制模块无法在UAF框架内得到充分的利用。

## 9.1 TEE
为了让TEE支持UAF，可设计一个的特殊Trustlet（在TEE内部运行的可信赖应用），实现本文档中指定的UAF Authenticator功能，并实现一些用户认证技术（生物特征识别，密码或其它东西）

需要额外创建一个知道如何调用Trustlet的ASM。

## 9.2 Secure Elements
为了让SE支持UAF，可设计一个的特殊Applets（在SE内部运行的可信赖应用），实现本文档中指定的UAF Authenticator功能，并实现一些用户认证技术（生物特征识别，密码或其它东西）

需要额外创建一个知道如何调用Applets的ASM。

## 9.3 TPM
TPM通常内置有认证模块支持，但支持的TPM认证模块和UAF基本认证模块不兼容。UAF的未来改进计划可能会包括兼容认证。

TPM通畅还有一个内置的PIN认证功能，这个功能可以被UAF利用。为了让这个TPM模块支持UAF，厂商需要写一个如下的ASM：
    * 通过调用TPM API转换UAF数据为TPM数据
    * 使用TPM API创建断言（TLV）
    * 向FIDO UAF Client报告自己是一个合法的UAF authenticator

为TPM设计的一个特殊的断言方案也必须是被FIDO联盟创建和发布的(see [[UAFAuthnrMetadata](#bib-UAFAuthnrMetadata)]）。当FIDO Server接收到这个断言方案的断言，则将这个数据当作是TPM生成的数据并解析和验证。

## 9.4 不可靠的传输
本文档描述的命令结构是基于可靠传输的假设上的，并且不支持应用层检测或修正例如乱序，重复，丢失和信息被修改这些问题。如果ASM和Authenticator之间的传输层不可靠，则可能需要一个UAF规范之外的机制来发现和修正传输错误。

# A. 安全指南
*本章节不是规范。*

分类|建议
---|---
AppIDs and KeyIDs           |   在进行用户验证前，authenticator不可以以原文的形式返回注册过的AppIDs和KeyIDs（译者注：先验证再解密）。<br>如果入侵者成功物理访问到了roaming authenticator，它不应该轻易的获取到AppIDs和KeyIDs。
Attestation Private Key     |   Authenticators必须把认证私钥当做非常敏感的数据来保护自己的认证私钥。authenticator整体的安全性基于该密钥的保护水平。<br><br>强烈建议在一个防篡改硬件模块中存储和操作该私钥，如[[SecureElement](#bib-SecureElement)]。<br><br>registration断言方案认为，authenticator拥有对使用认证私钥进行签名的独占控制权。<br><br>FIDO Authenticators必须保证认证私钥：<br><br>1. 只用于认证key生成（only used to attest authentication keys generated），并由authenticator使用FIDO定义的数据格式KeyRegistrationData来保护该私钥。<br>2. 永远不能被authenticator的安全边界之外访问。<br><br>认证操作必须在某种程度上实现两个不同的依赖方不能连接registrations操作，authentications操作或其他交易操作（参考[[UAFProtocol](#bib-UAFProtocol)]）。
Certifications              |   厂商的authenticators尽可能的通过通用的安全标准认证，例如[[FIPS140-2](#bib-FIPS140-2)]，[[CommonCriteria](#bib-CommonCriteria)]或类似的安全标准。通过这样的认证可以让authenticator更好的实现UAF协议。
Cryptographic (Crypto) Kernel|  加密核心是authenticator的一个实现了UAF需要的加密功能的模块（如密钥生成，签名，打包解包密钥等），并具有对UAuth.priv，认证私钥，Wrap.sym的访问权限。<br><br>为了安全，该模块最好和UAuth.priv，Att.priv和Wrap.sym拥有相同的安全边界。如果安全边界不同，则实现必须保证它们在同一个模块中时有同样的安全等级。<br><br>强烈建议在一个防篡改硬件模块中存储和操作该私钥，如[[TEE](#bib-TEE)]。<br><br>在模块有物理入侵和侧信道攻击威胁的情况下，强烈建议使用防篡改硬件模块。<br><br>基于软件的authenticators必须确保使用现代代码保护技术和混淆技术来保护该模块，和白盒加密技术保护相关密钥。<br><br>Authenticators需要使用高质量的随机源来保证好的随机数生成器，来进行：<br><br>1. 生成认证密钥<br>2. 生成签名<br>3. 计算authenticator生成的挑战码<br><br>authenticator的随机数生成器（RNG）应该是不能被禁用的或可预测的。<br><br>如果authenticator没有足够的随机性去生成健壮的随机数，它应该安全地失败。<br><br>查看表格下面的Randon numbers来获取更多信息。
KeyHandle                   |   强烈建议使用经过认证的加密算法来使用Wrap.sym对key handles进行解包。像AES-GCM和AES-CCM算法是最合适该操作的算法。
Liveness Detection（活体检测）|   用户验证函数应该包括活体检测[[NSTCBiometrics](#bib-NSTCBiometrics)]，比如一个可以保证样本提交者的是一个活人的技术。<br><br>在基于PIN匹配的情况下，则应该使用[[TEESecureDisplay](#bib-TEESecureDisplay)]实现，以保证恶意软件无法模拟PIN重试。
Matcher（译者注：如检测指纹是否匹配的组件）|  根据定义，匹配器组件是authenticator中的一部分。这里不强制限制authenticator匹配器的实现，但实现者需要保证匹配器和authenticator的其他部分之间有合适的安全边界。<br><br>篡改匹配器模块可能会导致严重的安全后果。强烈建议该模块放置在authenticator的完整边界中，并能够检测篡改。<br><br>强烈建议在一个信任的运行环境中（如[[TEE](#bib-TEE)]）或在一个安全的器件中（如[[SecureElement](#bib-SecureElement)]）运行该模块。<br><br>拥有分离的matcher和加密核心模块的Authenticators应该实现一个机制，该机智允许加密核心安全地从匹配器模块中接收断言，并标明用户的本地认证状态。<br><br>基于软件的Authenticators（如果不在信任的运行环境中）必须保证使用现有技术的代码保护和混淆技术来保护该模块。<br><br>当Authenticator接收到不合法的UserVerificationToken的时候，应该把其视为攻击，并不再使用缓存的UserVerificationToken。<br><br>一个UserVerificationToken有效时间应不超过10秒。<br><br>Authenticators必须为他们的匹配器实现反攻击保护。<br><br>基于生物特征识别的authenticators必须保护收集到的生物特征数据（如指纹）和参照数据（模板），并保证生物特征数据永远不会离开authenticators的安全边界。<br><br>匹配器必须只接受已登记用户的认证参考数据，即authenticators不可以包含任何默认密码或默认生物识别参考数据。
Private Keys (UAuth.priv and Attestation Private Key)|本文档要求（a）attestation 密钥只用于attestation 用途（译者注：对称密钥，用于生成非对称密钥和KRD加解密，只有一个，即上面的Attestation Private Key），和（b）authentication 密钥只用于FIDO authentication用途（译者注：用来签名的密钥，可能有多个）。相关的待签名对象（如Key Registration Data和SignData）被设计成减少如下攻击的可能性：<br><br>1. 以某个指定的tag开头，并以此标记为FIDO对象<br>2. 包含一个authenticator生成的随机数。因此所有待签名对象都是基本上是唯一的。<br>拥有一个只允许很少不受控制的值（即不是由authenticator生成或修改的值）的结构。
Random Numbers          |   FIDO Authenticator使用其随机数生成器去生成认证密钥对，客户端的挑战码，和（可能）创建ECDSA签名。弱的随机数会使FIDO Authenticator不足以应对对应的攻击。让FIDO Authenticator使用强随机数是非常重要的。<br><br>authenticators使用（伪）随机数应能成功的通过[[Coron99](#bib-Corom99)]的随机测试，且应该遵循[[SP800-90b](#bib-SP800-90b)]指南。<br><br>另外，当requests中有`ServerChallenge`时，authenticators可能会选择结合FIDO Server提供的熵（详情[[UAFProtocol](#bib-UAFProtocol)]）。<br><br>当要混合多个熵源时，应使用一个合适的混合函数，例如[[RFC4086](#bib-RFC4086)]。
RegCounter                  |   `RegCounter`为依赖方提供反欺诈信息。依赖方使用`RegCounter`可以检测authenticators是否被多次注册。<br><br>如果实现了`RegGounter`：应保证：<br><br>1. 因registration operation作而递增，且<br>2. 不可被除此之外的操作或修改（如API调用等）<br><br>一个注册计数器应该实现为全局计数器，即所有AppID的registrations使用同一个计数器。该全局计数器应该每次registration operation后增1。<br><br>NOTE：RegCounter值不应因`Deregistration`而降值。
SignCounter             |   当攻击者可以从注册过的authenticator中解包Uauth.priv时，该key可以独立于原authenticator使用。这被认为是该authenticator的克隆。<br><br>对Uauth私钥的好的保护措施是一个防止克隆authenticators的方法。在一些情况下该保护措施可能并不足够。<br><br>如果Authenticator有`SignCounter`，则FIDO Server会有额外的方法来检测克隆authenticators。<br><br>如果实现了`SignCounter`：确保<br><br>1. 任意的authentication和交易确认操作都会使其增值，且<br>2. 不可被除此之外的操作或修改（如API调用等）<br><br>签名计数器应被实现为专用于每个私钥，以确保用户的隐私。<br><br>每当UAuth.priv密钥对一个断言签名，`SignCounter`应该增1。<br><br>每当UAuth密钥删除时，`SignCounter`也应该同时删除。<br><br>如果authenticator无法管理太多的签名计数器，则所有私钥应该使用同一个全局的签名计数器。当任意一个UAuth.priv签名断言的时候，全局的`SignCounter`应该增加一个随机的正数。<br><br>（译者注：1.2版本新增提示：<br>NOTE：<br>`SignCounter`在registration response中为0可能由多种可能导致的。而如果authentication response的`SignCounter`值为0的表明该authenticator不支持`SignCounter`这个概念。）
Transaction Confirmation Display|   交易确认显示必须保证向用户展示交易内容，如不被其他显示元素覆盖且可以清楚辨认。详情[[CLICKJACKING](#bib-CLICKJACKING)]了解威胁及其可选的反制措施的一些例子。<br><br>更多的参考可参见[[TEESecureDisplay](#bib-TEESecureDisplay)]。
UAuth.priv                  |   Authenticators必须把认证私钥当做非常敏感的数据来保护自己的认证私钥。authenticator整体的安全性基于该密钥的保护水平。<br><br>强烈建议在一个信任的运行环境中生成，存储和操作该私钥。<br><br>在模块有物理入侵和侧信道攻击威胁的情况下，强烈建议使用防篡改硬件模块。<br><br>FIDO Authenticators必须保证UAuth.priv密钥：<br><br>1. 专用于同一个依赖方的特定账户（依赖方以AppID标识）<br>2. 用好的随机数生成器以足够的熵生成。registration和authentication operations期间FIDO Server提供的挑战码应该混合进熵池（entropy pool）以获得额外的熵。<br>3. 永远不要直接暴露，如永远保留FIDO Authenticator的独占控制权<br>4. 只用于定义的认证模块，如：<br>&nbsp;&nbsp;&nbsp;&nbsp;1. 认证已经生成过密钥的应用程序（以AppID标识），或<br>&nbsp;&nbsp;&nbsp;&nbsp;2. 确认已经生成过密钥的应用程序（以AppID标识）的交易，或<br>&nbsp;&nbsp;&nbsp;&nbsp;3. 只用于创建FIDO定义的数据格式，如KRD，SignData。
Username                    |   用户名在除SIGN command以外的任何情况下不可以以明文方式返回。其他所有情况下用户名必须保存在`KeyHandle`中。
Verification Reference Data |   认证参考数据，如指纹模板或PIN的参考值，属于authenticator的定义部分。但对authenticator的实现不做任何特定限制，但实现者需要保证有何事的安全边界将所有authenticator绑定到一起。。
Wrap.sym                    |   如果authenticator有打包密钥（Wrap.sym），则authenticator必须把它当做非常敏感的数据来保护。authenticator整体的安全性基于该密钥的保护水平。<br><br>Wrap.sym密钥强度必须等于或高于存储在RawKeyHandle中的机密的强度。参照[[SP800-57](#bib-SP800-57)] 和 [[SP800-38F](#bib-SP800-38F)]发布文档获得有关选择正确打包算法和正确实现的更多信息。<br><br>强烈建议在一个信任的运行环境中生成，存储和操作该私钥。<br><br>在模块有物理入侵和侧信道攻击威胁的情况下，强烈建议使用防篡改硬件模块。<br><br>如果authenticator使用Wrap.sym，则其必须保证调用者无法区别解包损坏的KeyHandle和解包有错误信息的数据（如其他源生成的非法KeyHandle）。


# B. 图片列表

[Fig. 1 UAF Authenticator Commands](#fig-uaf-authenticator-commands)

[Fig. 2 FIDO Authenticator Logical Sub-Components](#logical-sub-components)

# C. 参考
## C.1 规范参考
<section id="references" class="appendix" typeof="bibo:Chapter" resource="#ref" rel="bibo:Chapter"><dt id="bib-Coron99">[Coron99]</dt><dd rel="dcterms:requires">J. Coron and D. Naccache <a href="http://www.jscoron.fr/publications/universal.pdf"><cite>An accurate evaluation of Maurer's universal test</cite></a>. LNCS 1556, February 1999, URL: <a href="http://www.jscoron.fr/publications/universal.pdf">http://www.jscoron.fr/publications/universal.pdf</a>
</dd><dt id="bib-FIDOGlossary">[FIDOGlossary]</dt><dd rel="dcterms:requires">R. Lindemann, D. Baghdasaryan, B. Hill, J. Hodges, <cite>FIDO Technical Glossary</cite>. FIDO Alliance Proposed Standard. URLs: <br>HTML: <a href="https://fidoalliance.org/specs/fido-uaf-v1.0-ps-20141208/fido-glossary-v1.0-ps-20141208.html">fido-glossary-v1.0-ps-20141208.html</a> <br>PDF: <a href="https://fidoalliance.org/specs/fido-uaf-v1.0-ps-20141208/fido-glossary-v1.0-ps-20141208.pdf">fido-glossary-v1.0-ps-20141208.pdf</a>
</dd><dt id="bib-ITU-X690-2008">[ITU-X690-2008]</dt><dd rel="dcterms:requires"><a href="http://www.itu.int/rec/T-REC-X.690-200811-I/en"><cite>X.690: Information technology - ASN.1 encoding rules: Specification of Basic Encoding Rules (BER), Canonical Encoding Rules (CER) and Distinguished Encoding Rules (DER), (T-REC-X.690-200811)</cite></a>. International Telecommunications Union, November 2008 URL: <a href="http://www.itu.int/rec/T-REC-X.690-200811-I/en">http://www.itu.int/rec/T-REC-X.690-200811-I/en</a>
</dd><dt id="bib-SP800-90b">[SP800-90b]</dt><dd rel="dcterms:requires">Elaine Baker and John Kelsey, <a href="http://csrc.nist.gov/publications/drafts/800-90/draft-sp800-90b.pdf"><cite>NIST Special Publication 800-90b: Recommendation for the Entropy Sources Used for Random Bit Generation</cite></a>. National Institute of Standards and Technology, August 2012, URL: <a href="http://csrc.nist.gov/publications/drafts/800-90/draft-sp800-90b.pdf">http://csrc.nist.gov/publications/drafts/800-90/draft-sp800-90b.pdf</a>
</dd><dt id="bib-UAFAuthnrMetadata">[UAFAuthnrMetadata]</dt><dd rel="dcterms:requires">B. Hill, D. Baghdasaryan, J. Kemp, <cite>FIDO UAF Authenticator Metadata Statements v1.0</cite>. FIDO Alliance Proposed Standard. URLs: <br>HTML:  <a href="https://fidoalliance.org/specs/fido-uaf-v1.0-ps-20141208/fido-uaf-authnr-metadata-v1.0-ps-20141208.html">fido-uaf-authnr-metadata-v1.0-ps-20141208.html</a><br>PDF: <a href="https://fidoalliance.org/specs/fido-uaf-v1.0-ps-20141208/fido-uaf-authnr-metadata-v1.0-ps-20141208.pdf">fido-uaf-authnr-metadata-v1.0-ps-20141208.pdf</a>
</dd><dt id="bib-UAFProtocol">[UAFProtocol]</dt><dd rel="dcterms:requires">R. Lindemann, D. Baghdasaryan, E. Tiffany, D. Balfanz, B. Hill, J. Hodges, <cite>FIDO UAF Protocol Specification v1.0</cite>. FIDO Alliance Proposed Standard. URLs: <br>HTML: <a href="https://fidoalliance.org/specs/fido-uaf-v1.0-ps-20141208/fido-uaf-protocol-v1.0-ps-20141208.html">fido-uaf-protocol-v1.0-ps-20141208.html</a> <br>PDF: <a href="https://fidoalliance.org/specs/fido-uaf-v1.0-ps-20141208/fido-uaf-protocol-v1.0-ps-20141208.pdf">fido-uaf-protocol-v1.0-ps-20141208.pdf</a>
</dd><dt id="bib-UAFRegistry">[UAFRegistry]</dt><dd rel="dcterms:requires">R. Lindemann, D. Baghdasaryan, B. Hill, <cite>FIDO UAF Registry of Predefined Values</cite>. FIDO Alliance Proposed Standard. URLs: <br>HTML: <a href="https://fidoalliance.org/specs/fido-uaf-v1.0-ps-20141208/fido-uaf-reg-v1.0-ps-20141208.html">fido-uaf-reg-v1.0-ps-20141208.html</a> <br>PDF: <a href="https://fidoalliance.org/specs/fido-uaf-v1.0-ps-20141208/fido-uaf-reg-v1.0-ps-20141208.pdf">fido-uaf-reg-v1.0-ps-20141208.pdf</a>
</dd></dl></section>

## C.2 非规范参考

<section id="informative-references" typeof="bibo:Chapter" resource="#ref" rel="bibo:Chapter"><dl class="bibliography" about=""><dt id="bib-CLICKJACKING">[CLICKJACKING]</dt><dd rel="dcterms:references">D. Lin-Shung Huang, C. Jackson, A. Moshchuk, H. Wang, S. Schlechter <a href="https://www.usenix.org/system/files/conference/usenixsecurity12/sec12-final39.pdf"><cite>Clickjacking: Attacks and Defenses</cite></a>. USENIX, July 2012, URL: <a href="https://www.usenix.org/system/files/conference/usenixsecurity12/sec12-final39.pdf">https://www.usenix.org/system/files/conference/usenixsecurity12/sec12-final39.pdf</a>
</dd><dt id="bib-CommonCriteria">[CommonCriteria]</dt><dd rel="dcterms:references"><a href="http://www.commoncriteriaportal.org/cc/"><cite>CommonCriteria Publications</cite></a>. CCRA Members, Work in progress, accessed March 2014. URL: <a href="http://www.commoncriteriaportal.org/cc/">http://www.commoncriteriaportal.org/cc/</a>
</dd><dt id="bib-FIDOSecRef">[FIDOSecRef]</dt><dd rel="dcterms:references">R. Lindemann, D. Baghdasaryan, B. Hill, <cite>FIDO Security Reference</cite>. FIDO Alliance Proposed Standard. URLs: <br>HTML: <a href="https://fidoalliance.org/specs/fido-uaf-v1.0-ps-20141208/fido-security-ref-v1.0-ps-20141208.html">fido-security-ref-v1.0-ps-20141208.html</a><br>PDF: <a href="https://fidoalliance.org/specs/fido-uaf-v1.0-ps-20141208/fido-security-ref-v1.0-ps-20141208.pdf">fido-security-ref-v1.0-ps-20141208.pdf</a>
</dd><dt id="bib-FIPS140-2">[FIPS140-2]</dt><dd rel="dcterms:references"><a href="http://csrc.nist.gov/publications/fips/fips140-2/fips1402.pdf"><cite>FIPS PUB 140-2: Security Requirements for Cryptographic Modules</cite></a>. National Institute of Standards and Technology, May 2001, URL: <a href="http://csrc.nist.gov/publications/fips/fips140-2/fips1402.pdf">http://csrc.nist.gov/publications/fips/fips140-2/fips1402.pdf</a>
</dd><dt id="bib-NSTCBiometrics">[NSTCBiometrics]</dt><dd rel="dcterms:references">NSTC Subcommittee on Biometrics, <a href="http://biometrics.gov/Documents/Glossary.pdf"><cite>Biometrics Glossary</cite></a>. National Science and Technology Council. 14 September 2006, URL: <a href="http://biometrics.gov/Documents/Glossary.pdf">http://biometrics.gov/Documents/Glossary.pdf</a>
</dd><dt id="bib-RFC2119">[RFC2119]</dt><dd rel="dcterms:references">S. Bradner. <a href="https://tools.ietf.org/html/rfc2119"><cite>Key words for use in RFCs to Indicate Requirement Levels</cite></a>. March 1997. Best Current Practice. URL: <a href="https://tools.ietf.org/html/rfc2119">https://tools.ietf.org/html/rfc2119</a>
</dd><dt id="bib-RFC4086">[RFC4086]</dt><dd rel="dcterms:references">D. Eastlake 3rd, J. Schiller, S. Crocker <a href="http://www.ietf.org/rfc/rfc4086.txt"><cite>Randomness Requirements for Security (RFC 4086)</cite></a>, IETF, June 2005, URL: <a href="http://www.ietf.org/rfc/rfc4086.txt">http://www.ietf.org/rfc/rfc4086.txt</a>
</dd><dt id="bib-SP800-38F">[SP800-38F]</dt><dd rel="dcterms:references">M. Dworkin, <a href="http://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-38F.pdf"><cite>NIST Special Publication 800-38F: Recommendation for Block Cipher Modes of Operation: Methods for Key Wrapping</cite></a>. National Institute of Standards and Technology, December 2012, URL: <a href="http://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-38F.pdf">http://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-38F.pdf</a>
</dd><dt id="bib-SP800-57">[SP800-57]</dt><dd rel="dcterms:references"><a href="http://csrc.nist.gov/publications/nistpubs/800-57/sp800-57_part1_rev3_general.pdf"><cite>Recommendation for Key Management – Part 1: General (Revision 3)</cite></a>. SP800-57. July 2012. U.S. Department of Commerce/National Institute of Standards and Technology. URL: <a href="http://csrc.nist.gov/publications/nistpubs/800-57/sp800-57_part1_rev3_general.pdf">http://csrc.nist.gov/publications/nistpubs/800-57/sp800-57_part1_rev3_general.pdf</a>
</dd><dt id="bib-SecureElement">[SecureElement]</dt><dd rel="dcterms:references"><a href="https://www.globalplatform.org/specifications.asp"><cite>GlobalPlatform Card Specifications</cite></a> GlobalPlatform. Accessed March 2014. URL: <a href="https://www.globalplatform.org/specifications.asp">https://www.globalplatform.org/specifications.asp</a>
</dd><dt id="bib-TEE">[TEE]</dt><dd rel="dcterms:references"><a href="https://www.globalplatform.org/specifications.asp"><cite>GlobalPlatform Trusted Execution Environment Specifications</cite></a> GlobalPlatform. Accessed March 2014. URL: <a href="https://www.globalplatform.org/specifications.asp">https://www.globalplatform.org/specifications.asp</a>
</dd><dt id="bib-TEESecureDisplay">[TEESecureDisplay]</dt><dd rel="dcterms:references"><a href="https://www.globalplatform.org/specifications.asp"><cite>GlobalPlatform Trusted User Interface API Specifications</cite></a> GlobalPlatform. Accessed March 2014. URL: <a href="https://www.globalplatform.org/specifications.asp">https://www.globalplatform.org/specifications.asp</a>
</dd><dt id="bib-TPM">[TPM]</dt><dd rel="dcterms:references"><a href="http://www.trustedcomputinggroup.org/resources/tpm_main_specification"><cite>TPM Main Specification</cite></a> Trusted Computing Group. Accessed March 2014. URL: <a href="http://www.trustedcomputinggroup.org/resources/tpm_main_specification">http://www.trustedcomputinggroup.org/resources/tpm_main_specification</a>
</dd></dl></section>

版权所有，转载请注明出处：

[https://github.com/SickWorm/FIDO-UAF-Chinese-Document](https://github.com/SickWorm/FIDO-UAF-Chinese-Document)
