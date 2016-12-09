本文将介绍FIDO UAF各个文档的内容：

### fido-appid-and-facets
介绍了appID和facetID的命名规则和作用。
appID是app（可能是手机软件，也可能是网页）调用UAF Client时提供的标识名，而facetID是app中更具体的标识。

对于手机软件（Android和iOS），appID和facetID是一致的，Android的facetID必须是apk的签名信息，iOS的facetID必须app的bundle-id。

对于网页（可能是电脑，手机上的网页，或app内嵌的网页），FacetID就是一个URL，而appID必须和FacetID有相同的公共的一段域名（其中端口和子路径将被忽略），具体文档有详细描述。

对于appID和facetID如何防伪，好像是靠系统本身机制（不确定）。比如Android是Client自己获取的facetID，Web好像是想把UAF直接做成W3C标准，让浏览器实现。由UAF Client主动获取的方式来实现，就不用担心伪造了。

### fido-glossary
文档涉及到的一些技术词汇的简单介绍。

### fido-security-ref
FIDO UAF实现需要面对的一些攻击类型以及防御指南。

### fido-uaf-asm-api
主要介绍**Client层和ASM层之间**的通讯协议和ASM层的消息处理流程。

### fido-uaf-authnr-cmds
主要介绍**ASM层和Authenticator层之间**的通讯协议和Authenticator层的消息处理流程。

### fido-uaf-authnr-metadata
定义Server端存储Authenticator Metadata的数据格式。

### fido-uaf-client-api-transport
主要介绍**app如何发现和调用UAF Client**。

### fido-uaf-metadata-service
主要介绍如何构建存储Authenticator Metadata的Service以及其对外接口。

### fido-uaf-overview
FIDO UAF协议的概述。

### fido-uaf-protocol
主要介绍**UAF Server和UAF Client之间的通信协议**与各自的处理流程。
UAF Server和UAF Client中间通过app作为中介，app先向UAF Server发送操作请求，获得请求信息；然后发给UAF Client，并接收回复；再把回复发送给UAF Server，获取操作结果。

### fido-uaf-reg
主要介绍规范中定义的常量，包括它们的意义和值。

> 版权所有，转载请注明出处： 
> https://github.com/SickWorm/FIDO-UAF-Chinese-Document
