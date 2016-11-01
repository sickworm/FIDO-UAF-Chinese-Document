在FIDO UAF中一共有4种Authenticators（认证设备）：
* first-factor bound authenticator（第一因素绑定认证设备）
* second-factor bound authenticator（第二因素绑定认证设备）
* first-factor roaming authenticator（第一因素漫游认证设备）
* second-factor roaming authenticator（第二因素漫游认证设备）

其中，first-factor和second-factor的区别在于，first-factor authenticator会根据用户名存储认证密钥，而second-factor authenticator不会。因此对于同一个app，second-factor所使用的认证密钥是同一个（因为无法区分用户）。

而bound和roaming的区别在于，bound authenticator是与ASM绑定的。而roaming authenticator不与ASM绑定。因此，bound authenticator的密钥会加密存储在ASM中，而不会存储在自己身上。而roaming authenticator的密钥不会存储在ASM中。也就是说，如果你通过另一个ASM去访问到这个bound authenticator，你无法得到原来的密钥。

特殊的是，对于不存储密钥在ASM中的roaming authenticator，first-factor roaming authenticator会把密钥存储在自己身上，而second-factor roaming authenticator会把密钥存储在FIDO Server。至于为什么这样设计，我也没有具有说服力的解释。但从结果来看，second-factor roaming authenticator不需要额外的存储，只需要存储一个不变的用于加解密的密钥（对），这使得其变得简单廉价。

注意：**4种Authenticators的分类并不是规范，在厂商不需要和其他厂商的FIDO UAF软硬件联合使用时，厂商可以自行选择Authenticator的实现方法。但如果你需要用到别人的ASM，那你还是要按照该文档的内容进行实现。**

相关的官方**节选**文档如下：（译文）

> # 7. KeyIDs与key handles

> *该章节不是规范。*

> 该文档根据其在处理command时的特性和行为差异一共定义了4种Authenticator。其中一个主要的差别是它们存储和处理key handles的差异。该章节试图描述所有Authenticator在处理相关command的时候的行为。

> ## 7.1 first-factor Bound Authenticator
> **注册时：**

> Authenticator不存储key handles。KeyHandles由ASM存储并提供。
> KeyID为随机生成的32位随机数（或简单的就做成KeyHandle的hash值）。

> **注销时：**

> 因为Authenticator不存储任何key handles，所以Authenticator没什么需要删除的。
> **ASM**找到KeyID对应的KeyHandle然后删除。

> ## 7.2 2ndF Bound Authenticator
> **注册时：**

> Authenticator不存储key handles。KeyHandles由ASM存储并提供。
> KeyID为随机生成的32位随机数（或简单的就做成KeyHandle的hash值）。

> **注销时：**
> 因为Authenticator不存储任何key handles，所以Authenticator没什么需要删除的。

> **ASM**找到KeyID对应的KeyHandle然后删除。


> ## 7.3 first-factor Roaming Authenticator
> **注册时：**

> Authenticator存储key handles到自己的内部存储空间。ASM不会发给Authenticator。

> KeyID为随机生成的32位随机数（或简单的就做成KeyHandle的hash值）。

> **注销时：**

> **Authenticator**找到KeyID对应的KeyHandle然后删除。



> ## 7.4 2ndF Roaming Authenticator
> **注册时：**

> Authenticator和ASM都不存储key handles。KeyHandles（将取代KeyID）将发送给Server，由Server存储到用户记录中。从Server的角度来说，KeyHandles被当作KeyID。实际上KeyID就是KeyHandles。

> **注销时：**

> 因为Authenticator和ASM不存储任何key handles，所以client端没什么需要删除的。

——————————————————————————————————————————————————————————————————

> 版权所有，转载请注明出处：
> [https://github.com/SickWorm/FIDO-UAF-Chinese-Document/](https://github.com/SickWorm/FIDO-UAF-Chinese-Document/)
