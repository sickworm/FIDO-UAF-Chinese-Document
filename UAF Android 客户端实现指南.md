# 1. 文档简介

本文档介绍了我们的 FIDO UAF 客户端的全套实现方案，包括 app，client application，UAF Client，UAF ASM，和指纹类型的 UAF Authenticator。文档旨在帮助你理解 UAF 的实现原理，且阅读时尽可能不需要你去额外从官方文档中获取信息。但我依然强烈建议你先阅读一遍官方规范，因为许多细节本文档没有办法全部覆盖、在正式的 FIDO UAF 测试中，你需要使用到这些信息，与其它实现厂商进行讨论。

本文档由我单独编写，不可避免会有疏漏错误之处。如果你遇到有疑问和认为有误的地方请务必与我交流。

本文档适用于 UAF 1.0 规范。1.1 规范其实功能上基本没有改动，所以也可以用作规范 1.1 的参考，但我没有把区别找出来。

由于时间不允许，本文档后半部分未能详细描述，接下来我会 review 并慢慢补充。

# 2. FIDO 简介
本章将从用户使用角度出发介绍 FIDO 以及其规范 UAF 和 U2F。

官网上描述： FIDO，世界上最大的认证生态系统，它基于标准的可交互的认证。FIDO 旨在提供更安全更简单的网上认证方式。FIDO 目前有两个规范，分别是 UAF 和 U2F。

UAF （ Universal Authentication Framework ），旨在提供免密认证的规范协议。用户可通过指纹，虹膜，声音，脸部等生物识别技术来完成验证。UAF 也支持 PIN 码输入的验证方法。只要你实现了 UAF 的规范，UAF 支持的验证方法是非常多的，甚至支持蓝牙，NFC 外接验证设备。但主要 UAF 还是尽可能利用设备已有的模块来完成验证。

U2F （ Universal Second Factor ），旨在提供一种硬件设备来代替现存的密码验证（和 U 盾那样的功能）。 你只需要一个 U 盾这样的 U2F USB key，在支持 U2F 的网站上登陆时插入，即可代替复杂的密码输入（当然 U2F 设备需要先和账户绑定）。网站可以信任该设备允许直接登陆，或者只需要再要求用户输入一个比如只有4位的简单密码进行验证。

总的来说 UAF 是个更宽泛，更复杂的框架，并且 UAF 也是支持 second factor 设备的。而 U2F 脱离于 UAF，更简单且专注于持有外部设备这样的验证方式。这样使得 U2F 的商业化更为容易，无论从易用性还是实现难易的角度来看。U2F 可以直接由浏览器和网页支持，但 UAF 必须在本地设备有相关的软件组件。

# 3. FIDO UAF 简介
本章将介绍 UAF 的协议框架。

## 3.1 FIDO UAF 功能简介

 FIDO UAF 提供了用户注册，登陆认证，交易认证，注销等功能。登陆认证和交易认证需要用户先进行注册。

### 3.1.1 reg 注册
注册需要提供 username。first factor authenticator 会为注册的 username 生成一对非对称密钥对（或所有 username 都使用同一张预植的证书），second factor authenticator 不会识别 username，所以所有使用同一个 second factor authenticator 的 user 会共享同一对非对称密钥对（或预植证书）。FIDO UAF 要求密钥对存储在 UAF Authenticator 或 UAF ASM 的安全环境中。

注册的实质就是 authenticator 为用户生成一对非对称密钥对（或使用已有预植证书），在登陆认证和交易认证的时候使用私钥进行签名，服务器再通过 authenticator 提供的公钥对签名报文进行验签，验签成功则认可这个用户，从而实现免密登陆的特性。authenticator 使用私钥签名之前需要用户先通过指纹，虹膜或其他的验证方式（由该 authenticator 的特性决定）。也就是说，用户以指纹，虹膜或其他的验证方式来代替了普通密码登陆。

注册得到的非对称密钥会和 app，以及 UAF Client 绑定，加密存储在 UAF ASM 或 UAF Authenticator （根据 authenticator 特性）中。

### 3.1.2 auth 登陆认证
在某个 authenticator 注册后，再之后的登陆认证则可以使用该 authenticator 进行认证操作。auth 分为带 username 登陆和不带 username 登陆（更严格来说 username 属于一种 session ），即按照 auth 是否知道当前需要认证的用户是谁可分为两种情况。

* auth with username ：此时 UAF Server 会查找该 username 已注册的密钥对，并返回其已注册的 keyIDs （可以密钥对相关数据结构的简单 hash，由 authenticator 在注册时提供）。UAF Client 只可以使用 UAF Server 提供的 keyIDs 中其中一个进行签名（通常来说只会有1个）
* auth without username ：因为没有相关用户的信息，所以此时 UAF Server 不会返回任何 keyIDs。对于这种情况，UAF ASM + first factor authenticator 会把该 app 通过该 UAF Client 注册的所有 keyIDs 拿出来，并找到其对应的 username，然后弹出 username 列表给用户选择：你想登陆哪个用户？。选择后再用该 username 对应的私钥进行签名并通过 app 返回给 UAF Server。而 second factor authenticator 是无法处理的，因为他没有存储 username 信息，无法选择特定的用户登陆。

### 3.1.3 transaction 交易认证
交易认证比登陆认多了一个交易信息，交易信息需要显示给用户（如果支持）并加入签名报文，其他和登陆认证是一样的。所以交易认证的操作类型也是 auth （ UAF 只有 reg，auth，dereg 三种操作）。交易信息可以是 text，也可以是 image 或其他，具体看 authenticator 的特性。

根据业务流程，交易认证肯定需要先进行登陆，所以 username 是必须要知道的，也就不存在 auth without username 的情况。

### 3.1.4 dereg 注销
已注册的用户可以通过注销删除用户记录。特殊的是注销操作不需要把 UAF Client 的结果发送给 UAF Server。UAF Server 在接收到注销操作后，就直接删除用户了。也就是说 UAF Server 不管你 UAF Client 结果怎样，我都会删除（规范也保证了这样的设计是可靠的，就算 UAF Client 注销失败，也有其他办法修正）。app 再把 UAF Server 返回的数据发送给 UAF Client 执行注销操作。

## 3.2 FIDO UAF 流程简介
根据 FIDO UAF 文档指引，FIDO UAF 在移动设备上的实现将分为三层： UAF Client，UAF ASM，UAF Authenticator。而 UAF Client 是不具备与 UAF Server 通信的能力的，UAF Client 和 UAF Server 之间还需要一个中介。所以我们最终实现一共有5层： app， client application，UAF Client，UAF ASM，UAF Authenticator。各层的简要功能如下：

* app ：提供 UI 用于演示功能，显示 UAF Server 和 UAF Client 的信息交互，提供简单的 Server 设置。
* client application ：封装与 UAF Server 交互的 API，以及与 UAF Client 交互的 API。app 调用 client application 的 API 与 UAF Server 及 UAF Client 交互。
* UAF Client ：负责处理 Server 的 protocol message，并根据内容向 UAF ASM 发出请求。
* UAF ASM ：用于处理 UAF Client 层的信息，并管理 UAF Authenticator。实际情况中 UAF ASM 通常与 UAF Authenticator 看做一个整体，对外打包为1个 apk。
* UAF Authenticator ：用于认证用户，一个 UAF Authenticator 可能存在多个内部的 authenticator，它们拥有不同的特性，提供不同的认证方式。

