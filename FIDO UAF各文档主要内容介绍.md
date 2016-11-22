本文将介绍FIDO UAF各个文档的内容：

### fido-appid-and-facets-v1.0-ps-20141208.html
介绍了appID和facetID的命名规则和作用。appID是app（可能是手机软件，也可能是网页）调用UAF Client时提供的唯一标识名，而facetID是app中更具体的标识。

对于手机软件（Android和iOS），appID和facetID是一致的，Android的facetID必须是apk的签名信息，iOS的facetID必须app的bundle-id。

对于网页（可能是电脑，手机上的网页，或app内嵌的网页），FacetID就是一个URL，而appID必须和FacetID有相同的公共的一段域名（其中端口和子路径将被忽略），具体文档有详细描述。

对于appID和facetID如何防伪，好像是靠系统本身机制（不确定）。比如Android是Client自己获取的facetID，Web好像是想把UAF直接做成W3C标准，让浏览器实现。由UAF Client主动获取的方式来实现，就不用担心伪造了。