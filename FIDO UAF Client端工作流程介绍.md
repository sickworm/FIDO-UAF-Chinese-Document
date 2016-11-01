*本文将介绍FIDO UAF的运作流程。*

根据FIDO UAF文档介绍，FIDO UAF在移动设备上的实现将分为三层：Client，ASM，Authenticator。
![](http://sickworm.com/wp-content/uploads/2016/11/fido-uaf-architecture.png)

当用户在App上进行操作时，App会先向Server发出请求，然后将Server返回的数据转发给Client。最后再将Client返回的结果转发给Server。Server最后回复操作结果。具体如下：

首先，App向FIDO Server发送`GetUAFRequest`，Server会返回`ReturnUAFRequest`，里面包含了与Client交互的数据。然后，App调用UAF Client并传入Server的数据执行注册，签名，注销灯操作。UAF Client收到请求后，选择合适的ASM下发，ASM选择合适的UAF Authenticator下发。而UAF Authenticator可能内部拥有多个内部Authenticator（通过authenticatorIndex区分），UAF Authenticator也会选择对应的内部Authenticator来处理这条指令。指令处理完毕后将原路返回。App收到UAF Client的回复后向FIDO Server发送`SendUAFResponse`，并收到FIDO Server的`ServerResponse`，里面包含了操作的结果。

### 对于Android：
多个UAF Client可能同时存在，通过指定Intent寻找（UAF Client以Activity或Service的形式提供服务，这是官方给出的2种方案）。最终App只会选择一个UAF Client进行通讯。可以使用PackageName来辨别UAF Client。
多个ASM可能同时存在，通过指定Intent寻找（同上）。
ASM应该了解（或者说能找到）自己管理的Authenticator。对于ASM如何发现Authenticator，官方没有提供具体方案，你可以选择用代码耦合；或使用socket连接方式，但需考虑身份检测，数据校验，和AuthenticatorIndex分配等问题。如果是使用外部硬件作为UAF Authenticator，则需要编写对应的设备驱动。

多个UAF Authenticator可能同时存在，官方没有给发现方案（Authenticator连接方式有多种，BT，NETWORK，WIFI等，通过FIDORegistry.ATTACHMENT_HINT_*指定）。官方允许Authenticator和ASM之间用代码紧耦合（不实现Authenticator文档中的指令规范），但如果你要和其它厂商的Authenticator或ASM接入的话，就必须实现文档规范。
Authenticator可能存在很多个Internal Authenticator，比如指纹，脸部识别，安全硬件等验证模块。ASM通过authenticatorIndex和指定内部Authenticator交互。内部Authenticator之间可能有一定联系，比如iPhone指纹识别出错次数过多，就只能用Pin解锁这样的关系。目前没有实现这个机制，文档中也只是提及，没有具体的规范指引。

UAF Authenticator可分为4类，first-bound Authenticator， second-bound Authenticator，first-bound Authenticator，second-broaming Authenticator，。first和second区别在于是否有用户名这个概念。 bound和roaming区别在于是否可以用在多个设备中。它们的组合会影响key handle的存储位置。具体请看：[FIDO UAF中4种Authenticators的区别](http://sickworm.com/?p=216)


> 版权所有，转载请注明出处：
> [https://github.com/SickWorm/FIDO-UAF-Chinese-Document](https://github.com/SickWorm/FIDO-UAF-Chinese-Document)