![交互流程](//)

1. app 通过 client application 向 UAF Server 发送 GetUAFRequest 请求，里面包含了你想要做的 reg 操作，reg 注册，auth 签名，dereg 注销三者之一。如果请求合法，Server 会返回 ReturnUAFRequest 给 app;
2. app 把 ReturnUAFRequest 中的 uafRequest 发送给 UAF Client，进行注册，签名，注销等操作（与 GetUAFRequest 的操作请求一致）。
3. UAF Client 在处理这条信息的过程中，会与 UAF ASM 发生数次通信，如查找可用的 authenticator，获取已注册用户的信息，认证等。处理完毕后整理结果返回给 app；
4. 在 UAF ASM 收到来自 UAF Client 的请求时，UAF ASM 也会与 UAF Authenticator 进行通信，以收集结果返回给 UAF Client；
5. UAF Authenticator 内部会有1个到多个的 internal authenticator，它们通过 authenticatorIndex 分辨。UAF Authenticator 根据 UAF ASM 的具体指令找到特定的 internal authenticator 并交互得到结果；
6. App 收到回复 UAF Client 的回复后，通过 client application 向 FIDO Server 发送 SendUAFResponse 请求，并收到 FIDO Server 的 ServerResponse，里面包含了操作的结果。

注：

1. 在一个设备中，多个 UAF Client 可能同时存在。对于 Android 设备，将通过特定的 Intent 寻找，UAF Client 将通过 Activity 或 Service 组件，通过 Intent 通信来提供服务。实际情况中，Service 作为可选方案，且有一定的劣势（如生命周期，枚举限制等），基本没有厂商会采用 Service 方案。我们的 Android 移动端两种都实现了。对于 iOS 设备，app 通过 canOpenURL()来寻找 UAF Client。

2. 在 Android 设备中，多个 UAF ASM 可能同时存在。UAF Client 会枚举所有的 UAF ASM 出来并收集各个 ASM 所管理的 UAF Authenticator 的信息，并让用户选择将要进行认证的 authenticator。在 iOS 设备中，一个 Client 只会对应一个 ASM。

3. 一般来说 ASM 与 Authenticator 是绑定关系，通过代码耦合即可。文末我提供了一种让其他 UAF Authenticator 接入 ASM 的方案，用于在未知对方 UAF Authenticator 的情况下接入到我们的 UAF ASM。但恐怕并没有这样的使用场景，因为 UAF Client 在枚举 UAF ASM 的时候已经变相的枚举了 UAF Authenticator，对于多个 UAF Authenticator，只需要给他们分别配上单独的 UAF ASM 即可。

4. 一个 UAF ASM 可能会管理多个 UAF Authenticator。

5. UAF Authenticator 可能存在很多个 internal authenticator，比如存在指纹，脸部识别，TEE 等使用了不同验证模块实现的 authenticator，甚至是否支持用户名存储，密钥存储的位置在哪也会有不同的 authenticator。internal authenticator 之间可能有一定联系，比如 iPhone 指纹识别出错次数过多，就只能用 Pin 解锁这样的关系。目前我们没有实现这个机制，文档中也只是提及这样的情况，并没有具体的规范指引。

## 3.3 UAF Android/iOS 的打包方式
上面提到，我们的客户端分为了5层 app，client application，UAF Client，UAF ASM，UAF Authenticator。

### 3.3.1 Android 
app，client application 该两层属于同一个 apk/ipk。功能上 app + client application 可用 FIDO 官方的 conformance.apk 代替。

UAF Client 可以打包为一个独立的 apk。

UAF ASM，UAF Authenticator 在大部分情况下打包为一个独立的 apk。UAF ASM 一般只管理自己的 UAF Authenticator，很少有其他的 UAF Authenticator 需要接入到你的 UAF ASM 的情况。

当然你全部打包成一个 apk 也是可以的，特别是以库的形式提供给客户的时候。他们很可能并不想用到其他 App 的 UAF Client 和 UAF ASM，此时你需要写死你的 client application 和 UAF Client 只会搜寻自己的 UAF Client 和 UAF ASM。

### 3.3.2 iOS 
app，client application 该两层属于同一个 apk/ipk。功能上 app + client application 可用 FIDO 官方的 conformance.ipk 代替。

UAF Client，UAF ASM，UAF Authenticator 将打包为一个独立的 apk。因为 FIDO UAF 没有定义 iOS 下 UAF Client 与 UAF ASM 通信的方式，因为 iOS 较为封闭的 App 间通信方式，导致了 UAF Client 无法枚举 iOS 设备上所有存在的 UAF ASM。

# 4. FIDO UAF Android 客户端实现方案
本章将详细介绍上文提到的5个模块（ app，client application，UAF Client，UAF ASM，UAF Authenticator ）的实现方案。

## 4.1 app 
app 实现了如下功能：

1. 提供功能测试按钮： reg 注册，auth 认证，transaction 带交易信息的认证，login 无用户名认证，dereg 注销，discovery 发现可用的 authenticator。没有实现 check policy 功能
2. 显示与 UAF Server 和 UAF Client 之间通信的结果
3. 显示该 app 的 facet-id 
4. 可修改 UAF Server URL 

这几个功能点的 UI 都没有实现难度。


## 4.2 client application 
client application 主要功能就封装与 UAF Server 和 UAF Client 通信的接口。本章节将介绍与 UAF Server 和 UAF Client 的实现方案。

client application 与 UAF Server 通信的数据皆为 UTF-8编码的 json 数据，并使用 gson 转换为对象。

本章节对应文档为 fido-uaf-client-api-transport-v1.0-ps-20141208。

### 4.2.1 UAF Server 通信实现
文档[fido-uaf-client-api-transport](https:// fidoalliance.org/specs/fido-uaf-v1.1-id-20170202/ fido-uaf-client-api-transport-v1.1-id-20170202.html)和 Guildelines to create Adapter （文档找不到了）定义了 UAF Server 通信 API。

#### 4.2.1.1 网络库选择
网络库选择 Retrofit，优点是功能强大，支持 ORM，RxJava 等特性；缺点是体积大，封装程度高，有时遇到其他厂商的 UAF Server 数据不对的时候，ORM 转换失败，不好定位问题。由于用于制作 demo，所以网络库的选择并没有太大的考究。

#### 4.2.1.2 Retrofit 对象构建
代码示例：

```
Retrofit retrofit = new Retrofit.Builder()
       .baseUrl(url)
       .addConverterFactory(ScalarsConverterFactory.create())
       .addConverterFactory(GsonConverterFactory.create(new Gson()))
       .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
       .build();
ServerAPI serverAPI = retrofit.create(ServerAPI.class);
```

* `ScalarsConverterFactory`为支持解析返回数据为 String，调试用，非必需

* `GsonConverterFactory`为支持返回数据为任意对象，通过 Gson 进行 ORM 转换。

* `RxJavaCallAdapterFactory`为支持返回类型为 RxJava 的接口 Call<>。

ServerAPI.java:

```
public interface ServerAPI {
    @POST("Get")
    Call<ReturnUAFRequest> getUAFRequest(@Body GetUAFRequest request);

    @POST("Send/Reg")
    Call<ServerResponse> sendUAFRegResponse(@Body SendUAFResponse request);

    @POST("Send/Auth")
    Call<ServerResponse> sendUAFAuthResponse(@Body SendUAFResponse request);

    @POST("Send/Dereg")
    Call<ServerResponse> sendUAFDeregisterResponse(@Body SendUAFResponse request);

    @GET("Settings")
    Call<String> getSettings();
}
```

`ServerAPI.java`定义了 UAF Server 的测试接口。仅用作通过 FIDO 测试时使用，实际产品不必按照这个要求。参考：[https://conformance.fidoalliance.org/v2_tool/help_doc/Guidelines%20to%20create%20Adapter.pdf](https://conformance.fidoalliance.org/v2_tool/help_doc/Guidelines%20to%20create%20Adapter.pdf)

#### 4.2.1.3 使用 Retrofit 发起 GetUAFRequest 请求
最后发送请求时使用`serverAPI`对象调用相关接口，得到 Call 对象，再把其加入队列：

```
Callback<ReturnUAFRequest> callback = new Call<ReturnUAFRequest>(){
      @Override 
      public void onResponse(Call<ReturnUAFRequest> call , final Response<ReturnUAFRequest> response){
          if(response.body()== null){
              Log.w("TAG","incorrect ReturnUAFRequest format , code"+ response.code());
              return;
          }
          Log.w("ReturnUAFRequest:\n"+ new Gson().toJson(response.body()));
          if(response.body().uafRequest == null){
              Log.w("TAG","uafRequest null");
              return;
          }
          
          // send the message to UAF Client 
      }
      @Override 
      public void onFailure(Call<ReturnUAFRequest> call , Throwable t){
          Log.w("TAG","ReturnUAFRequest failed:\n"+ t);
      }
}
Call<ReturnUAFRequest> requestCall = serverAPI.getUAFRequest(request);
requestCall.enqueue(callback);
```


### 4.2.2 UAF Client 通信实现
文档[fido-uaf-client-api-transport](https:// fidoalliance.org/specs/fido-uaf-v1.1-id-20170202/ fido-uaf-client-api-transport-v1.1-id-20170202.html)定义了 UAF Client 通信 API。

#### 4.2.2.1 UAFMessage 
通过与 UAF Server 通信，我们得到了 ReturnUAFRequest 对象。其中 UAF Client 真正用到的是 ReturnUAFRequest.uafRequest。但在发送给 UAF Client 之前，我们还需要先把他包装成 UAFMessage ：

UAFMessage.java ：

```
public class UAFMessage {
    @Required public String uafProtocolMessage;
    public AdditionalData additionalData;
}
```

`uafProtocolMessage`：即 ReturnUAFRequest.uafRequest 

`additionalData`：如果 UAF Client 支持额外的参数，则把额外的数据放在 AdditionalData。

`@Require`： Required 注释，带上 Required 注释的成员不能为空，否则下层会拒绝该 message。下同。

#### 4.2.2.2 UAF Intent 
得到 UAFMessage 后，我们需要把他放入 Intent 中传递给 UAF Client。UAF Client 支持多种操作，每种操作要求的数据不一样。根据`UAFIntentType`，向 UAF Intent 可以分为几种种类：

UAFIntentType.java ：

```
enum UAFIntentType {
   "DISCOVER",
   "DISCOVER_RESULT",
   "CHECK_POLICY",
   "CHECK_POLICY_RESULT",
   "UAF_OPERATION",
   "UAF_OPERATION_RESULT",
   "UAF_OPERATION_COMPLETION_STATUS"
};
```


* `DISCOVER`：查找目前可用的 authenticator。回复为`DISCOVER_RESULT`。
* `CHECK_POLICY`：根据提供的`uafProtocolMessage`，检查是否能匹配上至少一个 authenticator。回复为`CHECK_POLICY_RESULT`。
* `UAF_OPERATION`：常规的 UAF 操作，包括 reg , auth , dereg。回复为`UAF_OPERATION_RESULT`。
* `UAF_OPERATION_COMPLETION_STATUS`：用于 app 把 UAF Client 的操作结果发送给 UAF Server 并得到回复后，把 UAF Server 的回复再发给 UAF Client。例如 UAF Client 完成了一项注册，app 把结果发送给 UAF Server，UAF Server 却拒绝了这个注册结果（可能是验签不过，或者字段缺失等问题）。此时 app 把 UAF Server 的结果发送给 UAF Client，UAF Client 就需要把这个不合格的用户注册给注销掉，因为这个用户注册并没有被 UAF Server 接受，它是无用的。`UAF_OPERATION_COMPLETION_STATUS`只需要发送给 UAF Client，不需要接收回复。

UAF Intent 可能需要的参数，我定义了一个专门的类 UAFIntent 来管理。UAFIntent 可以和 Intent 互相转换（通过 Intent.putExtra()和 Intent.get *** Extra()），以方便进行数据提取。

UAFIntent.java:

```
public class UAFIntent {
    public UAFIntentType uafIntentType; // 操作类型，注意不是 reg，auth，dereg，而是上面提到的操作
    public String discoveryData; // DISCOVER_RESULT 返回使用的字段
    public String componentName; // UAF Client 返回结果时会带上该 UAF Client 的 ComponentName。作用是如果 app 同时与多个 UAF Client 进行通信，则 componentName 可以帮助分辨该结果是来自哪个 UAF Client。
    public Short errorCode; // UAF Client 返回结果时的错误码。详见 fido-uaf-client-api-transport 4.5 ErrorCode 
    public String message; // UAF Server 提供的 uafRequest。仅用于 UAF_OPERATION
    public String origin;  // 如果 UAF Client 是通过 web 浏览器调用的话，orgin 则代表调用的 Web Origin（比如你做的 app 是一个浏览器，且支持 UAF，则网站请求认证的时候，app 把网站的 Web Origin 加入到里面。Web Origin 定义请看 RFC6454）
    public String channelBindings; // 包含 https 的 TLS 数据，用于验证连接是否是安全的。目前没有厂商用到这个，好像原因是服务器限制。但是 channelBindings 不能为 null，你需要给一个空的 channelBindings。
    public Short responseCode; // 仅用于 UAF_OPERATION_COMPLETION_STATUS，报告 UAF Server 的处理结果
}
```

具体 UAF Intent 的参数要求如下：

| UAFIntentType value | discoveryData | componentName | errorCode | message | origin | channelBindings | responseCode 
|---|---|---|---|---|---|---|---|
|"DISCOVER"||||||||
|"DISCOVER_RESULT"| OPTIONAL | REQUIRED | REQUIRED | | | | |
|"CHECK\_POLICY"| | | | REQUIRED | OPTIONAL | | |
|"CHECK\_POLICY\_RESULT"| | REQUIRED | REQUIRED | | | | |
|"UAF\_OPERATION"| | | | REQUIRED | OPTIONAL | REQUIRED | |
|"UAF\_OPERATION\_RESULT"| | REQUIRED | REQUIRED | OPTIONAL | | | |
|"UAF\_OPERATION\_COMPLETION\_STATUS"| | | | REQUIRED | | | REQUIRED |

##### 4.2.2.1.1`DISCOVER`
DISCOVER 请求不需要附带额外参数。

DISCOVER\_RESULT ：

`discoveryData`：是一个 JSon 格式的 String，如下：

DiscoveryData.java ：

```
public class DiscoveryData {
    @Required Version[]       supportedUAFVersions; // UAF Client 支持的协议版本
    @Required DOMString       clientVendor;  // 开发 UAF Client 的厂商
    @Required Version         clientVersion;  // UAF Client 的版本，非协议版本支持
    @Required Authenticator[] availableAuthenticators; // 目前可用的 authenticators 
};
```

`componentName`：调用的 UAF Client 的 ComponentName。

`errorCode`：操作的错误码，如下：

ErrorCode.java ：

```
public class ErrorCode {
    public static final short NO_ERROR = 0x0; // 无错误
    public static final short WAIT_USER_ACTION = 0x1; // 正在等待用户操作
    public static final short INSECURE_TRANSPORT = 0x2; // 不安全的通信，当 ChannelBindings 检查不通过时返回
    public static final short USER_CANCELLED = 0x3; // 用户取消了操作
    public static final short UNSUPPORTED_VERSION = 0x4; // 不支持的 UAF 协议版本
    public static final short NO_SUITABLE_AUTHENTICATOR = 0x5; // 没有找到合适的 Authenticator 
    public static final short PROTOCOL_ERROR = 0x6; // 协议错误，如消息内容不符合协议规范（缺少字段等）
    public static final short UNTRUSTED_FACET_ID = 0x7; // 不信任的 facetID，在 Android 中 facetID 是 app 的签名 hash 
    public static final short UNKNOWN = 0xFF; // 未知错误，多用于 UAF Client 内部产生的错误
}
```

DISCOVER 操作不需要与 UAF Server 交互。

##### 4.2.2.1.2 UAF\_OPERATION 

UAF\_OPERATION ：

`message`：即 UAFMessage 的 json 格式，UAFMessage 内含 UAF Server 提供的`uafRequest`。

`channelBindings`：没有做太多了解，因为其他厂商貌似都不支持这个。如果 UAF Server 不支持，直接传一个空的 ChannelBinding （注意是 empty 不是 null ）就可以了。

`origin`：如果 app 是一个 Web 浏览器，而 UAF 请求是来自网页，则 origin 应该为该网页的 Web Origin，否则不需要该参数。

UAF\_OPERATION\_RESULT ：


`componentName`：调用的 UAF Client 的 ComponentName。

`errorCode`：操作的错误码。

`message`：当`errorCode`为`NO_ERROR`时，`message`为 UAF Client 回复的 UAFMessage，里面的`uafProtocolMessage`需要发送给 UAF Server。当`errorCode`不为`NO_ERROR`时，不会返回`message`。

##### 4.2.2.1.3 CHECK\_POLICY 
CHECK\_POLICY 充当的作用就是，预先让 UAF Client 测试以下这条 UAF Server 的信息是否能够被 UAF Client 接收并找到对应 authenticator 处理。UAF Client 并不会处理这条信息。这个功能可以帮助 app 在多个 UAF Client 中寻找能够处理这条信息的 UAF Client。

CHECK\_POLICY 失败的原因可能是： UAF Client 不支持该信息要求的协议版本，UAF Client 没匹配到该信息要求的指定 authenticator。或者这条信息本身就是不合规范的，也有可能 UAF Client 实现不规范。

CHECK\_POLICY 不需要提供`channelBindings`。

CHECK\_POLICY\_RESULT 不会返回`message`，app 只需要关注 errorCode 的结果即可。

##### 4.2.2.1.4 UAF\_OPERATION\_COMPLETION\_STATUS 
发送 UAF\_OPERATION\_RESULT 的 message 给 UAF Server，收到 UAF Server 的回复后，把 UAF Server 的结果发给 UAF Client。

`message`：发送给 UAF Server 的`uafProtocolMessage`的 UAFMessage。

`errorCode`： UAF Server 返回的结果。

#### 4.2.2.3 与 UAF Client 通信的 Communicator 实现
client application 与 UAF Client 通信是通过 Activity.startActivityForResult()（ Activity ）或 AIDL （ Service ）进行通信。通信的 Intent 需要指定：

```
Intent.setAction("org.fidoalliance.intent.FIDO_OPERATION");
Intent.setAction("application/fido.uaf_client + json");
```

Service 的 AIDL 还好，可以拿到具体的通信对象，但 startActivityForResult()只能在 Activity.onActivityResult()回调。当在一段处理逻辑中，在中间需要打开其他 Activity 进行通信，则需要去 Activity.onActivityResult()中获取结果。那你是在原来的逻辑中加一个锁等待，还是把剩下的逻辑写在 onActivityResult()中？如果这段逻辑会发生数次 Activity.startActivityForResult()，而每次请求的数据不一样呢？那样代码就会变得杂乱。在平常的开发中应尽量避免逻辑依赖 startActivityForResult()结果这样的情况发生，用状态机可能是个更好的方法。

当然如果 Activity.startActivityForResult()可以设置 listener，就不会这么麻烦了。但 Android 并没有这样设计。

本着低耦合，高内聚的原则，我希望 client application 对 UAF Client 的通信有专门的 Communicator 负责，我可以有异步的，同步的通信方法来取得结果，以方便我的逻辑流程处理。我定义了 ClientCommunicator 接口：

```
interface ClientCommunicator {

    void sendRequest(Intent clientRequest , IUAFResponseListener listener);

    boolean init(Context context);

    void release();
}
```

其中 IUAFResponseListener 是 AIDL 中定义的一个简单的回调接口，直接使用方便兼容 Activity 和 Service 方案：

```
interface IUAFResponseListener {

    void onResult(in Intent uafResponse);
}
```

这个接口可以实现逻辑与 Activity 的解耦。

如果使用常规方法，则需要把处理逻辑写进 Activity （因为你需要监控 onActivityResult()），这

##### 4.2.2.3.1 Activity 方案的 Communicator 实现
如何为 UAF Client Actvitiy 实现 ClientCommunicator 接口？我想到了两种思路：

1. 声明 StartActivityListener 接口，交由 app 的 Activity 实现：

```
interface StartActivityListener {
    
    startActivityForResult(Intent intent);
}
```

并且 ClientCommunicator 实现类还需要实现：

``` 
onResult(Intent intent);
```

在需要调用 UAF Client Activity 的时候 startActivityForResult 的时候调用自身的 startActivityOnResult，在得到结果的时候调用 ClientCommunicator 实现类的 onResult。

方案缺点：耦合严重，Communicator 和 Activity 互相调用，且要求 app 提供 Activity 对象。实现起来很难受。

2. ClientCommunicator 启动专门的 Activity 实现与 Activity UAF Client 通信。

创建一个 RequestActivity，在调用 ClientCommunicator.sendRequest()的时候启动 RequestActivity，由 RequestActivity 与 UAF Client Activity 通信。RequestActivity 得到 UAF Client 的结果后，再返回给 ClientCommunicator。

在 sendRequest 中我做了如下的设计：

* 考虑到并发，每次交由 RequestActivity 的请求都有唯一的 requestCode 对应；
* 每一个请求对应的 requestCode 和回调 listener 都会保存在一个 List 中，等待 RequestActivity 返回结果；
* 传递给 RequestActivity 数据的方法是通过把数据放入 startActivity 的 Intent 中实现；
* RequestActivity 返回的方法是通过 ClientCommunicator 实现类的静态方法 notifyOnce(int requestCode , Intent intent)。当 RequestActivity 拿到 UAF Client Activity 的返回 Intent 时，调用 ClientCommunicator 实现类的静态方法 notifyOnce，并视情况结束自己；
* notifyOnce 中会根据 requestCode 在 List 中寻找对应的 listener，把其从 List 中移除，并调用 listener。（先移除会更安全一些，以免 listener 的某些操作影响原有队列）
* RequestActivity 返回结果后，考虑到可能同时有好几个请求进入 RequestActivity，需要增加一个计数器 count，来判断当前是否还有请求没返回。如果已经没有请求了，再执行 finish()结束自己。


```
private Context context; // 上层提供的 context，用于组件通信
private ComponentName componentName; // 通讯对象 UAF Client 的 ComponentName，如果为空则 Android 会弹出 UI 让用户选择使用哪个 UAF Client，不为空则直接使用该 UAF Client。通过 Intent.setComponent()实现
        
@Override 
public void sendRequest(Intent uafRequest , final IUAFResponseListener listener){

    Intent request = new Intent(context , RequestActivity.class);
    request.putExtra(ESDefinitions.EXTRA_INTENT , uafRequest);  // 将需要发送给 UAF Client 的 Intent 发给 RequestActivity，由 RequestActivity 代发
    if(componentName != null){
        request.putExtra(ESDefinitions.EXTRA_COMPONENT_NAME , componentName); // 如果已经有选定的 UAF Client，也把对应的 componentName 发送给 RequestActivity 
    }

    int requestCode = generateRequestCode(); // startActivityForResult 的 requestCode 由 ClientCommunicator 生成，以便识别是哪次请求。为了并发考虑，generateRequestCode()要求生成的 requestCode 唯一。
    request.putExtra(ESDefinitions.EXTRA_REQUEST_ID , requestCode);
    addListener(requestCode , new OnActivityResult(){ // requestCode 和 listener 是一一对应的，把两者加入 listener List 中。这里 OnActivityResult()也只是一个简单的 Intent 结果回调。
        @Override 
        public void onActivityResult(Intent clientResponse){
            if(listener != null){
                try {
                    listener.onResult(clientResponse); // 把结果返回给上层
                } catch(RemoteException e){
                    e.printStackTrace();
                }
            }
        }
    });
    if(context instanceof Service){ // 如果 context 时 Service，需要加上 NEW_TASK，否则会失败
        request.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
    }
    context.startActivity(request);
}
```

在 RequestActivity 中：

```
public class RequestActivity extends AppCompatActivity {
    private int count = 0; // 计数当前正在等待结果的请求数

    @Override 
    protected void onCreate(Bundle savedInstanceState){
        super.onCreate(savedInstanceState);
        setContentView(new View(this)); // 空白的 View 
        processIntent(getIntent());
    }

    @Override 
    protected void onNewIntent(Intent intent){
        // 如果 RequestActivity 正在等待上一次的请求的结果，而 ClientCommunicator 通过 startActivity()又发了一个请求过来，则会进入到 onNewIntent()
        super.onNewIntent(intent);
        processIntent(intent);
    }

    private void processIntent(Intent intent){
        count ++;
        int requestCode = intent.getIntExtra(ESDefinitions.EXTRA_REQUEST_CODE , ESDefinitions.DEFAULT_REQUEST_CODE);
        ComponentName componentName = getIntent().getParcelableExtra(ESDefinitions.EXTRA_COMPONENT_NAME);
        Intent clientIntent = getIntent().getParcelableExtra(ESDefinitions.EXTRA_INTENT);
        if(clientIntent == null){
            ClientOperationHelper.ClientActivityCommunicator.notifyOnce(requestCode , null);
            finish();
            return;
        }

        clientIntent.setAction("org.fidoalliance.intent.FIDO_OPERATION"); // UAF Client 规范
        clientIntent.setType("application/fido.uaf_client + json"); // UAF Client 规范
        if(componentName != null){
            clientIntent.setComponent(componentName);
        }

        try {
            startActivityForResult(clientIntent , requestCode);
        } catch(SecurityException e){
            // 某些 UAF Client 可能会因为权限问题拒绝你的访问，但此时 onActivityResult()依然会被调用，但 Intent data 是 null 
            Log.w("TAG","not permission to access activity"+ componentName);
        }
    }

    @Override 
    protected void onActivityResult(int requestCode , int resultCode , Intent data){
        super.onActivityResult(requestCode , resultCode , data);

        ClientOperationHelper.ClientActivityCommunicator.notifyOnce(resultCode , data);
        count--;
        // 如果当前没有请求了，则 finish 掉自己
        if(count == 0){
            finish();
        }
    }
}
```

由于 RequestActivity 只是一个辅助工具，所以我把他设置为透明的。这样用户便无法察觉该 Activity 的存在。设置方法，在 AndroidManifest.xml 中为 RequestActivity 使用透明主题：

```
<activity 
    android:name=".RequestActivity"
    android:theme="@style/Theme.Transparent">
</ activity>
```

RequestActivity 需要继承 AppCompatActivity，否则不能完全透明。

方案缺点：多启动一个空白的 RequestActivity，影响性能。（实际情况在 MI max 没有明显问题）

最后我选用了方案2，牺牲部分性能保证代码灵活性。

##### 4.2.2.3.2 Service 方案的 Communicator 实现
首先要说明的是，Service 的 UAF Client 方案在 UAF 规范中只是一个替代方案，且在正式测试中没有厂商使用该方案进行测试。不实现 Server 方案是完全可以的。

Service UAF Client 的实现方案比 Activity UAF Client 简单，因为与 Service 通信是通过新建一个 ServiceConnection，并得到相关的 AIDL 接口来实现的，不与 Activity 组件进行强绑定。由于在 Android 高版本（貌似4.0）以上已经不允许泛访问（不能像 Activity 那样不指定 ComponentName，而让用户选择） Service，所以需要先指定 Service （你也可以自己做一个 UI 让用户选择 Service，可通过 PackageManager.queryIntentServices 查询）。

调用接口 IUAFOperation.aidl ：

```
// IUAFOperation.aidl 
package com.excelsecu.uaf.client;

import com.excelsecu.uaf.client.IUAFResponseListener;

// Declare any non-default types here with import statements 
interface IUAFOperation {

    /**
     *
     * The oneway keyword modifies the behavior of remote calls. When used , a remote call does not 
     * block; it simply sends the transaction data and immediately returns. The implementation of 
     * the interface eventually receives this as a regular call from the Binder thread pool as a 
     * normal remote call. If oneway is used with a local call , there is no impact and the call 
     * is still synchronous.
     */
    oneway void process(in Intent uafRequest , in IUAFResponseListener uafResponseListener);
}
```

`oneway`：表示异步，调用后会立刻返回。

回调接口 IUAFResponseListener.aidl ：

```
// IUAFResponseListener.aidl 
package com.excelsecu.uaf.client;

// Declare any non-default types here with import statements 

interface IUAFResponseListener {

    /**
     * @param uafResponse the Intent that would be passed to onActivityResult.
     */
    void onResult(in Intent uafResponse);
}

```

实现类 ClientServiceCommunicator.java ：

```
private static class ClientServiceCommunicator implements ClientCommunicator {
    private ServiceConnection conn;
    private IUAFOperation operation;  // UAF Client 规范定义的 AIDL 
    private ComponentName clientComponentName; // 指定要通信的 Service 
    private Context context = null;

    public ClientServiceCommunicator(ComponentName clientComponentName){
        this.clientComponentName = clientComponentName;
    }

    @Override 
    public void sendRequest(Intent clientRequest , final IUAFResponseListener listener){
        if(operation == null){
            Log.w("TAG","client ServiceCommunicator not init yet");
            return;
        }
        clientRequest.setComponent(componentName); // 设置通信对象 Service 的 componentName 

        try {
            operation.process(clientRequest , new IUAFResponseListener(){
                @Override 
                public void onResult(Intent uafResponse)throws RemoteException {
                    if(listener != null){
                        listener.onResult(uafResponse);
                    }
                }

                // AIDL 自动生成的接口方法，在回调接口中无用处
                @Override 
                public IBinder asBinder(){
                    return null;
                }
            });
        } catch(RemoteException e){
            KLog.e(e);
        }
    }

    @Override 
    public boolean init(Context context){
        this.context = context;
        PackageManager manager = context.getPackageManager();
        Intent intent = new Intent();
        intent.setAction("org.fidoalliance.aidl.FIDO_OPERATION");
        intent.setType("application/fido.uaf_client + json");

        // 寻找声明了 UAF Client 的 Service 中是否有和传入 componentName 匹配的 Service 
        List<ResolveInfo> infoList = manager.queryIntentServices(intent , 0);
        boolean found = false;
        for(ResolveInfo info : infoList){
            if(clientComponentName == null ||
                   (info.serviceInfo.name.equals(clientComponentName.getClassName())
                            && info.serviceInfo.packageName.equals(clientComponentName.getPackageName()))){
                intent.setComponent(clientComponentName);
                found = true;
                break;
            }
        }
        if(! found){
            Log.w("TAG","uaf client service not found with name:"+ clientComponentName);
            return false;
        }

        // 与 Service 建立通信。注意外部可能是从主线程调用 init，这里如果加锁等待 bindService 结果，就会造成死锁（ ServiceConnection 回调也在主线程）。但不加锁又不能保证下一次调用的时候已经获得了 IUAFOperation 对象。因为 demo 的操作是 button 触发的，所以基本不会出现调用时 IUAFOperation 仍未获取到的问题。而后面 UAF Client 的实现中解决了这个问题。
        conn = new ServiceConnection(){

            @Override 
            public void onServiceConnected(ComponentName componentName , IBinder iBinder){
                operation = IUAFOperation.Stub.asInterface(iBinder);
            }

            @Override 
            public void onServiceDisconnected(ComponentName componentName){
            }
        };
        return context.bindService(intent , conn , Context.BIND_AUTO_CREATE);
    }

    @Override 
    public void release(){
        if(context != null && conn != null){
            context.unbindService(conn);
        }
        conn = null;
        operation = null;
        context = null;
    }
}
```

##### 4.2.2.3.3 Activity 方案与 Service 方案的封装

## 4.3 UAF Client 

从 client application 层我们可以知道 UAF Client 功能就是支持解析`DISCOVER`，`CHECK_POLICY`，`UAF_OPERATION`，`UAF_OPERATION_COMPLETION_STATUS`4种命令。UAF Client 可用 Activity 和 Service 作为接口接收并处理上层消息。

UAF Client 与 UAF Server 以及 UAF ASM 通信的数据皆为 UTF-8编码的 json 数据，并使用 gson 转换为对象。

本章节对应文档为 fido-uaf-protocol-v1.0-ps-20141208。

### 4.3.1 UAF Client 的生命周期

FIDO UAF 规范没有对 UAF Client 的生命周期做有关描述。我从无状态，节省内存占用需求考虑的出发，在空闲状态下，Activity 应为完全退出状态（ onDestroy ），Service 最好应为 unbinded 状态（由上层是否 unbindService 决定）。各模块应在信息处理前进行初始化，处理完信息后 release 掉（大部分持有 context ）。这样可以保证 UAF Client 不会长期占用内存。且因为无状态的设计，如果发生一次错误，不会影响到下一次操作。缺点是每次都要进行初始化，会消耗更多的时间（实测没有太大影响）。

#### 4.3.1.1 UAF Client 的并发考虑
在理想的情况下，UAF Client 一次只处理一条信息，等到信息返回后再处理下一条信息。如果在一条信息的处理过程中 UAF Client 又收到了一条新信息，此时应阻塞，或返回`WAIT_USER_ACTION`错误码。

对于`WAIT_USER_ACTION`的解释是，在 UAF Client 处理信息的过程中，可能会出现多次需要与用户交互的情况，此时如果有消息进来，则回复这个错误码给上层，说明 UAF Client 正忙。但在 UAF Client 正在处理自己的逻辑，还没到等待用户交互的时间间隔应该怎么做呢？我个人认为应该也返回`WAIT_USER_ACTION`。这样就保证了一次只处理一条信息的设计。而如果要严格按照 FIDO UAF 规范，只在等待用户交互时返回`WAIT_USER_ACTION`，处理逻辑时阻塞等待，直到上一条信息处理到等待用户交互时名，该条信息返回`WAIT_USER_ACTION`，或上一条信息出错了，则开始处理该条信息。这样的逻辑没有太大意义，而且略显臃肿。

UAF Client 什么时候会出现并发？可能是调用方连续发了数条信息进来，或者是用户操作一个 app 向 UAF Client 发送信息，又切换到另一个 app 向 UAF Client 发送信息。虽然是一种极端的使用方法，但 UAF Client 应能处理这种情况。

#### 4.3.1.2 UAF Client 的简单并发处理
参考 linux 驱动的做法，我声明了一个计数器，用于记录当前有多少请求进入。简单来说，外部不直接调用`init()`和`release()`，而是调用`comeIn()`和`comeOut()`。当前没有请求时（ count=0），调用`comeIn()`会执行`init()`，当前总请求数为1时，调用`coumeOut()`会执行`release()`。


还有一个情况，当用户强制退出 UAF Client （返回键或其他方式）时，在 Activity 或 Service 的`onDestroy()`出需要执行 release()，否则下一次 app 发送请求给 UAF Client 时 UAF Client 的当前状态是错误的。

##### 4.3.1.2.1 初始化并发处理
UAF Client 逻辑实现类 ClientExecutor.java ：

```
public static final int STATE_READY = 1; // Executor 初始化完成且空闲
public static final int STATE_INITIALIZING = 2; // Executor 正在初始化
public static final int STATE_ASM_NOT_FOUND = 3; // Executor 没有找到至少一个 ASM （非必需）
public static final int STATE_ERROR = 4; // Executor 初始化失败
public static final int STATE_NOT_START = 5; // Executor 未初始化
public static final int STATE_EXECUTING = 6; // Executor 正在处理信息

private int processingCount = 0;
private int runningState = STATE_NOT_START; // 描述处理类当前状态

public synchronized void comeIn(Context context , boolean isActivityMode){
    processingCount ++;
    Log.d("TAG","ClientExecutor comeIn , count"+ processingCount);
    init(context , isActivityMode);
}

public synchronized void comeOut(){
    processingCount--;
    Log.d("TAG","ClientExecutor comeOut , count"+ processingCount);
    if(processingCount == 0){
        release();
    }
}

/**
 *
 * @param isActivityMode the way to search UAF ASM model. ATTENTION: not the way to search UAF Client model.
 */
private void init(Context context , boolean isActivityMode){
    Log.d("TAG","ClientExecutor init"+ runningState);
    this.context = context;
    if(runningState == STATE_READY || runningState == STATE_INITIALIZING || runningState == STATE_EXECUTING){
        return;
    }

    runningState = STATE_INITIALIZING;
    
    // do something initialize 
}

public void release(){
    Log.d("TAG","ClientExecutor release");
    processingCount = 0;
    helper.release();
    context = null;
    runningState = STATE_NOT_START;
}
```

##### 4.3.1.2.2 信息并发处理

ClientExecutor 提供了一个唯一的入口`execute()`：

```
private ExecutorService threadPool = Executors.newCachedThreadPool(); // cache 线程池，因为不是高并发，简单的 ExecutorService 即可满足要求

// 每个任务都开启一个线程去执行
public void execute(final Intent uafRequest , final IUAFResponseListener uafResponseListener , final String facetID , final String componentName){
    threadPool.execute(new Runnable(){
        @Override 
        public void run(){
            Intent response = syncExecute(uafRequest , facetID , componentName);
            if(uafResponseListener != null){
                try {
                    uafResponseListener.onResult(response);
                } catch(RemoteException e){
                    Log.e("TAG", e);
                }
            }
        }
    });
}

private Intent syncExecute(final UAFIntent content , String facetID , String componentName){
    if(runningState != STATE_READY){
        long time = System.currentTimeMillis();
        // 初始化是异步的，防止主线程阻塞。初始化结果通过回调返回。等待时间为 INIT_TIMEOUT_MILL，等待时睡眠间隔为 WAIT_GAP 
        if(runningState == STATE_INITIALIZING){
            // wait INIT_TIMEOUT_MILL 
            int count = INIT_TIMEOUT_MILL/WAIT_GAP;
            while((count--> 0)&&(runningState != STATE_READY)){
                try {
                    Thread.sleep(WAIT_GAP);
                } catch(InterruptedException e){
                    KLog.e(e);
                }
            }
        }

        // 如果已有任务在处理，则直接返回 WAIT_USER_ACTION。如果 Executor 初始化错误，则返回 UNKNOWN 
        if(runningState == STATE_EXECUTING){
            return new UAFIntent(content.uafIntentType , componentName , ErrorCode.WAIT_USER_ACTION).toIntent();
        } else if(runningState != STATE_READY){
            KLog.w("client executor not ready , state"+ runningState);
            return new UAFIntent(content.uafIntentType , componentName , ErrorCode.UNKNOWN).toIntent();
        }
        KLog.d("initialize took"+(System.currentTimeMillis()-time)+"mills");
    }

    runningState = STATE_EXECUTING;
}
```

之前 client application 提到了一个初始化的缺陷：在主线程执行初始化时，加锁会导致死锁，不加锁（立即返回）可能导致初始化未完成。解决方法就是初始化时立即返回，异步通知初始化结果，通过变量记录初始化状态，并在执行方法`execute()`中加入初始化的延时判断。

#### 4.3.1.3 ASM 命令简介
在介绍 UAF Client 各个命令实现之前，需要先介绍 ASM 支持的命令，因为 UAF Client 的功能是依靠 ASM 的功能来实现的。既然上层依赖下层，那为什么本文不选择从下往上的介绍方式，而是选择从上往下的介绍方式呢？因为从上往下更符合人的思考习惯，整个系统的结构会更加清晰。我在学习 FIDO UAF 的时候是从下往上看的，结果看第一遍的时候很多东西都没有理解到位。

ASM 命令列表：

* GetInfo ：获取该 ASM 当前可用的所有 Authenticator 的信息。信息中包括名称，版本，标识符 AAID，以及其支持的特性。获取后可与这些 Authenticator 进行通信。
* Register ：指定向某个 Authenticator 发起注册请求（对应`reg`操作），Authenticator 必须是 GetInfo 中获取到的。
* Authenticate ：指定向某个 Authenticator 发起认证请求（对应`auth`操作），Authenticator 必须是 GetInfo 中获取到的。
* Deregister ：指定向某个 Authenticator 发起注销请求（对应`dereg`操作），Authenticator 必须是 GetInfo 中获取到的。
* GetRegistrations ：向 ASM 查询目前 GetInfo 中所有 Authenticator 的所有已注册 keyID，以及 keyID 对应的 appID。
* OpenSettings ：要求某个 Authenticator 打开设置界面，该 Authenticator 需要声明其支持打开设置界面这个特性。

#### 4.3.1.4`DISCOVER`命令实现

UAF Client 在收到`DISCOVER`命令时，会向设备上所有的 ASM 发送 GetInfo 命令，然后收集起来返回给上层。具体格式在 client application 中已描述。

遍历 ASM 之后需要存储各个 Authenticator 是来自哪个 ASM，以便针对特定 Authenticator 的操作可以发送到正确的 ASM 中。详细细节在4.3.1.8描述。

> 在 FIDO UAF 的正式测试中，各个厂商的 UAF Client 是会发送给各个厂商的，但 ASM + Authenticator 一般不会发出来，只交给自己演示并测试。所以设备上很可能只存在一个 ASM。则遍历 ASM 就无从测试了。当时有一个厂商发了 ASM + Authenticator 出来，导致某些 Client 在装了多个 ASM 的情况下，没办法正确选择 Authenticator。猜测他们的 Client 只选择第一个，并没有遍历功能。
> 
> 在测试过程中，ASM 这个概念是被隐藏的。Authenticator 默认就是指 ASM + Authenticator，一个 Authenticator 就配着一个 ASM，为一个独立的 apk。原因是 Authenticator 没有直接的通信接口（无法通过 Intent 通信），而 ASM 有 Intent 通信接口。


#### 4.3.1.5 CHECK\_POLICY 命令实现

UAF Client 在收到 CHECK\_POLICY 命令时：

1. 向设备上所有的 ASM 发送 GetInfo 命令，然后收集起来；
2. 提取命令中的`header`和`policy`字段，分别对应数据结构`OperationHeader`和`Policy`；
3. 对每一个 GetInfo 得到的 Authenticator，分别发送 GetRegistration 命令（需要传入 header 字段），取得其对应 appID 注册过的 keyIDs；
4. 根据 header 和 policy 字段，过滤这些 Authenticator 及其对应的 keyIDs；
5. 将过滤后剩下的 Authenticator 返回给上层（如果全部被过滤掉了，则为空）。

这里涉及到对 UAF Server 传来的协议消息中的 header 和 policy 的处理，下面将详细描述。

##### 4.3.1.5.1 UAF Server 协议信息
在发送`reg`，`auth`，`dereg`请求后，UAF Server 会返回对应的协议信息（ protocol message ）交给 UAF Client 处理。`reg`请求会返回`RegistrationRequest`数据结构，`auth`会返回`AuthenticationRequest`数据结构,`dereg`会返回`DeregistrationRequest`数据结构。共同点是，这三个数据结构都包含了`header`和`policy`。CHECK\_POLICY 只会用到这两个字段，所以下面只讨论这两个。

##### 4.3.1.5.1 OperationHeader 数据结构
操作函数头 OperationHeader.java ：

```
public class OperationHeader {
    @Required Version   upv; // 该信息的协议版本，发送任意的命令给 ASM 时需要附带上协议版本，如果 ASM 不支持则拒绝该命令。
    @Required Operation op; // 该信息的操作类型。reg，auth，dereg 其中之一
    String              appID; // application ID。有可能是一个 https 链接，有可能是 app 的签名值。
    String              serverData; // 由于 UAF Server 是 RESTful Web Service，所以需要把状态信息记录在 serverData。操作完成后回复需要把该字段不作修改返回
    Extension[]         exts; // UAF Server 与 UAF Client 约定好的额外信息提供，如果没有则为 null。
};
```

其中`upv`的数据类型`Verion`描述了版本信息，主要用于描述 UAF 协议版本。（也有一个地方用于描述 UAF Client 的版本，而不是协议版本）。Version.java ：

```
public class Version {
    public int major; // 主版本
    public int minor; // 副版本
}
```

即 major.minor。如规范1.0则是 major=1，minor=0。

##### 4.3.1.5.2 Policy 数据结构

策略对象 Policy 会包含允许规则和不允许规则。Policy.java ：

```
public class Policy {
    @Required public MatchCriteria[][] accepted; // 策略允许的 authenticator 的规则
    public MatchCriteria[]            disallowed; // 策略不允许的 authenticator 的规则
}
```

`accepted`是一个二维数组。第一维（`MatchCriteria[]`的数组）是“或”的关系，一个 Authenticator 如果不匹配第一组`MatchCriteria[]`规则，但是匹配第二组`MatchCriteria[]`规则，也算是匹配上了这个 Policy。第二维（`MatchCriteria`的数组）是“且”的关系，用于实现多 Authenticator 组合认证的功能。比如`MatchCriteria[]`长度为2，且第一个`MatchCriteria`匹配了 Authenticator a，第二个`MatchCriteria`匹配了 Authenticator b，则该`MatchCriteria[]`要求 ab 组合认证。

> accepted 部分稍难理解，我反复理解错对了若干次。请结合规范3.1.12阅读。

`disallowed`是一个一维数组。一个 Authenticator 匹配上数组中任意一个 MatchCriteria 规则，都算是不匹配这个 Policy。

FIDO UAF 文档中要求： UAF Client 从所有 UAF ASM 那得到所有可用的 Authenticator 集合后，先使用过滤掉匹配`disallowed`规则的 Authenticator，再用剩下的 Authenticator 去匹配`accepted`，匹配上`accepted`的 Authenticators 会留下来。

策略匹配规则 MatchCriteria.java ：

```
public class MatchCriteria {
    public AAID[]           aaid; // AAID 是 Authetnicator 的唯一标识，需要向 FIDO 联盟申报。至少匹配其中一个
    public String[]         vendorID; // 厂商 ID，是 AAID 的前半部分。至少匹配其中一个
    public String[]         keyIDs; // 注册时会提交给 Server 的 keyID，至少匹配其中一个
    public Integer          userVerification; // 这个判断稍微复杂一点，详情看文档3.1.11.1
    public Short            keyProtection; // 要求至少匹配其中一个 bit 
    public Short            matcherProtection; // 要求至少匹配其中一个 bit 
    public Integer          attachmentHint; // 要求至少匹配其中一个 bit 
    public Short            tcDisplay; // 要求至少匹配其中一个 bit 
    public Short[]          authenticationAlgorithms; // 至少匹配其中一个
    public String[]         assertionSchemes; // 至少匹配其中一个
    public Short[]          attestationTypes; // 至少匹配其中一个
    public Short            authenticatorVersion; // Authenticator 的版本，非协议版本。要求该版本值低于 Authenticator 中的 TAG_UAFV1_REG_ASSERTION 或 TAG_UAFV1_AUTH_ASSERTION 的结构中包含的 authenticatorVersion 版本值。
    public Extension[]      exts; // 文档只提到匹配 policy 的扩展 Extension，应为 UAF Server 与 UAF Client 约定好的辅助匹配的数据，没有硬性要求
```

MatchCriteria 的数据结构没有使用原生类型（ int，short 等），因为在源 json 数据中，MatchCriteria 的成员都有可能不存在。如果使用对象不存在的成员则可以直接使用 null 代替。如果使用原声类型，在使用过程中就不好分辨是默认值还是源数据的值。

##### 4.3.1.5.3 Policy 的规则和使用情况

Policy 和 MatchCriteria 看起来有很多匹配的规则，很复杂，但实际上用到情况很简单。在 FIDO UAF 组织的测试中，只出现了 AAID 和 keyIDs 的组合情况。

典型的使用情况如下：

* reg 注册时`accepted`为1 x 1的二维数组，MatchCriteria 只有 aaid 有值，为一串可接受的 AAID 列表，因为 AAID 是 Authenticator 的唯一标识，所以这样的`accepted`表示 UAF Server 接受的 Authenticator 的列表。`disallow`是长度为 n 的一维数组，每个 MatchCriteria 包含的是`GetUAFRequest`中提交的 username 对应的已注册的 keyID （通常为1个）以及对应的注册的 Authenticator 的 AAID。通过这样的方法来告知 UAF Client 这个 username 已经在哪几个 Authenticator 注册过，并且 keyID 是多少。如果 UAF Client 从 ASM 处查询到的 Authenticator 及 GetRegistration 信息确实匹配 UAF Server 所提供的`disallowed`，则会过滤掉这个 Authenticator，意思是你已经在这个 Authenticator 注册过了，没有必要再注册一次了。但如果 keyID 和 UAF Client 查询到的不吻合，说明可能数据有误，或者 Authenticator 的数据丢失或清空了。这时候就可以重新注册，得到新的 keyID 并传回 UAF Server。
* （带 username 的） auth 认证时`accepted`和注册时`disallow`很相似为1 x n 的二维数组，每个 MatchCriteria 包含的是`GetUAFRequest`中提交的 username 对应的已注册的 keyID （通常为1个）以及对应的注册的 Authenticator 的 AAID，意思是你只可以用我 UAF Server 已经记录的 Authenticator 以及对应的 keyID 做认证，不然你用没注册过的 Authenticator 和 keyID 做认证，肯定是要被拒绝的。`disallow`一般为空，因为`accepted`已经限制的很死了。
* （不带 username 的） auth 认证时`accepted`和注册时的`accepted`是一样的，都是一串可接受的 Authenticator 的 AAID 列表。因为你没有提供 username （也就是说没有 session ），UAF Server 没办法提供对应的注册信息（总不能把整个数据库拖给你吧）。然后 UAF Client 会调用 ASM，ASM 会让用户选一个已经在本设备注册过的 username，然后认证并返回结果。`disallow`一般为空，没有 session UAF Server 也没有什么好提供的。
* dereg 注销时和带 username 的 auth 认证是一样的，提供已注册的 keyIDs 以及对应的 Authenticator 信息。

##### 4.3.1.5.4 检测 Policy，过滤 Authenticator 的实现方法
虽然说实际使用情况很简单，但你总不能就只写这几种情况吧（这样 UAF FIDO 的 automated test 会不通过）。下面将描述如何从一堆 Authenticator 以及对应注册了的 keyIDs 中用 Policy 进行过滤，得到结果。

本方法稍微偏理论，能够以不变应万变，但实现起来有点晦涩，易读性有点差。实现者可酌情简化。

文档中对`MatchCriteria`成员的组合有一些的规定（见文档3.4.6）：

* aaid 可以和 keyIDs，attachmentHint，authenticatorVersion，exts 成员组合使用，不可以和其他成员组合使用；
* 如果不提供 aaid，则至少需要提供 authenticationAlgorithms 和 assertionSchemes。

在 FIDO UAF automated test 中也有一些关于 MatchCriteria 非法组合的测试，我在匹配规则里有加上这些非法组合的判断，但实现逻辑没有根据这些条件简化处理逻辑（因为这属于 Server 部分，我很晚才发现这段描述）。我实现的方式是更范式的方法。整个 Policy 处理流程如下：

1. 通过 ASM 的 GetInfo 命令获取所有设备上的 Authenticator 的信息`AuthenticatorInfo`；
2. 通过 ASM 的 GetRegistration 命令获取所有 Authenticator 的已注册 keyID 信息，并与 Authenticator 组成一个整体（以下称其为 A & K；
3. 使用`disallow`对 A & K 进行匹配过滤，去掉匹配上`disallow`的 A & K；
4. 新建一个列表，用于存储使用`accepted`匹配的 A & K 组合。该列表的成员也是列表，每个子列表代表每个`MatchCriteria[]`匹配的 Authenticator 组合（由于`MatchCriteria[]`大部分情况长度是1，所以子列表大部分情况长度也是1）
5. 对`accpeted`进行遍历：
    1. 对每一个`MatchCriteria`匹配到的 A & K （一个`MatchCriteria`可能匹配上多个 A & K ）建立一个列表`l`，建立一个新对象`FilteredAuthenticator`，`FilteredAuthenticator`会存储：
        1. `AuthenticatorInfo`。从 GetInfo 中得到。
        2. `keyIDs`。值为 A & K 中的 keyIDs 与`MatchCriteria`的 keyIDs 的交集（这点很重要，你只能使用`MatchCriteria`提供的 keyID，如果 keyID 为空，则将 A & K 的 keyID 设为空列表）。
        3. `atteatationType`。这个变量表示 Authenticator 的认证方式。目前 Authenticator 可能支持多种认证方式（使用预植证书或使用自生成证书的认证方式，在 UAF Authenticator 中会介绍）。如果`MatchCriteria`含有`atteatationType`，则你只能使用`MatchCriteria`的`atteatationType`，即是你支持多种认证方式。该变量在后面签名时会用到。
    2. 当遍历完一个`MatchCriteria[]`集合时，对所有的`l`进行排列组合，并去重。例子：

    * 在极端条件下，如： mm[0]：要求支持指纹，mm[1]：要求支持面部识别，mm[2]：要求支持 RSA 算法。而现在符合 mm[0]的有 a，b，c 三个，符合 mm[1]有 d，e，符合 mm[2]有 a，c，g。即每个`MatchCriteria[]`{ a , b , c },{ d , e },{ a , c , g }
    * 那么，符合策略的组合就有3 * 2 * 3 = 18个。但中间会有比如 ada 这样的重复组合，也有 adc，cda 这样的重复组合，这些组合需要去掉
    * 所以先将所有的组合先进行排序，然后将 ada 这样的组合剔除（ ada 变成了 aad ），再将完全一样的组合剔除（ adc，cda 都变成了 acd ）。剩下的就是可供选择的 Authenticator 组合。（是不是觉得有点杀鸡用牛刀了，但规范确实没有很完整的限制 MatchCritia 的组合情况，只考虑几个情况以覆盖大部分又让我很不安。。）

如果想简单的实现 Policy 策略，建议从有/没有 aaid，有/没有多个`MatchCriteria[]`组合进行分支处理。

Policy 的返回数据在 client application 中已描述。找到至少一个 authenticator 则返回`NO_ERROR`否则返回`NO_SUITABLE_AUTHENTICATOR`。

#### 4.3.1.6 UAF\_OPERATION 命令实现

UAF\_OPERATION 即 UAF Client 的核心功能：为用户进行 reg 注册，auth 认证和 dereg 注销操作。对于该三个操作，protocol 文档提供了约定的处理步骤，本节将以注释的形式丰富说明，帮助理解。

##### 4.3.1.6.1 reg 注册实现

参考 protocol 文档3.5.7.2：

客户端处理 reg 注册**必须**遵循以下步骤：

1. 选择主要版本为`1`，次要版本为`0`的消息 m。

    > UAF Server 返回的数据其实是一个数组，里面可能包含多个不同版本的消息。对于实现规范1.0的 UAF Client 来说，就需要选择 OperationHeader 中的 Version 为1.0的消息，规范1.1则选择 Version 为1.1的消息。这样做可以使 UAF Server 同时为不同版本的 UAF Client 提供服务。  
    > 按理说 UAF Client 只需要支持一个版本即可。由于 UAF 规范是向下兼容的，而我一开始实现的是1.2的规范版本（官网上只发布到1.1，估计可能会跳过直接发布2.0），所以我在该步骤的实现是：只要是1.2以下的消息 m 我都可以接受。这种做法没有进过严格考究，目前（2017/03/27）来说只支持一个1.0版本即可。

2. 解析消息 m。
    
    > 即把消息 m 转换成 RegistrationRequest 对象：  
    
    ```
    public class RegistrationRequest {  
        @Required public OperationHeader header; // 消息头
        @Required public String challenge; // 挑战码，签名会用到，用于保证消息的随机性和唯一性
        @Required public String username; // 注册的用户名
        @Required public Policy policy; // Authenticator 选择策略
    } 
    ```
    

3. 如果 UAF 消息中的必填字段为空或者某字段的类型和值不符，则拒绝该操作。

    > 如何检测必填字段为空或者某字段的类型和值不符：请看文档 [TODO]()

4. 用给定个策略筛选现有的 Authenticator，并将已筛选的 Authenticator 呈现给用户。确保不包含 RegRequest.policy.disallowed[].keyIDs 中该用户已注册的 Authenticator。
    
    > 筛选即 policy 的过滤，详见本文档4.3.1.5.2和4.3.1.5.3。
    >
    > 此步需要弹出 UI 界面呈现给用户，让用户选择用于注册的 Authenticator 组合。根据 policy，一个组合可能是由多个 Authenticator 组成的，弹出的 UI 界面必须以组合为单位提供用户选择。此处还有2个问题：
    > 
    > 1. 只找到一个 Authenticator 组合的时候是不是直接选默认即可，不用弹出 UI 界面？不强制，大部分厂商不会弹出 UI 界面。但我觉得需要让用户知道将要注册的 Authenticator 是哪个，所以我在只有一个 Authenticator 组合可选的时候也会弹出 UI。
    > 2. 是否允许同时选择多个组合（组合是单选还是多选）？没有明确说明。但显然，单选一个组合更符合一般的使用逻辑。没事同时注册好几个干嘛？
    > 
    > 以上两个问题在 FIDO UAF 正式测试中不会被测到，因为测试环境是单用户，单 Authenticator。
    
5. 获取请求应用的`FacetID`。如果`AppID`缺失或为空，将 `AppID`设定为`FacetID`。  
    根据[FIDOAppIDAndFacets]中的算法，校验 FacetID 对于此 AppID 是被授权的。
    
    > FacetID，可以理解为调用方的信息，在 Android 中就是调用 UAFClient 的 app 的签名信息。注意是调用者的信息，因为 FacetID 用于鉴别该 app 是否是 UAF Server 允许交互的。如果你用的是一个山寨的 app，签名信息不一样，这一步就会被 UAF Client 拒绝。FacetID 由 UAF Client 主动获取，在系统级别上防止了 FacetID 伪造的可能性。
    > 
    > AppID，我没有很好的理解它，但它像是一个比 FacetID 更宽泛的 ID，一个 AppID 可能对应多个 FacetID。AppID 会成为到签名报文的一部分，所以伪造 AppID 的签名结果将会被 UAF Server 拒绝。
    > 
    > * 当 AppID 为 https 的一个 url 时，它指向的是一个 TrustedFacetList，UAF Client 需要使用 Http Get 方法获取这个 list。该 list 为信任的 FacetID 列表。UAF Client 需要在这个 list 中找到与调用 app 的 FacetID 匹配的条目。对于 Android 来说（ FacetID 为签名信息），需要在信任列表中找到一致的 facetID，对于 Https 的 facetID 来说，则需要匹配上其 public suffix；
    > * 当 AppID 为非 url 时，AppID 需要与 facetID 一致；
    > * 当 AppID 为空时，意思是 UAF Server 允许任何 facetID 通过。在这里 AppID 会被设置为 FacetID。** ASM 收到的 AppID 就是 FacetID。**
    > 
    > 更具体的描述见规范 fido-appid-and-facets-v1.0-ps-20141208。该规范也包含了各个系统获取 FacetID 的方法

6. 如果 TLS 数据可用，则获取该数据。 

    > TLS 数据由 client application 传过来，可能为空（ empty ）。TLS 用于生成 json 格式的 String `fcp`变量，`fcp`变量会加入到签名报文。如果 TLS 是 empty 的，也把 empty 的`channelBinding`的加入`fcp`（**否则不能通过 FIDO automated test **）。
    > 
    > 目前（20170327）没有看到厂商的 UAF Server 支持 TLS。似乎是如果 UAF Server 用了 nginx 或者 apache 就没办法获取到这些底层数据？（该说法存疑）。因此我们的 UAF Client 也没有支持 TLS。

7. 创建 FinalChallengeParams 结构`fcp`，并正确设定`fcp.AppID`，`fcp.challenge`，`fcp.facetID`和`fcp.channelBinding`。使用 UTF8编码`fcp`将结果序列化[RFC4627]后在使用 base64url 进行编码。
    * `FinalChallenge = base64url(serialize(utf8encode(fcp)))`

    > `fcp`结构如下：
    
    ```
    public class FinalChallengeParams {
        @Required public String             appID; // 消息 m 提供的 appID 
        @Required public String             challenge; // RegistrationRequest 中的 challenge 
        @Required public String             facetID; // UAF Client 获取的 facetID 
        @Required public ChannelBinding     channelBinding; // 消息 m 提供的 channelBinding 
    }
    ```
    
    > 将`FinalChallengeParams`转换成 json 格式后，获取 UTF8编码的 bytes，然后用 Base64编码。即 toJson()->getBytes("UTF-8")->encode()。
    > 
    > 强烈建议在所有用到 Base64编码的地方使用 Flag `Base64.NO_WRAP | ESBase64.NO_PADDING | ESBase64.URL_SAFE`。**服务器验签不过可能是因为你 Base64编码的 Flag 和 UAF Server 不一致**。（但上文提到的，获取 facetID 也用到了 Base64除外。因为已经提供了官方代码，且 facetID 不会发送到 UAF Server ）

8. 对每个匹配的 UAF 协议版本（参考版本协商部分）并且用户同意注册的 Authenticator：
    1. 添加 AppID，Username，FinalChallenge，AttestationType 和所有其他**必须**的字段到 ASMRequest [UAFASM]。  
    FIDO UAF 客户端**必须**遵守服务器策略并且找出最优的鉴别类型。单独的见证类型必须提供给 ASM 
    2. 发送 ASMRequest 给 ASM。

    > 该步要求 Authenticator 是匹配的 UAF 协议版本并且用户同意注册的。以我的理解，**“匹配的 UAF 协议版本的 Authenticator”应该在用户同意注册前处理**，否则用户选择了一个不匹配 UAF 协议版本的 Authentcator，就会发生错误了。
    > 
    > 版本协商部分中说明：每一个 ASM 都要记录下其对应的协议版本，每一个 Authenticator 也要记录下对应的协议版本。根据选择的消息`m`的`upv`，也就是消息`m`的协议版本，来选择兼容的 ASM 和 Authenticator ：
    > 
    > 1. 收集完设备上的 ASM 后（ Android 通过 Intent，4.3.1.9有介绍），根据 ASM 版本过滤掉不兼容的 ASM，只向兼容的 ASM 下发 GetInfo 命令。根据 UAF 向下兼容的特性，1.1可以兼容1.0，1.2可以兼容1.1和1.0。
    > 2. 从兼容的 ASM 中得到 Authenticator 信息后，根据 Authenticator 协议版本过滤掉不兼容的 Authenticator。
    > 3. 剩下的就是兼容的 Authenticator。注意：**`DISCOVER`不存在版本协商，只有 UAF\_OPERATION 才有版本协商，因为`DISCOVER`因为没有提供`upv`。**
    > 
    > 目前（20170327）普遍只测1.0规范的情况下，该点很难被测试到。
    > 
    > 发送 ASMRequest 即发送 Register 命令。比如你要对 Authenticator a，b，c进行注册，a，b属于 ASM A，c属于 ASM B。则执行三次Register命令，两次发给ASM A，一次发给ASM B（命令会通过`authenticatorIndex`指定需要和哪个Authenticator通信）
    >
    > 仍需注意的一点是：如果你需要注册3个 Authenticator，第一个成功了，但第2个失败了，应该怎么办？规范没有说明，我认为应该注销掉第一个，然后返回上层失败的信息。虽然不注销对整个系统也不会有影响（其实还是会有，比如失败太多把密钥存储撑爆了），但我觉得保持系统清洁是才是最吼的。

9. 创建`RegistrationResponse`信息。
    > RegistrationRespons.java
    
    ```
    public class RegistrationResponse {
        @Required OperationHeader header; // 消息 m 中的 OperationHeader。
        @Required String fcParams; // 上面算出来的FinalChallenge
        @Required AuthenticatorRegistrationAssertion[] assertions; // 各个Authenticator通过ASM返回的数据
    };
    ```
    
    > 注意！**如果`header`中`appID`为空，这里也必须为空，不能赋值为`facetID`**。很容易犯错的一点就是，当appID为空时，直接把`header.appID`赋值为`facetID`，导致`header`的值发生了变化，从而导致测试不通过。正确做法是：`appID`设为`facetID`对`fcp`和`ASM`层命令有效，对`OperationHeader`无效。


##### 4.3.1.6.2 auth 注册实现
由于 reg 和 auth 步骤相似，所以雷同的步骤请直接参考 reg。 

参考 protocol 文档3.5.7.2 和 3.4.6.4：

客户端处理 auth 注册**必须**遵循以下步骤：

1. 选择主要版本为`1`，次要版本为`0`的消息`m`。
2. 解析消息`m`。
    * 如果 UAF 消息中的必填字段为空或者某字段的类型和值不符，则拒绝该操作。
    
    > 即把消息 m 转换成 RegistrationRequest 对象：  
    
    ```
    public class AuthenticationRequest {
        @Required public OperationHeader   header; // 消息头
        @Required public String            challenge; // 挑战码，签名会用到，用于保证消息的随机性和唯一性
        public Transaction[]               transaction; // 交易信息，交易认证才会有，登陆（身份）认证没有
        @Required public Policy            policy; // Authenticator 选择策略
    ```
    
3. 获取请求应用的`FacetID`。如果`AppID`缺失或为空，将`AppID`设定为`FacetID`。  
根据[FIDOAppIDAndFacets]中的算法，校验`FacetID`对于此`AppID`是被授权的。
    * 如果请求应用的`FacetID`没有呗授权，拒绝该操作。
4. 根据给定的策略过滤有效 Authenticator，并将过滤的列表呈现给用户。
    * 如果`AuthenticatorRequest.policy.accepted`列表为空，则给用户推荐任意已注册的 Authenticator 用于鉴别。
5. 让用户选择其首选的 Authenticator。
6. 如果TLS数据可用则获取该数据。
7. 创建FinalChallengeParams结构`fcp`，并正确设定`fcp.APPID`，`facp.challenge`和`fcp.channelBinding`。使用 UTF8编码`fcp`，将结果序列化[RFC4627]后再使用 base64url 进行编码。
    * `FinalChallenge = base64url(serialize(utf8encdoe(fcp)))`
8. 对每个支持与消息版本AuthenticationRequest.header.upv（参考版本协商）兼容的 Authenticator 接口版本（AIV）的 Authenticator 以及用户同意用于鉴别的 Authenticator：
    1. 将`AppID`，`FinalChallenge`，`Transactions`（如果存在）和其他所有字段添加到 ASMRequest。
    2. 将 ASMRequest 发送到 ASM。

    > 发送 ASMRequest 即发送 Authenticate 命令。
    > 
    > AIV = Authenticator Interface Version，没有在其他地方找到相关解释，猜测就是Authenticator protocol Version。

9. 创建`AuthenticationResponse`信息。
    > AuthenticationResponse.java
    
    ```
    public class AuthenticationResponse {
        @Required public OperationHeader header; // 消息 m 中的 OperationHeader。
        @Required public String fcParams; // 上面算出来的FinalChallenge
        @Required public AuthenticatorSignAssertion[] assertions; // 各个Authenticator通过ASM返回的数据
    }
    ```


思考：在实现 auth 的过程中我发现和 reg 的步骤较为相似，于是我想着把这两个操作的共同代码复用。结果代码搞得代码乌烟瘴气又恢复了回来。总结过后，对于代码复用我觉得需要考虑以下几点：

0. 复用的本质是减少重复代码。重复代码的坏处是：1. 一点修改，多处修改；2. 占用代码空间，降低代码可读性。
1. 复用是否对**以后**的代码修改有帮助。修改的原因可能是bug，需求变更，结构变更等。帮助就是不需要多处修改。如果代码复用正确，确实只需要改一次就可以了。但如果修改导致了原有复用代码不能再复用，而需要重新拆分设计，则复用就成了事倍功半的工作。像我复用 reg 和 auth 那样，由于对某些逻辑理解有误，导致原有复用代码又不得不拆开，能复用的越来越少，结果就是花拳绣腿，得不偿失。
2. 复用是否能提高代码阅读性。像上面那样，只是把2个代码段变成1个代码段，而且还要新建一个传入众多变量的函数，则没有必要了。

所以初期不做过于细致的代码优化是有道理的。

##### 4.3.1.6.2 dereg 注销实现

参考文档3.6.4.2：

FIDO UAF 客户端**必须**遵循以下步骤：

1. 选择主要版本为`1`，次要版本为`0`的消息。
2. 解析消息。
    * 如果`DeregistrationRequest`消息中的必填字段为空或某个字段的类型和值不符，则拒绝该操作。
    
    > 即把消息 m 转换成 DeregistrationRequest 对象：  
    
    ```
    public class OperationHeader {
        @Required public OperationHeader header; // 消息头
        @Required public DeregisterAuthenticator[] authenticators; // 待注销的authenticators
    ```
    
    > DeregisterAuthenticator.java：
    
    ```
    public class DeregisterAuthenticator {
        @Required public AAID aaid; // Authenticator唯一标记AAID
        @Required public String keyID; // 待删除的Authenticator的密钥对应的keyID
    }
    ```
    
3. 对于每个与消息版本`DeregistrationRequest.header.upv`一致的 Authenticator，并且该 Authenticator 包含一个`AAID`与提供的`AAID`之一相同：
    1. 为注销功能创建合适的`ASMRequest`并发送给 Authenticator。
    
    > dereg 不需要回复UAF Server，UAF Server默认注销成功。无论UAF Client注销是否成功，UAF Server都不会再使用该keyID。在注册，带用户名登陆认证，交易，注销时UAF Server都会指定相关keyID给UAF Client；在无用户名登陆认证的时候，UAF ASM和Authenticator会做自我检查（如果储存在它们那里），只保留最近注册的密钥，多余的密钥将会被删除。

#### 4.3.1.7 UAF\_OPERATION\_COMPLETION\_STATUS 命令实现

UAF\_OPERATION\_COMPLETION\_STATUS 用于在执行完 UAF\_OPERATION 之后，app将结果发回给 UAF Server，UAF Server 返回结果通过 UAF\_OPERATION\_COMPLETION\_STATUS 告知 UAF Client 处理的结果。目前在规范 1.0，该命令具有很大的自主性，规范没有明确说明需要你实现什么功能，只是提到了一点：UAF Client 执行完注册后，UAF Server 由于验签失败，或数据格式不对等原因拒绝了该条信息，则 UAF Client 需要把该条注册密钥给注销掉。具体做法就是通过 ASM 的 GetResitration 命令，把对应的 Authenticator 找到，然后执行 Deregister 命令。

#### 4.3.1.8 获取调用方的 componentName 

详见文档[TODO]()

#### 4.3.1.9 ASM Communicator 实现

ASM Communicator 的实现和 UAF Client Communicator 是比较相似的，因为它们都是通过 Intent的方式与 Acitivity/Service 进行通信。ASM Comunnitcator 同样是用 UAF Client Communitcator的方式进行封装，Activitiy 用新建 RequestActivity 的方式来解耦合。下面讲一下 ASM Communicator 不同的需求点：

1. ASM Communicator 在 UAF Client 的`DISCOVER`阶段需要遍历本设备上的所有ASM；
2. 在其他阶段，上层传入 AuthenticatorInfo，ASM Communicator 需要根据之前`DISCOVER`的结果来找到对应的Authenticator。


## 4.4 UAF ASM

本章节对应文档为fido-uaf-asm-api-v1.0-ps-20141208（中文：FIDO UAF 认证器特定模块API v1.0）。

ASM全称叫Authenticator-Specific Module，它的功能有几个：

1.  管理 Authenticator，包括连接，初始化等；
2.  为Authenticator提供密钥存储，查询和删除服务（如果Authenticator不自行存储密钥的话）；
3.  向 UAF Client 向上提供多个功能命令，将UAF Client json 格式的命令转换为 Authenticator 可识别的格式（规范 1.0 中为 TLV 格式）
4.  在无 username 的 auth 认证中，提供UI让用户选择已注册 username。

UAF ASM通信的数据皆为UTF-8编码的json数据。我自己写了一个 类似 gson 的 json 的 ORM 转换库，可以自动检测 @Required 域，但效果很一般。推荐还是使用 gson 进行转换。gson 可以自定义某些类的转换规则，也很好用。检测 @Required 域在转换完成后在外部利用反射进行检测即可。检测方法：[TODO]()

下面以规范文档中 ASM 的各个命令的处理步骤为基础，帮助理解实现思路。

在 4.3 中已经提到过的 ASM 支持的各个命令：

* GetInfo：获取该ASM当前可用的所有Authenticator的信息。信息中包括名称，版本，标识符AAID，以及其支持的特性。获取后可与这些Authenticator 进行通信。
* Register：指定向某个Authenticator发起注册请求（对应`reg`操作），Authenticator必须是GetInfo中获取到的。
* Authenticate：指定向某个Authenticator发起认证请求（对应`auth`操作），Authenticator必须是GetInfo中获取到的。
* Deregister：指定向某个Authenticator发起注销请求（对应`dereg`操作），Authenticator必须是GetInfo中获取到的。
* GetRegistrations：向ASM查询目前GetInfo中所有Authenticator的所有已注册keyID，以及keyID对应的appID。
* OpenSettings：要求某个Authenticator打开设置界面，该Authenticator需要声明其支持打开设置界面这个特性。

### 4.4.1 ASM的并发考虑

ASM 的通信设计整体和 UAF Client 是一样的，即拥有简单的并发能力。可直接参考4.3.1.1 和 4.3.1.2。

### 4.4.2 Authenticator Communitcator 设计

本节描述的是如何封装和建立与 Authenticator 的通信。

### 4.4.2 Authenticator 命令

按照惯例，在描述如何实现 ASM 的各个命令前，需要先告诉大家 Authenticator 支持什么功能命令。Authenticator 支持 4 种命令：

* GetInfo：Authenticator 也有GetInfo命令，返回的是 TLV格式的 `TAG_UAFV1_GETINFO_CMD_RESPONSE`数据。
* Register：对某个 username 进行注册，期间可能会发生认证，生成密钥，加解密，签名等操作。
* Sign：对某个 KeyHandle（可以理解为密钥信息） 进行签名，期间可能会发生认证，加解密，签名等操作。
* Deregister：对某个KeyHandle进行注销（删除）。删除不需要认证，但可能需要加解密。
* OpenSettings：打开 Authenticator 的设置界面，如果该 Authenticator 声明支持设置的话。

### 4.4.3 GetInfo 命令

`GetInfo`命令的实现很简单，ASM对所有可用的 Authenticator 发送 GetInfo 命令即可。然后收集所有 Authenticator 返回的数据，组成`GetInfoOut`：

```
public class GetInfoOut extends OutObject{
    @JsonName(name = "Authenticators")
    @Required public AuthenticatorInfo[] authenticators;

    public GetInfoOut(AuthenticatorInfo[] authenticators) {
        this.authenticators = authenticators;
    }

    private GetInfoOut() {

    }
}
```

AuthenticatorInfo.java：

```
public class AuthenticatorInfo implements Serializable {
    @Required public byte                 authenticatorIndex; // UAF Client 将用该 index 来指定与该 ASM 中的哪个 authenticator 通信
    @Required public Version[]            asmVersions; // 该 authenticator 支持的 ASM 的协议版本列表。ASM 的协议版本必须在这里面才可以使用该 authetnicator，以兼容版本
    @Required public boolean              isUserEnrolled; // 该 authenticator 是否已经有用户登记过了。比如是否登记过指纹／虹膜／面部等信息（如果没登记过就没法使用了）。和 reg 用户注册的概念不一样。
    @Required public boolean              hasSettings; // 该 authenticator 是否支持 OpenSettings 命令
    @Required public AAID                 aaid; // 该 authenticator 的 AAID
    @Required public String               assertionScheme; // 该 authenticator 的断言方案，目前规范 1.0 & 1.1 只有 UAFV1TLV 一种，也就是 TLV 格式
    @Required public short                authenticationAlgorithm; // 该 authenticator 的认证算法，ECDSA，RSA之类的
    @Required public short[]              attestationTypes; // 该 authenticator 的认证类型（Full预植证书/Surrogate自生成密钥）
    @Required public int                  userVerification; // 该 authenticator 使用了哪种认证方法（指纹，密码，掌纹等）
    @Required public short                keyProtection; // 密钥保护的方法（软件保护，硬件保护，TEE保护，SE保护，远程存储）
    @Required public short                matcherProtection; // 匹配器（就是用来认证的那个东西）保护方法（软件保护，TEE保护，芯片保护）
    @Required public int                  attachmentHint; // 该 authenticator 的连接方式，比如内部连接，有线，WIFI，蓝牙等连接方式
    @Required public boolean              isSecondFactorOnly; // 是否是 second factor authenticator
    @Required public boolean              isRoamingAuthenticator; // 是否是 roaming factor authenticator
    @Required public String[]             supportedExtensionIDs; // 支持的扩展ID（自定义扩展）
    @Required public short                tcDisplay; // 交易信息显示相关的特性
    public String                         tcDisplayContentType; // 交易信息的类型（图片，文字等）
    public DisplayPNGCharacteristicsDescriptor[] tcDisplayPNGCharacteristics; // 交易信息为图片时的各种参数
    public String                         title; // 该 authenticator 的名字
    public String                         description; // 该 authenticator 的描述
    public String                         icon; // 该 authenticator 图标（根据[RFC2397]将图片转换为 String，网上有工具）
    public byte[] userVerificationToken; // 如果 authenticator 支持 verification token，则提供此 token，token 将会在稍后进行注册时用到，如果使用了 verification token，就不需要再进行认证。
```

### 4.4.3.1 生成 authenticatorIndex

在 ASM 层面上，规范要求每个通过 Authenticator GetInfo 得到的`authenticatorIndex`都必须是唯一的，稳定的，不重复的。因为`authenticatorIndex`在 UAF Client 的操作中会反复用到，如果`authenticatorIndex`是重复的，或发生了变化，与 Authenticator 的通信就会出错。但 Authenticator 之间不会互相了解对方用了哪个`authenticatorIndex`，所以需要在 ASM 的层面上对所有的 Authenticator 分配`authenticatorIndex`。

最简单的办法就是写死了，为每一个 Authenticator 直接写死一个不重复的`authenticatorIndex`也基本没有副作用。因为 ASM 基本不会开放，只会为自己的 authenticator 服务。另一个方法是专门用一个类，用一个`HashMap<AAID, Byte>`记录每一个 Authenticator 的`authenticatorIndex`。

FIDO UAF 规范确实提到了你的 Authenticator 也可以和别的厂商的 ASM 配合工作。这个可能性在章节 4.6 中我在做了讨论。

### 4.3.3.2 AuthenticatorInfo的转换

Authenticator 的 GetInfo 返回的 TLV 数据如下：

```
public class GetInfoCommandResponse extends CommandResponse {
    @Tag public static final short TAG = CommandTag.TAG_UAFV1_GETINFO_CMD_RESPONSE;

    @TagMember(tag = AuthenticatorCommandTag.TAG_STATUS_CODE, order = 1)
    public short statusCode;

    /**
     * Authenticator 的 API 版本，目前规范 1.0 & 1.1 必须为 0x01
     */
    @TagMember(tag = AuthenticatorCommandTag.TAG_API_VERSION, order = 2)
    public byte apiVersion;

    /**
     * 可能有0个或多个
     */
    @TagMember(tag = AuthenticatorCommandTag.TAG_AUTHENTICATOR_INFO, order = 3)
    @Optional public List<> authenticatorInfo;
```

其中包含主要信息的是`TAG_AUTHENTICATOR_INFO`：

```
public class AuthenticatorInfo extends TagStructure {
    @Tag public static final short TAG = AuthenticatorCommandTag.TAG_AUTHENTICATOR_INFO;
    @TagMember(tag = AuthenticatorCommandTag.TAG_AUTHENTICATOR_INDEX, order = 1)
    public byte authenticatorIndex;

    @TagMember(tag = UAFProtocolTag.TAG_AAID, order = 2)
    public String aaid;

    @TagMember(order = 3)
    public AuthenticatorMetadata metadata;

    @TagMember(tag = AuthenticatorCommandTag.TAG_TC_DISPLAY_CONTENT_TYPE, order = 4)
    @Optional public byte[] tcDisplayContentType;

    @TagMember(order = 5)
    @Optional public TCDisplayPNGCharacteristics[] tcDisplayPNGCharacteristics;

    /**
     * currently in uaf 1.2 version, support "UAFV1TLV" only.
     */
    @TagMember(tag = AuthenticatorCommandTag.TAG_ASSERTION_SCHEME, order = 6)
    public String assertionScheme;

    @TagMember(tag = AuthenticatorCommandTag.TAG_ATTESTATION_TYPE, order = 7)
    public short[] attestationType;

    @TagMember(tag = AuthenticatorCommandTag.TAG_SUPPORTED_EXTENSION_ID, order = 8)
    @Optional public String[] supportedExtensionIDs;
```

注意：**这里 Authenticator 的`AuthenticatorInfo`和`GetInfoOut`的`AuthenticatorInfo`是两个数据类型（虽然类名字一样）。**通过比较，我们可以发现 Authenticator 的`AuthenticatorInfo`比`GetInfoOut`的`AuthenticatorInfo`少提供了几个成员。那么我们如何补充这些 Authenticator `GetInfo`不提供的成员呢？

* `asmVersions`：该 Authenticator 支持的 ASM 版本列表。这属于 Authenticator 的相关属性，理应从 Authenticator 层获取。解决方法：所有的 authenticator 提供一个新接口，提供支持的 ASM Versions；或更简便的，为所有的 Authenticator 统一设置一致的`asmVersions`，如果你的 ASM 和 Authenticator 都由你开发的话。实际使用中该变量暂时没有太大的意义。
* `attachmentHint`：Authenticator 的连接方式。作为 ASM 是肯定知道自己如何连接该 Authenticator 的，我们在设计 Authenticator Communitcator 的时候也考虑到了，直接填入，或从 Authenticator Communitcator 中获取即可。
* `title`，`description`，`icon`：这三个都是 authenticator 的相关概述。我觉得解决方法可以考虑以下两点：1. 我使用的方法是，拓展了Authenticator 的`GetInfo`命令，在其TLV格式后增加了三个TLV Tag，用于传递`title`，`description`，`icon`；2. 和`asmVersions`一样所有的 authenticator 提供一个新接口。


### 4.4.4 Register 命令

参考 FIDO ASM 规范 3.6.3。

查阅[UAFAuthnrCommand]文档可获得更多关于 TAGs 和本章节提及的架构的相关信息。

1. 使用`authenticatorIndex`来定位 Authenticator。如果 Authenticator 不能被定位，那么返回`UAF_ASM_STATUS_ERROR`。
    > 定位 Authenticator的方法：通过本章节 4.4.2 的 Authenticator Communicator 来获取对应`authenticatorIndex`的 Authenticator 的通讯接口，如果没有则返回错误。
    
2. 如果用户已经在 Authenticator 注册（注意这里及下面的注册都是 enroll，不是 register）过（比如，已采集生物特征、设置了 PIN），那么 ASM **必须**请求 Authenticator 验证该用户。  
    
    > 注释：
    > 
    > 如果 Authenticator 支持`UerVerificationToken`（请见文档[UAFAuthnrCommands]），那么 ASM 必须获取到该令牌以便在接下来的`Register`命令中包含它。
    * 如果校验失败，返回`UAF_ASM_STATUS_ACCESS_DENIED`。
    
    > 是否有用户 enroll 可根据`AuthenticatorInfo.isUserEnroll`来判断。
    >
    > 没有实现`UerVerificationToken`。`UerVerificationToken`是什么？就是在一次验证过后，一段时间内（由 Authenticator 设置）可以通过该 token 继续使用 Authenticator 的相关功能，而不用再次验证。`UerVerificationToken`是一个可选的实现，我没有实现。
    > 
    > 个人观点，`UerVerificationToken`可能会使得验证流程让使用者迷惑，有时需要验证有时又不需要。如果使用该功能，最好无需再次验证时有相关的文字提醒。
3. 如果用户没有在 Authenticator 注册，那么将用户指引到注册流程。
    * 如果注册失败，返回`UAF_ASM_STATUS_ACCESS_DENIED`
    
    > 该步我选择忽略，留给 Authenticator 处理（Authenticator 层也确实有 enroll 相关的处理），ASM 不应需要关注 Authenticator 的 enroll 流程，也没有规范的接口让其调用。如果你的 ASM 和 Authenticator 结合的很紧密（高耦合），那么在ASM层弹出 enroll 流程也可以。
4. 生成`KHAccessToken`。
    > `KHAccessToken`是一个 Authenticaor 用来验证 ASM 的机制，防止 Authenticator被非法使用。该 token 在注册时由 ASM 生成，并绑定到刚生成的密钥中。之后对该密钥的使用必须提供该 token，Authenticator 验证`KHAccessToken`合法后，才允许使用该密钥。
    >
    > 生成`KHAccessToken`的参数根据 Authenticator 类型有所区别。如果 Authenticator 将已加密的密钥存储在 ASM（bound authenticator），则`KHAccessToken`需要包含`AppID`，`ASMToken`，`PersonalID`和`CallerID`：
    > 
    > `AppID`：该密钥与`AppID`是绑定的，别的`AppID`用不了该密钥。AppID 即 UAF Client 传下来的`AppID`；
    > 
    > `ASMToken`：该密钥与 ASM 是绑定的，别的 ASM 用不了该密钥。`ASMToken`一般由 ASM 一次性生成并安全存储，之后一致沿用。
    > 
    > `PersonalID`：`PersonalID`代表当前操作系统的登陆用户。FIDO 设想的是，如果该系统是一个多用户系统，则用户 A 创建的密钥，用户 B 是无法使用的。但我不知道 Android 平台如何获取`PersonalID`（印象中 Android 貌似是支持多用户的？）。我在此处的做法也只是一次性生成了一个随机 ID 并存储，以后一直沿用。因为FIDO 也只是说通常包含这几个因素，并没有强制。
    > 
    > `CallerID`：即 UAF Client 的信息。对于 Android 来说就是 UAF Client apk 的签名信息，iOS 就是 bundle ID。则该密钥是与该 UAF Client 绑定的，其他 UAF Client 用不了该密钥。
    > 
    > 如果 Authenticator 的已加密密钥不存储在 ASM 中（roaming authenticator），则只与`AppID`进行绑定。
    >
    > 生成`KHAccessToken`的方法可以很简单，将所有参数拼接哈希是最快速的方法。
    
5. 使用特定 Authenticator 的哈希函数来哈希提供的`RegisterIn.finalChallenge`，得到`FinalChallengeHash`。 Authenticator 优选的哈希函数**必须**满足`AuthenticatorInfo.authenticationAlgorihtm`字段中定义的算法。
    > 根据`AuthenticatorInfo.authenticationAlgorihtm`来选择算法。除了`ALG_SIGN_SM2_SM3_RAW`使用的是 SM3 哈希算法，其他都是 SHA256 算法。
    
6. 创建一个`TAG_UAFV1_REGISTER_CMD`结构并传给 Authenticator。
    1. 复制`FinalChallengeHash`，`KHAccessToken`，`RegisterIn.Username`，`UserVerificationToken`，`RegisterIn.AppID`，`RegisterIn.AttestationType`。
        1. 根据`AuthenticatorType`，一些参数可能是可选的。参阅[UAFAuthnrCommands]获取更多的关于 Authenticator 类型和要求的参数的信息。
    
7. 调用命令，接收响应。
    > 通过 Authenticator Communicator 发送`TAG_UAFV1_REGISTER_CMD`给指定 Authenticator。
    
8. 解析`TAG_UAFV1_REGISTER_CMD_RESP`。
    * 解析`TAG_AUTHENTICATOR_ASSERTION`的内容（如：`TAG_UAFV1_REG_ASSERTION`）并提取`TAG_KEYID`。
9. 如果 Authenticator 是 bound Authenticator。
    * 将`CallerID`，`AppID`，`TAG_KEYHANDLE`，`TAG_KEYID`和`CurrentTimestamp`存储在 ASM 的数据库中。
    
    > 注释
    > 
    > 在该阶段 ASM 存储什么样的数据是由 Authenticator 的底层结构决定。比如，一些 Authenticator 可能会存储 AppID，KeyHandle，KeyID 在她们自己的安全存储中。在此种情况下，ASM 不需要在数据库中存储这些数据了。
    
    --
    > bound authenticator 需要存储这些数据，roaming authenticator 不需要，因为 roaming authenticator 不与 ASM 绑定，也就是说 roaming authenticator 可能通过其他的 ASM 进行操作产生新的数据，ASM 不应该依赖只经过自身的操作的数据。 
    
10. 创建一个`RegisterOut`对象。
    1. 根据`AuthenticatorInfo.assertionScheme`设定`RegisterOut.assertionScheme`。
    2. 以 base64url 格式编码`TAG_AUTHENTICATOR_ASSERTION`（比如`TAG_UAFV1_REG_ASERTION`）的内容，并设定为`RegisterOut.assertion`。
    3. 返回`RegisterOut`对象。 


### 4.4.5 Authenticate 命令

参考 FIDO ASM 规范 3.7.4。类似的步骤在 Sign 命令已有注释。

查阅[UAFAuthnrCommand]文档可获得更多关于 TAGs 和本章节提及的架构的相关信息：

1. 使用`authenticatorIndex`来定位 Authenticator。
2. 如果用户已经在 Authenticator 注册过（比如：生物登记法，PIN 设置），返回`UAF_ASM_STATUS_ACCESS_DENIED`。
3. ASM 必须请求 Authenticator 验证该用户，
    * 如果验证失败，返回`UAF_ASM_STATUS_ACCESS_DENIED`。
    
    > 注释：
    > 
    > 如果 Authenticator 支持`UerVerificationToken`（请见文档[UAFAuthnrCommands]），那么 ASM 必须包含该特征以便在接下来转到`Sign`命令。
4. 生成`KHAccessToken`。
5. 使用特定 Authenticator 的哈希函数来哈希提供的RegisterIn.finalChallenge，得到FinalChallengeHash。Authenticator 优选的哈希函数必须满足AuthenticatorInfo.authenticationAlgorihtm字段中定义的算法。
6. 如果这是 second factor authenticator 并且`AuthenticateIn.keyIDs`为空，那么返回`UAF_ASM_STATUS_ACCESS_DENIED`。
7. 如果`AuthenticateIn.keyIDs`不为空
    1. 如果是 bound authenticator，使用`AuthenticateIn.appID`和`AuthenticateIn.keyIDs`来查阅 ASM 数据库，获得相应的 KeyHandles。
        * 如果没有查到相关条目，返回`UAF_ASM_STATUS_ACCESS_DENIED`。
    2. 如果 roaming authenticator，那么就把`AuthenticateIn.keyIDs`看作 KeyHandles。

    > 这里有一个盲点，步骤 6 7 描述了`AuthenticateIn.keyIDs`不为空时的处理方法，和`AuthenticateIn.keyIDs`为空时 second factor authenticator 的处理方法，没有描述`AuthenticateIn.keyIDs`为空时 first factor authenticator 的处理方法。有些厂商会处理成只要`AuthenticateIn.keyIDs`为空就返回`UAF_ASM_STATUS_ACCESS_DENIED`错误码，这是不正确的。此处对应的处理方法在[UAFAuthnrCommands]的 7. KeyIDs and key handles中。
    >
    > [UAFAuthnrCommands]的 7. KeyIDs and key handles中描述了：`AuthenticateIn.keyIDs`为空时，对于first factor bound authenticator， ASM 应该把该 Authenticator 注册的所有加密 KeyHandles 取出传给它；对于 first factor roaming authenticator，ASM不需要做任何操作，该 Authenticator会把存储在自身上的 KeyHandles 全部取出来。
    >
    > `AuthenticateIn.keyIDs`为空意味着当前 app 没有用户记录（没有session），大家都不知道应该用哪个用户哪个 KeyHandle 登陆，所以就应该全部取出来。在接下来的步骤中会弹出 UI 界面进行选择。second factor authenticator 因为不支持 username，所以没办法进行 username 登陆选择，只能报错。second factor authenticator 不支持无 username 登陆。
    
8. 创建`TAG_UAFV1_SIGN_CMD`结构并把它传递给 Authenticator。
    1. 复制`AuthenticateIn.AppID`，`AuthenticateIn.Transaction.content`（如果不为空），`FinalChallengeHash`，`KHAccessToken`，`UserVerificationToken`，`KeyHandles`。
        * 根据AuthenticatorType，一些参数可能是可选的。参阅[UAFAuthnrCommands]获取更多的关于 Authenticator 类型和要求的参数的信息。
        * 如果提供了多个交易信息类型，ASM 必须选择一个最符合当前显示特性的。
        
        > 注释
        >
        > 举例来说，这可能取决于用户的设备在交易进行的那一刻是水平还是垂直放置的。
        
        * 在传递到 Authenticator 之前将 base64URL 编码的`AuthenticateIn.Tranaction.content`编码
    
    > 如何选择最合适的交易信息类型：UAF Server 可能会同时提供文字，图片（不同样式的图片）来作为交易报文信息。你可能需要通过`AuthenticatorInfo.tcDisplayContentType`和`AuthenticatorInfo.tcDisplayPNGCharacteristics`选择最佳显示的交易信息。
    
9. 调用命令，接收响应。
10. 解析`TAG_UAFV1_REGISTER_CMD_RESP`。
    * 如果是 first factor authenticator 并且响应含有`TAG_USERNAME_AND_KEYHANDLE`，然后：
        1. 从`TAG_USERNAME_AND_KEYHANDLE`字段中提取出用户名。
        2. 如果有两个相同的用户名，选取最近注册的那个。
        3. 显示剩下不同的用户名，让用户从中选择一个。
        4. 将`TAG_UAFV1_SIGN_CMD.KeyHandles`设置成和选取的用户名相关联的单独 KeyHandle。
        5. 转到步骤 8，发送新的`TAG_UAFV1_SIGN_CMD`命令。
    
    > 含有`TAG_USERNAME_AND_KEYHANDLE`的情况即无 username 登陆（无session）的情况。`TAG_USERNAME_AND_KEYHANDLE`包含的就是 authenticator 对所有 KeyHandles 过滤后剩下的 KeyHandles（通过KHAccessToken过滤）。而且`TAG_USERNAME_AND_KEYHANDLE`还包含了 KeyHandles 对应的 username。此时 ASM 将这些 username 通过 UI 展示给用户，让用户选择需要登录的 username。
    > 
    > 选择完之后，剩下的流程就变成正常的带 username 登录的情况了。重新发送只含有 1 个 KeyHandle 的 `TAG_UAFV1_SIGN_CMD` 命令。
    
11. 创建对象 AuthenticateOut。
    1. 将`AuthenticateOut.assertionScheme`设置为`AuthenticatorInfo.assertionScheme`。
    2. 将`TAG_AUTHENTICATOR_ASSERTION`的内容（如`TAG_UAFV1_AUTH_ASERTION`）以 base64url 形式编码，并设置为`AuthenticateOut.assertion`。
    3. 返回对象`AuthenticateOut`。

### 4.4.6 Deregister 命令

参考 FIDO ASM 规范 3.8.2。类似的步骤在 Sign 命令已有注释。

查阅[UAFAuthnrCommand]文档可获得更多关于 TAGs 和本章节提及的架构的相关信息：

1. 使用`authenticatorIndex`来定位 Authenticator。
2. 生成`KHAccessToken`。
3. 如果 Authenticator 为 bound authenticator，那么：
    * 在 ASM 数据库中查阅 Authenticator 的相关数据，删除与`DeregisterIn.appID`和`DeregisterIn.keyID`相关的记录。
4. 创建`TAG_UAFV1_DEREGITER_CMD`结构，复制`KHAccessToken`，`DeregisterIn.keyID`，并发送给 Authenticator。
    > 即使是 bound authenticator（数据都存储在 ASM），也是要把删除的相关指令再发一次给 Authenticator，以方便 Authenticator 做一些必要的操作。

### 4.4.7 GetRegistrations 命令

参考 FIDO ASM 规范 3.9.3。类似的步骤在 Sign 命令已有注释。

查阅[UAFAuthnrCommand]文档可获得更多关于 TAGs 和本章节提及的架构的相关信息：

1. 使用`authenticatorIndex`定位 Authenticator。
2. 如果 Authenticator 为 bound authenticator，那么：
    * 在 ASM 数据库中查询与 CallerID 和 AppID 相关的注册记录并且建立一个`AppRegistration`对象列表。
    
    > 注释
    > 
    > 一些 ASM 可能并没有将这些信息存储在自身数据库中，而是将器存储在 Authenticator 的安全存储空间。在这种情况下，ASM 必须发送一个适当的命令来获取相应的数据。
    
    --
    > 上面注释提到的不把注册信息存储在自身数据库中，其实就是面对 roaming authenticator的情况。对于 roaming authenticator，ASM 不自身存储，需要发送指令给它以取得`AppRegistration`数据。但是，FIDO UAF 没有为 UAF Authenticator 定义这样的指令。所以需要你自己创建一个指令或接口自行获取。我建议是尽量设计为 bound authenticator。如果你的 Authenticator 是游离于设备之外，拥有自己的安全存储空间的，就只能设计为 roaming authenticator，此时你就需要自己创建一个指令获取了。不过，如果你能保证该 roaming authenticator 只会通过一个 ASM 进行通信（通常也确实是这样），此处当作 bound authenticator 来处理也是可以的。
    
### 4.4.8 OpenSettings 命令

发送`TAG_UAFV1_OPEN_SETTINGS_CMD`给 Authenticator即可。如果 Authenticator 不支持打开设置，则无法打开。奇怪的是 FIDO UAF 要求该命令只能返回`UAF_ASM_STATUS_OK`，我这边没有按照这个规定，还是可能返回其他错误码。该项在测试中没有进行测试。

## 4.5 UAF Authenticator 

对应参考文档fido-uaf-authnr-metadata-v1.0-ps-20141208（中文FIDO UAF 认证器控制命令 v1.0）。

UAF Authenticator是用于认证用户的模块，它是提供验证的核心部分。UAF Authenticator 需要承担认证用户，生成密钥对，签名，加解密，展示交易信息等功能。

这里值得一提的是，UAF Authenticator 和 Authenticator 是不一样的概念。从`TAG_UAFV1_GETINFO_CMD_RESPONSE`我们可以看到，UAF Authenticator 返回的`GetInfo`信息中可能包含多个`TAG_AUTHENTICATOR_INFO`，也就是说一个 UAF Authenticator 是可能包含多个 Authenticator 的。

我在实现中把类`UAFAuthenticator`定义为接受 ASM 命令的通信层，把类`InternalAuthenticator`定义为实际工作的认证实现类，也就是`TAG_AUTHENTICATOR_INFO`所描述的 Authenticator。

Authenticator 可分为4类，first-factor（第一因子） Bound（绑定） Authenticator，second-factor（第二因子） Bound Authenticator，first-factor Roaming（漫游） Authenticator，second-factor Roaming Authenticator。first-factor 和 second-factor 区别在于是否有用户名这个概念。 bound 和 roaming 区别在于 Authenticator 是否与 ASM 绑定，绑定即该 Authenticator 只能通过该 ASM 访问。它们的组合会影响 key handle 的存储位置。

规范中说明，实现者不一定必须按照其分类特征实现（比如 Authenticator 可以有一点 first-factor 类特质，又有点 second-factor 类特质），只需要保证文档标明了必须规范实现的部分与文档保持一致即可。但最好还是按照分类的特征实现，因为这 4 种分类出现的频率很高，很多规范的描述都用到了这几种分类的概念。如果按自己想法实现的话，容易误解规范中的说明。

规范提到，如果你的 Authenticator要和其它实现者的 ASM 联合使用时，则必须遵守该文档中的所有描述。其实这样的情况是很少的，因为大家都更倾向于为自己的 Authenticator 单独提供一层 ASM，对外提供接口。

总的来说，还是尽量按照规范中描述来实现。

### 4.5.1 UAFAuthenticator 的封装

根据上面的思路，定义`UAFAuthenticator`类应实现的功能：

1. 初始化`InternalAuthenticator`，并持有他们的命令通信接口。
2. 解析从 ASM 层发过来的命令，并根据`authenticatorIndex`转发给对应的`InternalAuthenticator`。

初始化：约定好`InternalAuthenticator`的初始化接口即可。

持有命令通信接口：与 ASM 做法不同，`UAFAuthenticator`通过直接持有`InternalAuthenticator`实例进行通信，而没有另外声明一个通信接口，因为考虑到`UAFAuthenticator`和`InternalAuthenticator`的耦合要更紧密一些。

`UAFAuthenticator`会存储

ASM 层命令通过 Authenticator Communicator 与`UAFAuthenticator`通信，实际执行`UAFAuthenticator.execute()`进行命令处理：

```
// 主入口
public String execute(String cmd) {
    try {
        return executeTLVCommand(cmd);
    } catch (Exception e) {
        KLog.e("unknown error in UAFAuthenticator");
        KLog.e(UAFUtils.getStackTrace(e));
    }

    return null;
}

// 分开写是因为以后可能会出现新的数据格式
private String executeTLVCommand(String tlvCmd) {
    if (tlvCmd == null) {
        KLog.w("command null");
        return "";
    }
    byte[] tlvCmdBytes = UAFUtils.decode(tlvCmd);
    if (tlvCmdBytes == null || tlvCmdBytes.length < 2) {
        KLog.w("can not get tlv tags, command length: " + (tlvCmdBytes == null? "null" : tlvCmdBytes.length));
        return null;
    }

    // 获取 CMD 类型
    short tag = AuthenticatorUtils.bytesToShort(tlvCmdBytes);
    Class<? extends Command> commandType;
    switch (tag) {
        case CommandTag.TAG_UAFV1_GETINFO_CMD:
            // GetInfo 由 UAFAuthenticator 收集 InternalAuthenticator 信息，其他命令交由 InternalAuthenticator 执行
            return getInfo();
        case CommandTag.TAG_UAFV1_DEREGISTER_CMD:
            commandType = DeregisterCommand.class;
            break;
        case CommandTag.TAG_UAFV1_OPEN_SETTINGS_CMD:
            commandType = OpenSettingsCommand.class;
            break;
        case CommandTag.TAG_UAFV1_REGISTER_CMD:
            commandType = RegisterCommand.class;
            break;
        case CommandTag.TAG_UAFV1_SIGN_CMD:
            commandType = SignCommand.class;
            break;
        default:
            KLog.w("not a valid tag");
            return null;
    }

    // TLV 的 ORM 转换
    Command command = Command.parse(tlvCmd);
    if (command == null) {
        KLog.e(commandType.getSimpleName() + " parse failed");
        return returnSpecificResponse(StatusCode.UAF_CMD_STATUS_ERR_UNKNOWN, tag);
    }

    byte authenticatorIndex = command.getAuthenticatorIndex();
    // 获取实现对象
    InternalAuthenticator internalAuthenticator = getInternalAuthenticator(authenticatorIndex);
    if (internalAuthenticator == null) {
        KLog.e("can not find such authenticator with authenticator index " + authenticatorIndex);
        return returnSpecificResponse(StatusCode.UAF_CMD_STATUS_ERR_UNKNOWN, tag);
    }
    
    // 发送给指定的 InternalAuthenticator
    return internalAuthenticator.execute(command);
}
```

### 4.5.2 InternalAuthenticator 的封装

在 Authenticator 的实现中，我们把通用部分实现在父类，Authenticator 类别影响到的行为特征被实现为几个方法，默认根据类别判断，但子类可以进行重写。指令处理流程是通用流程，将实现在父类，`InternalAuthenticator`的特征差异（如哈希算法，非对称算法，KeyHandle 存储，签名，认证等）被抽象为接口，交由子类实现。

### 4.5.3 GetInfo 命令实现

`GetInfo`命令即由`UAFAuthenticator`收集`InternalAuthenticator`信息并返回。

### 4.5.4 Register 命令实现

参考规范 6.2.2.4：

1. 如果authenticator有交易确认展示界面可以展示AppID，则需要保证（ASM）提供了`Command.TAG_APPID`，并在用户确认的时候进行展示。使用`TAG_APPID`更新`Command.KHAccessToken`：
    * 把Command.KHAccessToken和Command.TAG_APPID混合，结果赋值给Command.KHAccessToken。举个混合例子（使用加密哈希）：
        * 例：Command.KHAccessToken=hash(Command.KHAccessToken | Command.TAG_APPID) （ | 表示拼接）
    
    > 注释
    > 
    > 这个方法允许我们避免在RawKeyHandle分开存储AppID。
    
    --
    > AppID 是可选域，但基本都会提供。如果 Authenticator 可以展示`AppID`，则`AppID`对于 Authenticator 是有识别意义的，Authenticator 的`KHAccessToken`是混合了`AppID`后的`KHAccessToken`，混合后将会存储在`RawKeyHandle`。如果不可以展示`AppID`，则`AppID`对于 Authenticator 没有意义，`KHAccessToken`不会混合`AppID`，Authenticator 将直接存储`KHAccessToken`。混合`AppID`后的`KHAccessToken`相当于存储了`AppID`，如果传入同样的`KHAccessToken`但`AppID`不同，则会校验失败。

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


### 4.5.5 Sign 命令实现

参考规范 6.2.3.4：

> 注释
> 
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
    
    > 注释
    >
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
        > 注释
        > 
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
    > 
    > Authenticator**不可以**不先验证用户而去处理Sign command  
    > Authenticator**不可以**不先验证用户而展示username  
    > Bound authenticators**不可以**不先验证KHAccessToken而先去处理Sign command  
    > UAuth.priv密钥在解析为明文的时候，**必须**留存在Authenticator的安全区域。UAuth.priv保护边界声明在[[UAFAuthnrMetadata](#bib-UAFAuthnrMetadata)]的`Metadata.keyProtection`中。  
    > 如果Authenticator的Metadata表示其支持“交易确认显示”（Transaction Confirmation display）——则其**必须**显示提供的交易信息并在TAG_UAFV1_SIGNED_DATA加入交易信息的哈希值  
    > Silent Authenticators**不可以**工作在first-factor模式，以遵循[[FIDOSecRef](#bib-FIDOSecRef)]中的假设  
    > 如果Authenticator不支持`SignCounter`，则在TAG_UAFV1_SIGNED_DAT中**必须**设置`SignCounter`为0。当Authenticator恢复出厂设置时`SignCounter`必须设置为0，以遵循[[FIDOSecRef](#bib-FIDOSecRef)]中的假设  
    >
    > 一些Authenticators可能支持“交易确认显示”功能，但不是在Authenticator内，而是在ASM的边界中。典型例子是基于软件的“交易确认显示”。当处理提供了交易的Sign command的时候，这些Authenticators应该假设他们的确有一个内置的“交易确认显示”，并讲交易信息的哈希值包括在最后的断言中，但并不显示东西给用户。不仅如此，这些Authenticator的Metadata文件**必须**明确显示交易确认显示”的类型。典型例子是“交易确认显示”的flag为TRANSACTION_CONFIRMATION_DISPLAY_ANY或TRANSACTION_CONFIRMATION_DISPLAY_PRIVILEGED_SOFTWARE。查看[[UAFRegistry](#bib-UAFRegistry)]了解“交易确认显示”的flag类型。

### 4.5.6 Deregister 命令实现

参考规范 6.2.4.4：

1. 如果authenticator有交易确认展示界面可以展示AppID，则需要保证（ASM）提供了`Command.TAG_APPID`，并在用户确认的时候进行展示。使用`TAG_APPID`更新`Command.KHAccessToken`：
    * 把Command.KHAccessToken和Command.TAG_APPID混合，结果赋值给Command.KHAccessToken。举个混合例子（使用加密哈希）：
        * 例：Command.KHAccessToken=hash(Command.KHAccessToken | Command.TAG_APPID)
    
    > NOTE
    >
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

### 4.5.7 OpenSettings 命令实现

我没有实现 OpenSettings，因为我们实现的 Fingerprint Authenticator 没办法直接打开设置（或者我没找到办法）。实现该功能没有什么难点，启动对应的设置 Activity 后返回成功即可。

### 4.5.8 实现一个 Fingerprint Authenticator

上面的章节只是描述了实现各个命令的流程，留下了验证用户，签名，加解密，生成密钥对，存储密钥对等抽象接口，等待具体的`InternalAuthenticator`实现类来实现这些方法。本章节将描述如何使用指纹认证，Android Key Store System 和数据库等工具来实现这些方法。

在 Android 设备上实现 Fingerprint认证，需要打开系统的锁屏，并添加指纹。

#### 4.5.8.1 Fingerprint 调用与UI展示

该部分我使用了 Android 官网上的指纹认证demo，并进行了精简。 demo 下载地址：[https://developer.android.com/samples/AsymmetricFingerprintDialog/project.html](https://developer.android.com/samples/AsymmetricFingerprintDialog/project.html)

该 demo 是提供了密码和指纹认证的认证方式，当指纹不可用时（比如错误次数过多），可以使用密码认证。我为了精简demo，把密码认证的部分去掉了。然后我进一步去掉了一些入口，让 demo 成为了通过 startActivityForResult 启动，通过 Intent 来接收参数，验证结果通过 setResult(Intent intent) 返回这样的一个验证模块。

提示：指纹验证成功后，所有该 apk 生成的验证后方可使用的私钥都将可用，而非只有在 FingerprintManager.authenticate() 时传入的 CryptoObject 对象对应的私钥可用。传入的CryptoObject对象仅是方便在回调时取出进行签名，可直接设为 null。另外一点是，用于非对称加解密的私钥无法封装为 CryptoObject，因为在验证指纹前取出其 PrivateKey 对象时会报错。而用于签名的私钥对象却可以在认证前取得（若未验证会在签名时报错）。两者行为不一致，不知道是不是 bug。这个问题一度让我以为没办法实现非对称加解密。

#### 4.5.8.2 Counters 存储实现

Counters 存储使用了阿里的聚安全 JAQ SDK，使用起来非常简单，安全性没有研究，大概是建表的白盒加密的方案，且和 APK 绑定。使用 JAQ SDK 的原因是，其支持 Android 2.2 以上的设备，如果用到 Android KeyStore 加密 Counters 后存储，则需要 Android 4.X 以上的设备。使用安全加密可以防止有 root 权限的设备篡改 Counters。

简易使用代码如下：

```
SecurityStorage securityStorage = new SecurityStorage(context);
// 存储
securityStorage.putString(signCounterName, Integer.toString(signCounter))
securityStorage.putString(signCounterName, Integer.toString(regCounter))

// 获取
Integer.parseInt(securityStorage.getString(regCounterName));
Integer.parseInt(securityStorage.getString(signCounterName));
```

SecurityStorage 只支持 String 类存储。SecurityStorage 使用时需要创建新对象，内部管理初始化的问题。而非使用常见的单例模式。这样规避了单例模式持有 context 容易导致内存泄露的问题，我感觉这样设计确实更好用。

JAQ SDK 下载地址：[https://jaq.alibaba.com/gc/](https://jaq.alibaba.com/gc/)

#### 4.5.8.3 KeyHandles 存储实现

KeyHandles 使用 SQLite 存储。建表结构为：

```
db.execSQL("CREATE TABLE " + TABLE_NAME + " ("
                + "ID            INTEGER AUTO INCREMENT PRIMARY KEY,"
                + "key_id        TEXT    NOT NULL,"
                + "key_handle    TEXT    NOT NULL,"
                + "generate_time INT8 NOT NULL);")
```

generate_time 注意要使用 INT8，使用 INTEGER （32 bit）存储时间戳会溢出。不使用 TIME 的原因是不好处理。

#### 4.5.8.4 对称加解密实现

这里我实现了两套方案：

1. 用阿里的聚安全 JAQ SDK 实现，很简单，但不需要验证即可使用，且所有的设备共享同一个对称密钥；
2. 自生成对称加解密的密钥，然后使用 Android KeyStore 的非对称密钥进行加密，验证后方可加解密。加密后的密钥存储在 JAQ SDK 的 SecurityStorage 中。

实际我使用的是第二种方案，安全性更高。

## 4.6. 一种开放式 ASM 的设计方案设想

ASM--UAF Authenticator 开放通信方案

文档中没有说明 ASM 和 UAF Authenticator 是如何通信的，但有提到需要保证信息完整。现设计需求如下：

1. 有收发确认（如 tcp ）
2. 有错误检查（如CRC ）
3. 注册方法：
	1. ASM 主动寻找 UAF Authenticator （ Intent-filter，或其它驱动），注册时进行握手，UAF Authenticator 提供识别信息（可能是内置非对称公钥，发送 ASM 进行验签）。（ ASM 主动寻找 UAF Authenticator 将不支持热拔插）
	2. UAF Authenticator 主动寻找 ASM，ASM 开放某个端口，UAF Authenticator 发送识别信息。然后 UAF Authenticator 和 ASM 建立 tcp （或 udp，广播）连接，
	 udp 广播好处是，不需要一直维持连接。且可以共用端口。
	 tcp 好处是，不需要自定义收发确认
4. 通信方法：1.有网络功能的 UAF Authenticator，握手后使用 tcp （或 udp ）通信；2.对于没有网络功能的 UAF Authenticator（蓝牙，音频等物理设备），ASM 内置驱动。

# 5. FIDO UAF iOS 客户端实现方案

// 预计5月补充
