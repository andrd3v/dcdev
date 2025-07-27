переработка старого доклада по dcdevice 

# Глава 1

## ```DeviceCheck.framework```


при создании токена из дсдевайс используется ```[DCDevice generateTokenWithCompletionHandler:]``` - внутри себя она вызывает ```DCDeviceMetadataDaemonConnection```

этот метод создает соеднинение с демоном айфона devicecheckd
```objc
NSXPCConnection *xpc_connection = (NSXPCConnection *)objc_msgSend(
                              objc_alloc((Class)&OBJC_CLASS___NSXPCConnection),
                              "initWithMachServiceName:options:",
                              CFSTR("com.apple.devicecheckd"),
                              0LL);
```


## ```devicecheckd```


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



## Разберем последовательно, что происодит в методах классов DCContext, DCDDeviceMetadata, DCCryptoProxyImpl при их вызове в `-[DCClientHandler fetchOpaqueBlobWithCompletion]`


## ```DeviceCheckInternal.framework```


Первое, что происходит в `-[DCClientHandler fetchOpaqueBlobWithCompletion]` это инициализация DCContext и присваивание ему наш ```<TeamID>.<BundleIdentifier>``` или ```<BundleIdentifier>```:

```objc
DCContext_class = objc_alloc_init((Class)&OBJC_CLASS___DCContext);
objc_msgSend(DCContext_class, "setClientAppID:", app_id);
```

Так как у DCContext нет метода `-init`, то он просто наследует реализацию метода от NSObject. Выделяется память под объект и выставляется указатель на класс DCContext. Поля, например `_clientAppID` при этом обнуляются. Далее посылается объекту DCContext селектор `-init`. Поскольку DCContext не переопределяет этот метод, в дело вступает стандартный `-[NSObject init]`, который просто возвращает self без дополнительной логики. После вызывается метод `-[DCContext setClientAppID:]`, который выставляет в `self->_clientAppID` наш ```<TeamID>.<BundleIdentifier>``` или ```<BundleIdentifier>```:
```objc
id __cdecl __noreturn -[DCContext clientAppID](DCContext *self, SEL a2)
{
  return objc_getProperty_33(self, a2, 8LL, 1);
}
```


Пропускаем аллокацию `DCDDeviceMetadata` и просматриваем `DCCryptoProxyImpl = objc_alloc_init((Class)&OBJC_CLASS___DCCryptoProxyImpl);` - аналогично, просто выделяется память под обьект, выставляется указатель на класс DCCryptoProxyImpl и вызывается `-[NSObject init]`.


А вот теперь инициализируется наш `DCDDeviceMetadata`, который был просто аллоцирован. 
```objc
init_DCDDeviceMetadata = objc_msgSend(
                           DCDDeviceMetadata,
                           "initWithContext:cryptoProxy:",
                           DCContext_class, // наш класс, который хранит <TeamID>.<BundleIdentifier> или <BundleIdentifier>
                           DCCryptoProxyImpl); 
```



Что же происходит при ините `DCDDeviceMetadata`:
```objc

// мы на самом деле работаем с областью памяти экземпляра, а не с C‑структурой как таковой
00000000 struct DCDDeviceMetadata // sizeof=0x18
00000000 {
00000000     unsigned __int8 superclass_opaque[8];
00000008     DCCryptoProxy *_cryptoProxy;
00000010     DCContext *_context;
00000018 };

id -[DCDDeviceMetadata initWithContext:cryptoProxy:](DCDDeviceMetadata *self, SEL a2, id DCContext_class_arg, id DCCryptoProxyImpl_class_arg)
{
  DCDDeviceMetadata *dc_device_metadata = -[DCDDeviceMetadata init](self, "init"); // -[NSObject init]

  if ( dc_device_metadata )
  {
    // настраиваем поля из DCDDeviceMetadata
    j__objc_storeStrong((id *)&dc_device_metadata->_cryptoProxy, DCCryptoProxyImpl_class_arg);
    j__objc_storeStrong((id *)&dc_device_metadata->_context, DCContext_class_arg);  // наш контекст с нашим <TeamID>.<BundleIdentifier> или <BundleIdentifier>
  }

  return (id *)dc_device_metadata;
}
```


Отлично, все классы были проинициализированны и теперь демон ```devicecheckd``` вызывает ```objc_msgSend(init_DCDDeviceMetadata, "generateEncryptedBlobWithCompletion:", v4);``` - метод, который начнет геренацию токена.


# Глава 2

 
## ```DeviceCheckInternal.framework```
***важная ремарка, почти все методы будут переработаны мной, для того, чтобы их было легко читать и понимать***


Наш демон вызвал метод ```objc_msgSend(init_DCDDeviceMetadata, "generateEncryptedBlobWithCompletion:", v4);``` - это начало создания токена.


```objc
void __cdecl -[DCDDeviceMetadata generateEncryptedBlobWithCompletion:](DCDDeviceMetadata *self, SEL a2, id completion_arg)
{
  id v4 = objc_retain(completion_arg); // держим ретейн ссылку комплетиона

  // как раз таки, что было настроенно в -[DCDDeviceMetadata initWithContext:cryptoProxy:]
  DCCryptoProxy *cryptoProxy = self->_cryptoProxy;
  DCContext *context = self->_context; // наш контекст с нашими <TeamID>.<BundleIdentifier> или <BundleIdentifier>

  [cryptoProxy fetchOpaqueBlobWithContext:context
                              completion:^(NSData *blob, NSError *error) {
      // этот блок соответствует __57__DCDDeviceMetadata_generateEncryptedBlobWithCompletion___block_invoke
      // берём указатель на исходный XPC‑блок‑completion, сохранённый при вызове.
      // если аргумент a2 (данные) не нулевой, вызывается completion(data, nil).
      // если a2 равен 0, создаётся NSError с кодом 0 и вызывается completion(nil, error).
  }];
}
```

Отлично, вызывается метод `-[DCCryptoProxy fetchOpaqueBlobWithContext:completion:]` в который передается наш контекст с нашими ```<TeamID>.<BundleIdentifier>``` или `<BundleIdentifier>`, ну а также комплетион. Ида плохо декомпилирует код, поэтому он будет чуть переписан.

```objc
void __cdecl -[DCCryptoProxyImpl fetchOpaqueBlobWithContext:completion:](
        DCCryptoProxyImpl *self,
        SEL a2,
        id DCContext_argDCContext_arg,
        id completion_arg)
{
  // держим контекст(<TeamID>.<BundleIdentifier> или <BundleIdentifier>) и комплетион
  id retainedContext = [DCContext_argDCContext_arg retain];
  void (^copiedCompletion)(NSData *, NSError *) = [completion_arg copy];

  if (os_log_type_enabled(self.logger, OS_LOG_TYPE_DEFAULT))
  {
    os_log(self.logger, "Generating certificate...");
  }

  __block id blockContext = retainedContext;
  __block void (^blockCompletion)(NSData *, NSError *) = copiedCompletion;

    [self _fetchPublicKey:^(NSData *publicKey)
    {
        DCCertificateGenerator *generator = [[DCCertificateGenerator alloc]
            initWithContext:blockContext
                   publicKey:publicKey];

        [generator generateEncryptedCertificateChainWithCompletion:
                                  ^(NSData *encryptedChain, NSError *error)
        {
            blockCompletion(encryptedChain, error);
            [blockCompletion release];
            [blockContext release];
            [generator release];
        }];
    }];
}
```

