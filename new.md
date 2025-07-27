переработка старого доклада по dcdevice 


```DeviceCheck.framework```


при создании токена из дсдевайс используется ```[DCDevice generateTokenWithCompletionHandler:]``` - внутри себя она вызывает ```DCDeviceMetadataDaemonConnection```

этот метод создает соеднинение с демоном айфона devicecheckd
```objc
NSXPCConnection *xpc_connection = (NSXPCConnection *)objc_msgSend(
                              objc_alloc((Class)&OBJC_CLASS___NSXPCConnection),
                              "initWithMachServiceName:options:",
                              CFSTR("com.apple.devicecheckd"),
                              0LL);
```


```devicecheckd```


После подключения к демону ```devicecheckd``` по xpc запускается нижеописанная цепочка.


При получении соедниения демон вызвает ```-[DCXPCListener listener:shouldAcceptNewConnection:]``` -> ```-[DCClientHandler initWithConnection:]```, а после вызова RPC от клиента - > ```-[DCClientHandler fetchOpaqueBlobWithCompletion]```


во первых  ```-[DCClientHandler fetchOpaqueBlobWithCompletion]``` вызывает ```if ( -[DCClientHandler _isSupported](self, "_isSupported") ) ```: значение этой переменной жестко закодирвоанно в ```DeviceIdentityIsSupported``` из ```DeviceIdentity.framework```.
```objc
__int64 DeviceIdentityIsSupported_1()
{
  return 1LL;
}
```
***предположу, что если дсдевайс недоступен на девайсе, то там будет другая сборка фреймворка и там будет закодирован 0***


после проверки поддрежки вызывается ```-[DCClientHandler _generateAppIDFromCurrentConnection]``` - этот метод получает ```<TeamID>.<BundleIdentifier>``` (формат: ```ABCDE12345.com.example.myApp```) с помощью entitlements (```-[NSXPCConnection valueForEntitlement:]```), а если же этот вариант не сработает, то тогда с помощью ```SecTaskCopyTeamIdentifier``` и ```SecTaskCopySigningIdentifier```; если team_id валидный и не "0000000000", то метод объединяет через точку team_id и bundle_id, иначе использует только bundle_id. Возвращает appID (```<TeamID>.<BundleIdentifier>``` или ```<BundleIdentifier>```): ```return [appID length] ? appID : nil;```.


Вернемся в `-[DCClientHandler fetchOpaqueBlobWithCompletion]` (Важное уточнение: я не буду рассматривать фэлл-бек`и, когда код идет в else, где обрабатываются ошибки.)
```objc
if ( app_id )
{
  DCContext_class = objc_alloc_init((Class)&OBJC_CLASS___DCContext);
  objc_msgSend(DCContext_class, "setClientAppID:", app_id);  // выставляем в классе наш "<TeamID>.<BundleIdentifier>"

  // аллоцируем DCDDeviceMetadata и инициализируем DCCryptoProxyImpl из DeviceCheckInternal.framework
  DCDDeviceMetadata = objc_alloc((Class)&OBJC_CLASS___DCDDeviceMetadata);
  DCCryptoProxyImpl = objc_alloc_init((Class)&OBJC_CLASS___DCCryptoProxyImpl);

  // инициализиурем DCDDeviceMetadata из DeviceCheckInternal.framework
  init_DCDDeviceMetadata = objc_msgSend(
                             DCDDeviceMetadata,
                             "initWithContext:cryptoProxy:",
                             DCContext_class,
                             DCCryptoProxyImpl);

  objc_msgSend(init_DCDDeviceMetadata, "generateEncryptedBlobWithCompletion:", v4);
}
```


Небольшой итог: Демон ```devicecheckd``` возвращает в ```DeviceCheck.framework``` зашифрованный токен (opaque blob) по XPC в блок ```completionHandler```, переданный через ```-[DCDDeviceMetadata generateEncryptedBlobWithCompletion:]```. Этот blob в дальнейшем используется как параметр token в ```-[DCDDeviceMetadata generateTokenWithCompletionHandler:]```.



** Разберем последовательно, что происодит в методах классов DCContext_class, DCDDeviceMetadata, DCCryptoProxyImpl при их вызове в `-[DCClientHandler fetchOpaqueBlobWithCompletion]` **


```DeviceCheckInternal.framework```
