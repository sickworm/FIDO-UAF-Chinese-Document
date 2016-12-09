实现UAF协议的时候，Client需要获取调用者的信息（获得其APK的签名）。用中文查了半天没查到获取Activity的方法，用英文一下就搜出来了（主要还是看英文累脑子）

Activity：

`getCallingActivity()`
`getCallingPackage()`

注意：
只有调用者使用的是`startActivityForResult()`时才可以获取，`startActivity()`返回的是null。

Service：

`Binder.getCallingUid()`
`Binder.getCallingPid()`

注意：
当AIDL使用的是oneway（异步）声明时，`Binder.getCallingPid()`返回的是0 。

获取到Uid之后，使用`getPackageManager().getPackagesForUid(uid)`获取到对应的包名。如果多个apk使用了shareUserId的话，返回值将会是多个包，这时候就没办法知道具体是哪个package调用的了。不过使用shareUserId的前提是使用相同的签名文件签名，而UAF要求是得到调用者apk签名的hash，这样的话哪个package调用结果都一样了。

> 版权所有，转载请注明出处：
> [https://github.com/SickWorm/FIDO-UAF-Chinese-Document/](https://github.com/SickWorm/FIDO-UAF-Chinese-Document/)
