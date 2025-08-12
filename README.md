### Работу выполнил [andrd3v](https://github.com/andrd3v). Помощь предоставил [whoeevee](https://github.com/whoeevee). Спасибо, что прочли!


переработка моего старого доклада по dcdevice 

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
              initWithContext:blockContext // наш контекст с нашими <TeamID>.<BundleIdentifier> или <BundleIdentifier>
                     publicKey:publicKey];
  
          [generator generateEncryptedCertificateChainWithCompletion:
                                    ^(NSData *encryptedChain, NSError *error)
            {
                blockCompletion(encryptedChain, error);
                [blockCompletion release];
                [blockContext release];
                [generator release];
            }
          ];
      }
    ];
}
```

### Разберемся с получением publicKey


Сначала получается `publicKey`, а потом он передается дальше. Как же мы получаем паблик ключ:
```objc
void __cdecl -[DCCryptoProxyImpl _fetchPublicKey:](DCCryptoProxyImpl *self, SEL a2, id complation)
{
  /*
  void *v4; // x20
  id v5; // x19
  _QWORD v6[4]; // [xsp+8h] [xbp-38h] BYREF
  id v7; // [xsp+28h] [xbp-18h]

  v4 = (void *)objc_claimAutoreleasedReturnValue_5(+[DCAssetFetcher sharedFetcher](&OBJC_CLASS___DCAssetFetcher, "sharedFetcher"));
  v6[0] = _NSConcreteStackBlock_ptr;
  v6[1] = 3221225472LL;
  v6[2] = __37__DCCryptoProxyImpl__fetchPublicKey___block_invoke;
  v6[3] = &unk_20A9B2838;
  objc_msgSend(v4, "fetchPublicKeyAssetWithCompletion:", v6);
  */

  DCAssetFetcher *fetcher = [DCAssetFetcher sharedFetcher];
  [fetcher fetchPublicKeyAssetWithCompletion:^(NSData *publicKey) {
      completion(publicKey);
  }];
}
```


Этот метод вызывает `-[DCAssetFetcher fetchPublicKeyAssetWithCompletion:]`.
```objc
void __cdecl -[DCAssetFetcher fetchPublicKeyAssetWithCompletion:](DCAssetFetcher *self, SEL a2, id publicKeyCompletion)
{
  /*
  DCAssetFetcherContext *v5 = objc_alloc_init(&OBJC_CLASS___DCAssetFetcherContext);
  id v4 = objc_retain(publicKeyCompletion);

  -[DCAssetFetcherContext setAllowCatalogRefresh:](v5, "setAllowCatalogRefresh:", 0LL);
  -[DCAssetFetcher _fetchAssetWithContext:completionHandler:](self, "_fetchAssetWithContext:completionHandler:", v5, v4);
  objc_release(v4);
  objc_release(v5);
  */

  DCAssetFetcherContext *context = [[DCAssetFetcherContext alloc] init];
  void (^completionBlock)(NSData *) = [publicKeyCompletion retain];

  [context setAllowCatalogRefresh:NO]; // self->_allowCatalogRefresh = NO;
  [self _fetchAssetWithContext:context
        completionHandler:completionBlock];
}
```

Пока что не комментирую, продолжаем идти по цепочке.
```objc
void __cdecl  -[DCAssetFetcher _fetchAssetWithContext:completionHandler:](
        DCAssetFetcher *self,
        SEL a2,
        (DCAssetFetcherContext *)context, // другой контекст, без тим и бандл айди
        id (void (^)(NSData *assetData, NSError *error))completion)
{
    DCAssetFetcherContext *retainedContext = [context retain];
    void (^completionBlock)(NSData *, NSError *) = [completion retain];

    if (os_log_type_enabled(self.logger, OS_LOG_TYPE_DEFAULT)) {
        os_log(self.logger, "Querying...");
    }

    [self _queryMetadataWithContext:retainedContext
                         completion:completionBlock];

    [completionBlock release];
    [retainedContext release];
}
```


Начинается самое интересное - `_queryMetadataWithContext`.
```objc
void __cdecl __noreturn -[DCAssetFetcher _queryMetadataWithContext:completion:](
        DCAssetFetcher *self,
        SEL a2,
        (DCAssetFetcherContext *)context, // другой контекст, без тим и бандл айди
        completion:(void (^)(NSData *assetData, NSError *error))completion)
{
    DCAssetFetcherContext *retainedContext = [context retain];
    void (^completionBlock)(NSData *, NSError *) = [completion retain];

    if (os_log_type_enabled(self.logger, OS_LOG_TYPE_DEFAULT))
    {
        os_log(self.logger,
               "Starting to fetch asset with context: %@",
               retainedContext);
    }

    id assetQuery = [[self _assetQuery] retain]; // claimAutoreleasedReturnValue
    NSUInteger resultCode = [assetQuery queryMetaDataSync]; // это все уже из libobjc.A

    // Ветка пропуска кэша или отсутствия (ignoreCachedMetadata || resultCode == 2)
    if ([retainedContext ignoreCachedMetadata] || resultCode == 2)
    {
        [self _handleMissingMetadataWithContext:retainedContext
                                   completion:completionBlock];
    } else {
      if (resultCode != 0)
      {
          // Ошибка: генерируем NSError и сразу возвращаем
          NSError *error = [NSError errorWithDomain:@"com.apple.twobit.fetcherror"
                                               code:0xFFFF_FFFF_FFFF_F448
                                           userInfo:nil];
          completionBlock(nil, error);
          [assetQuery release];
          return;
      }

            // Успех: передаём данные вверх
      [self _handleSuccessForQuery:assetQuery
                              completion:completionBlock];
    }

    [assetQuery release];
    [completionBlock release];
    [retainedContext release];
}
```

```objc
- (void)_handleMissingMetadataWithContext:(DCAssetFetchContext *)context
                               completion:(void (^)(DCAsset *asset, NSError *error))completion {
    NSLog(@"[DCAssetFetcher] Query sync result indicated missing asset catalog");

    if (context.allowCatalogRefresh) {
        context.allowCatalogRefresh = NO;
        context.ignoreCachedMetadata = NO;

        // запускаем обновление ассет-каталога и повторим запрос после завершения
        [self->_assetQuery refreshCatalogWithCompletion:^{
            [self _queryMetadataWithContext:context completion:completion];
        }];
        return;
    }

    NSError *error = [NSError errorWithDomain:@"com.apple.twobit.fetcherror"
                                         code:-3101
                                     userInfo:nil];
    completion(nil, error);
}
```

```objc
- (void)_handleSuccessForQuery:(DCAssetQuery *)query
                    completion:(void (^)(DCAsset *asset, NSError *error))completion {
    NSArray *results = [query results];

    if (results.count == 0) {
        NSError *error = [NSError errorWithDomain:@"com.apple.twobit.fetcherror"
                                             code:-3100
                                         userInfo:nil];
        completion(nil, error);
        return;
    }

    if (results.count > 1) {
        NSLog(@"[DCAssetFetcher] Warning: more than one asset found, using first one");
    }

    NSDictionary *mobileAsset = results.firstObject;

    NSError *validationError = nil;
    DCAsset *asset = [self _validateAsset:mobileAsset error:&validationError];

    if (!asset) {
        completion(nil, validationError);
        return;
    }

    [[DCXPCActivityController sharedInstance] updateActivityScheduleWithAsset:asset];
    completion(asset, nil);
}
```


```objc
- (DCAsset *)_validateAsset:(NSDictionary *)mobileAsset error:(NSError **)error {
    DCAsset *asset = [DCAsset assetWithMobileAsset:mobileAsset];
    if (!asset) {
        if (error) {
            *error = [NSError errorWithDomain:@"com.apple.twobit.fetcherror"
                                         code:-3200
                                     userInfo:nil];
        }
        return nil;
    }
    return asset;
}
```

```objc
+ (DCAsset *)assetWithMobileAsset:(NSDictionary *)mobileAsset {
    NSNumber *version = mobileAsset[@"com.apple.MobileAsset.AssetVersion"];
    if (![version isKindOfClass:[NSNumber class]] || version.integerValue != 1) {
        NSLog(@"[DCAsset] Unknown asset version: %@", version);
        return nil;
    }

    NSData *pubKeyData = mobileAsset[@"com.apple.devicecheck.pubvalue"]; //assetProperty:
    if (![pubKeyData isKindOfClass:[NSData class]] || pubKeyData.length == 0) {
        NSLog(@"[DCAsset] No public key found in asset");
        return nil;
    }

    DCAsset *asset = [[DCAsset alloc] init];
    asset.version = 1;
    asset.publicKey = pubKeyData;

    NSNumber *refreshInterval = mobileAsset[@"com.apple.devicecheck.refreshtimer"]; //assetProperty:
    if ([refreshInterval isKindOfClass:[NSNumber class]]) {
        asset.publicKeyRefreshInterval = refreshInterval.doubleValue;
    }

    return asset;
}
```

```objc
- (void)updateActivityScheduleWithAsset:(DCAsset *)asset {
    if (asset.publicKeyRefreshInterval <= 0) return;

    NSDictionary *criteria = @{
        XPC_ACTIVITY_INTERVAL : @(asset.publicKeyRefreshInterval),
        XPC_ACTIVITY_REPEATING : @YES,
        XPC_ACTIVITY_REQUIRE_NETWORK : @YES
    };

    xpc_activity_register("com.apple.devicecheck.notify", criteria, ^(xpc_activity_t activity) {
        [[DCAssetFetcher sharedInstance] performMetadataRefreshForActivity];
    });
}
```


и самое смешное - это все не нужно и это неверный путь, правильный ключ лежит в ```__37__DCCryptoProxyImpl__fetchPublicKey___block_invoke```
```objc
+[NSData dataWithBytes:length:](&OBJC_CLASS___NSData, "dataWithBytes:length:", &fallback_server_pubkey, 65LL);
```

он жестко закодирвоан и одинаковый на всех версиях мак ос и иос и айпад ос:
`0450d934fa67bcf6f2dfbf96629e0a7238e9205d75f28cfcd84f35a6592bbe058a9c0f8edbca2acb67efb774971ca45f7d856a694fb1b9c40b94fb2e7a5a9498b0`
это 130 байтов ключа-сам ключ 65 символов
```
╭─    ~ ······························································································································· ✔  at 12:04:28 
╰─ unhex 0450d934fa67bcf6f2dfbf96629e0a7238e9205d75f28cfcd84f35a6592bbe058a9c0f8edbca2acb67efb774971ca45f7d856a694fb1b9c40b94fb2e7a5a9498b0
P�4�g���߿�b�
r8� ]u���O5�Y+������*�g�t��_}�jiO���
                                    ��.zZ���%
```

вот и наш паблик ключик.



# Глава 3
## ```DeviceCheckInternal.framework```
***важная ремарка, почти все методы будут переработаны мной, для того, чтобы их было легко читать и понимать***


вернемся в fetchOpaqueBlobWithContext

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

    [self _fetchPublicKey:^(NSData *publicKey) // мы получили наш ключ
      {
          DCCertificateGenerator *generator = [[DCCertificateGenerator alloc]
              initWithContext:blockContext // наш контекст с нашими <TeamID>.<BundleIdentifier> или <BundleIdentifier>
                     publicKey:publicKey]; // наш ключ
  
          [generator generateEncryptedCertificateChainWithCompletion:
                                    ^(NSData *encryptedChain, NSError *error)
            {
                blockCompletion(encryptedChain, error);
                [blockCompletion release];
                [blockContext release];
                [generator release];
            }
          ];
      }
    ];
}
```


разберемся с инициализацией `generator`.
```objc

00000000 struct DCCertificateGenerator // sizeof=0x18
00000000 {
00000000     unsigned __int8 superclass_opaque[8];
00000008     NSData *_publicKey;
00000010     DCContext *_context;
00000018 };

id __cdecl -[DCCertificateGenerator initWithContext:publicKey:](DCCertificateGenerator *self, SEL a2, id context_arg, id publicKey_arg)
{
  DCCertificateGenerator *v9 = -[DCCertificateGenerator init](self, "init"); // -[NSObject init]
  if ( v9 )
  {
    j__objc_storeStrong((id *)&v9->_publicKey, publicKey_arg); // наш полученный паблик_ключ
    j__objc_storeStrong((id *)&v9->_context, context_arg); // наш контекст с нашими <TeamID>.<BundleIdentifier> или <BundleIdentifier>
  }
  return (id *)v9;
}
```


после успешной инициализации вызывается generateEncryptedCertificateChainWithCompletion - создание токена
```objc
void __cdecl -[DCCertificateGenerator generateEncryptedCertificateChainWithCompletion:](
        DCCertificateGenerator *self,
        SEL a2,
        id a3)
{
  id v3; // x20
  _QWORD v4[4]; // [xsp+0h] [xbp-40h] BYREF
  DCCertificateGenerator *v5; // [xsp+20h] [xbp-20h]
  id v6; // [xsp+28h] [xbp-18h]

  v4[0] = _NSConcreteStackBlock_ptr;
  v4[1] = 3221225472LL;
  v4[2] = __74__DCCertificateGenerator_generateEncryptedCertificateChainWithCompletion___block_invoke;
  v4[3] = &unk_20A9B2860;
  v5 = self;
  v6 = objc_retain(a3);
  v3 = objc_retain(v6);
  -[DCCertificateGenerator _generateCertificateChainWithCompletion:](v5, "_generateCertificateChainWithCompletion:", v4);
  objc_release(v6);
  objc_release(v3);
}
```


Первое-надо получить серт: `_generateCertificateChainWithCompletion`, а дальше передать его в `__74__DCCertificateGenerator_generateEncryptedCertificateChainWithCompletion___block_invoke` - тут он передастся в _encryptData, там где и создается токен

короче все это - получение двух корневых сертификатов из кейчейн- в теории это можно отреверсить или повторить логику(в чем я супер сомневаюсь, потому что там кейчейн), но получение серта идет в __DeviceIdentityIssueClientCertificateWithCompletion_block_invoke

вот примеры сертификатов из аргумента
```
-----BEGIN CERTIFICATE-----
MIIDPjCCAuWgAwIBAgIGAZhghIx7MAoGCCqGSM49BAMCMFMxJzAlBgNVBAMMHkJh
c2ljIEF0dGVzdGF0aW9uIFVzZXIgU3ViIENBMTETMBEGA1UECgwKQXBwbGUgSW5j
LjETMBEGA1UECAwKQ2FsaWZvcm5pYTAeFw0yNTA3MzAxMjQ1NTZaFw0yNjA3MTQy
MDE2NTZaMIGRMUkwRwYDVQQDDEA5ZjE2ZTNiNDU3OWY3Y2Q4MzBjMWFjNTNhM2Y2
MDk2MmFlZWIyMDgzMTgwNzI4Y2UzNTMyNmM2Y2JhZDdlOTIzMRowGAYDVQQLDBFC
QUEgQ2VydGlmaWNhdGlvbjETMBEGA1UECgwKQXBwbGUgSW5jLjETMBEGA1UECAwK
Q2FsaWZvcm5pYTBZMBMGByqGSM49AgEGCCqGSM49AwEHA0IABOY3WyNGuRO+wcrp
t2Yb6ARssM0g5GFCy302nZQ3p/DPxR3cG4wqLK73zYEiKjUXU1Uv0bgbV61CmTSv
Pkd0KmejggFkMIIBYDAMBgNVHRMBAf8EAjAAMA4GA1UdDwEB/wQEAwIE8DCCAT4G
CSqGSIb3Y2QKAQSCAS8EggErMYIBJ/+EmqGSUA0wCxYEQ0hJUAIDAIAV/4SqjZJE
ETAPFgRFQ0lEAgcKTVwAaAAu/4aTtcJjGzAZFgRibWFjBBFkNDphMzozZDozNDo2
YTowMP+Gy7XKaRkwFxYEaW1laQQPMzU5NDA0MDgyMDM4NzA1/4ebydxtFjAUFgRz
cm5tBAxGSzJWTUREVUpDTDj/h6uR0mQyMDAWBHVkaWQEKDU1MzcwZjUyYWQwOWRi
YmEyYjZhYTcwZDM4ZjZmMjc5NzExYmYxNmX/h7u1wmMbMBkWBHdtYWMEEWQ0OmEz
OjNkOjM0OjY5OmRm/4ebldJkOjA4FgRzZWlkBDAwNDI0MzEyQjFFNDM4MDAxNzIx
OTE0MTgyNTk0Mjg0NDdCRjZFMzJCQzNCN0YwQ0YwCgYIKoZIzj0EAwIDRwAwRAIg
LNjRS4pu3925qkoourOxUM3L+8hwVYbT/35bIW9q5KQCICf4Xpvuvdx/DMBbr5Wp
9GXn0bxP69/8S99Z9x+t0ow0
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
MIICIzCCAaigAwIBAgIIeNjhG9tnDGgwCgYIKoZIzj0EAwIwUzEnMCUGA1UEAwwe
QmFzaWMgQXR0ZXN0YXRpb24gVXNlciBSb290IENBMRMwEQYDVQQKDApBcHBsZSBJ
bmMuMRMwEQYDVQQIDApDYWxpZm9ybmlhMB4XDTE3MDQyMDAwNDIwMFoXDTMyMDMy
MjAwMDAwMFowUzEnMCUGA1UEAwweQmFzaWMgQXR0ZXN0YXRpb24gVXNlciBTdWIg
Q0ExMRMwEQYDVQQKDApBcHBsZSBJbmMuMRMwEQYDVQQIDApDYWxpZm9ybmlhMFkw
EwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEoSZ/1t9eBAEVp5a8PrXacmbGb8zNC1X3
StLI9YO6Y0CL7blHmSGmjGWTwD4Q+i0J2BY3+bPHTGRyA9jGB3MSbaNmMGQwEgYD
VR0TAQH/BAgwBgEB/wIBADAfBgNVHSMEGDAWgBSD5aMhnrB0w/lhkP2XTiMQdqSj
8jAdBgNVHQ4EFgQU5mWf1DYLTXUdQ9xmOH/uqeNSD80wDgYDVR0PAQH/BAQDAgEG
MAoGCCqGSM49BAMCA2kAMGYCMQC3M360LLtJS60Z9q3vVjJxMgMcFQ1roGTUcKqv
W+4hJ4CeJjySXTgq6IEHn/yWab4CMQCm5NnK6SOSK+AqWum9lL87W3E6AA1f2TvJ
/hgok/34jr93nhS87tOQNdxDS8zyiqw=
-----END CERTIFICATE-----
```

или

```
-----BEGIN CERTIFICATE-----
MIIDYTCCAwegAwIBAgIGAZY9utjzMAoGCCqGSM49BAMCMFMxJzAlBgNVBAMMHkJh
c2ljIEF0dGVzdGF0aW9uIFVzZXIgU3ViIE5BMTETMBEGA1UECgwKQXBwbGUgSW5j
LjETMBEGA1UECAwKQ2FsaWZvcm5pYTAeFw0yNTA0MTUwODMyNTdaFw0yNjA0MTAx
NTAwNTdaMIGRMUkwRwYDVQQDDEA1ODk5YWNjNTY4YWI2YjJiOThjYzdmMmRjMTYy
YWJlNTkzZjJlMDM0YjJlNDAyMjA2Y2MzOWMwMmFkNTQzNzcxMRowGAYDVQQLDBFC
QUEgQ2VydGlmaWNhdGlvbjETMBEGA1UECgwKQXBwbGUgSW5jLjETMBEGA1UECAwK
Q2FsaWZvcm5pYTBZMBMGByqGSM49AgEGCCqGSM49AwEHA0IABEfuSX2/VaqQkTcU
M3wLH7/5zIkiO/oJIsW08lKB5jGhL+v+pcqsYqt9yQsbxPS3axgjbqoCOcP/V3zY
Fy8kDm82jggGGMIIBgjAMBggqhkjOPQQDAgUAA0gAMEUCIHLplLqirOgMmrPkMSQa
DOpl/MAyEYejw/otUrfGGITiAiEAuLs1MYHGWuUdhyofXfY0S45GsSYXA/g8ombH
MkcU54A=
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
MIICIzCCAaigAwIBAgIIeNjhG9tnDGgwCgYIKoZIzj0EAwIwUzEnMCUGA1UEAwwe
QmFzaWMgQXR0ZXN0YXRpb24gVXNlciBSb290IENBMRMwEQYDVQQKDApBcHBsZSBJ
bmMuMRMwEQYDVQQIDApDYWxpZm9ybmlhMB4XDTE3MDQyMDA0MjAwMFoXDTMyMDMy
MDAwMDAwMFowUzEnMCUGA1UEAwweQmFzaWMgQXR0ZXN0YXRpb24gVXNlciBSb290
IENBMTETMBEGA1UECgwKQXBwbGUgSW5jLjETMBEGA1UECAwKQ2FsaWZvcm5pYTBZ
MBMGByqGSM49AgEGCCqGSM49AwEHA0gAMEUCIHLplLqirOgMmrPkMSQaDOpl/MAy
EYejw/otUrfGGITiAiEAuLs1MYHGWuUdhyofXfY0S45GsSYXA/g8ombHMkcU54A=
-----END CERTIFICATE-----
```

пропускаем все эти моменты и идем в _encryptData - создание токена


создадим токен
```objc
- (NSData *)_encryptData:(NSData *)data //серты
        serverSyncedDate:(NSDate *)serverDate 
                    error:(NSError **)error {
    // Log start of encryption
    os_log_t log = os_log_create("com.apple.devicecheck", "DeviceCheck");
    if (os_log_type_enabled(log, OS_LOG_TYPE_DEFAULT)) {
        os_log(log, "Encrypting data...");
    }

    // Get client App ID as UTF-8 data
    NSString *clientAppID = [self.context clientAppID];
    NSData *clientAppData = [clientAppID dataUsingEncoding:NSUTF8StringEncoding];
    NSUInteger clientAppLen = clientAppData.length;
    const void *clientAppBytes = clientAppData.bytes;

    // Prepare input data
    NSUInteger inputLen = data.length;
    const void *inputBytes = data.bytes;

    // Current timestamp from server-synced date
    NSTimeInterval timestamp = [serverDate timeIntervalSince1970];

    // AES-GCM mode descriptor from CommonCrypto
    const struct ccmode_gcm *gcm = ccaes_gcm_encrypt_mode();

    // Allocate buffers: output and plaintext payload
    size_t payloadLen = inputLen + clientAppLen + 81;
    size_t outputLen = inputLen + clientAppLen + 235;
    uint8_t *outBuf = calloc(1, outputLen);
    if (!outBuf) {
        if (error) *error = [NSError errorWithDomain:@"DeviceCheckError"
                                                code:-1 
                                            userInfo:nil];
        return nil;
    }
    uint8_t *payloadBuf = calloc(1, payloadLen);
    if (!payloadBuf) {
        free(outBuf);
        if (error) *error = [NSError errorWithDomain:@"DeviceCheckError"
                                                code:-1 
                                            userInfo:nil];
        return nil;
    }

    // Write header/type (value 2) and payload length in outBuf
    *((uint32_t *)outBuf) = 2;
    *((uint32_t *)(outBuf + 150)) = (uint32_t)payloadLen;

    // Copy device's static public key into outBuf at offset 5
    NSData *devicePubKeyData = self.publicKey;  // NSData containing the public key bytes
    memcpy(outBuf + 5, devicePubKeyData.bytes, devicePubKeyData.length);

    // Assemble payload: [timestamp (8 bytes), inputLen (4 bytes), clientAppLen (4 bytes), inputBytes, clientAppBytes]
    *((uint32_t *)(payloadBuf + 73)) = (uint32_t)inputLen;
    memcpy(payloadBuf + 81, inputBytes, inputLen);
    *((uint32_t *)(payloadBuf + 77)) = (uint32_t)clientAppLen;
    memcpy(payloadBuf + 81 + inputLen, clientAppBytes, clientAppLen);
    *((uint64_t *)(payloadBuf + 65)) = (uint64_t)timestamp;

    // Log system call (no-op with 0) as seen in disassembly
    DCLogSystem(0);

    // Create ephemeral ECDH key using the keybag
    id keybag = [self keybagHandle];
    uint64_t refKey = 0;
    int aksErr = aks_ref_key_create((__int64)keybag, 11, 4, 0, 0, &refKey);
    if (aksErr != 0) {
        DCLogSystem(aksErr);
        free(outBuf);
        free(payloadBuf);
        if (error) *error = [NSError errorWithDomain:@"DeviceCheckError"
                                                code:aksErr 
                                            userInfo:nil];
        return nil;
    }

    // Get ephemeral public key (should be 65 bytes) and copy it
    size_t ecdhPubLen = 0;
    const uint8_t *ecdhPub = (const uint8_t *)aks_ref_key_get_public_key(refKey, &ecdhPubLen);
    if (ecdhPubLen != 65) {
        DCLogSystem(ecdhPubLen);
        free(outBuf);
        free(payloadBuf);
        if (error) *error = [NSError errorWithDomain:@"DeviceCheckError"
                                                code:-2 
                                            userInfo:nil];
        return nil;
    }
    memcpy(outBuf + 85, ecdhPub, ecdhPubLen);

    // Compute ECDH shared secret with device's static public key
    const uint8_t *devicePubBytes = devicePubKeyData.bytes;
    size_t devicePubLen = devicePubKeyData.length;
    int ecdhErr = aks_ref_key_compute_key(refKey, 0, 0, (__int64)devicePubBytes, devicePubLen);
    if (ecdhErr != 0) {
        DCLogSystem(ecdhErr);
        free(outBuf);
        free(payloadBuf);
        if (error) *error = [NSError errorWithDomain:@"DeviceCheckError"
                                                code:ecdhErr 
                                            userInfo:nil];
        return nil;
    }

    // The shared secret is at refKey; skip first 2 bytes (per disassembly analysis)
    uint8_t *sharedSecret = (uint8_t *)refKey + 2;
    size_t sharedLen = devicePubLen - 2;

    // Derive key material with HKDF-SHA256 (44 bytes: 32-byte key + 12-byte IV)
    uint8_t hkdfOut[44];
    cchkdf(ccsha256_di(), sharedLen, sharedSecret, 0, NULL, 0x2C /*44 bytes*/, hkdfOut);
    uint8_t *aesKey = hkdfOut;          // first 32 bytes
    uint8_t *aesIV = hkdfOut + 32;      // next 12 bytes

    // Encrypt payloadBuf with AES-GCM; tag-> outBuf+1, ciphertext-> outBuf+154
    int gcmErr = ccgcm_one_shot(gcm,
                                32, aesKey,
                                12, aesIV,
                                0, NULL,
                                payloadLen, payloadBuf,
                                outBuf + 154,
                                16, outBuf + 1);
    if (gcmErr != 0) {
        DCLogSystem(gcmErr);
        free(outBuf);
        free(payloadBuf);
        if (error) *error = [NSError errorWithDomain:@"DeviceCheckError"
                                                code:gcmErr 
                                            userInfo:nil];
        return nil;
    }

    // Build NSData for the encrypted payload
    NSData *encryptedData = [NSData dataWithBytes:outBuf length:outputLen];

    // Log base64 payload if logging is enabled
    if (os_log_type_enabled(log, OS_LOG_TYPE_DEFAULT)) {
        NSData *b64 = [encryptedData base64EncodedDataWithOptions:0];
        NSString *payloadStr = [[NSString alloc] initWithData:b64
                                                     encoding:NSUTF8StringEncoding];
        os_log(log, "\nPayload (base64):\n%{public}s\n\n", payloadStr.UTF8String);
    }

    free(outBuf);
    free(payloadBuf);
    return encryptedData;
}
```

# Глава 4. Итог

В этом репозитории я попытался показать полный путь генерации `DeviceCheck`-токена: от вызова `DCDevice` в приложении, через XPC и демон devicecheckd, до финального шифрования пейлоада в `_encryptData`. Разобраны ключевые этапы — инициализация контекста с TeamID.BundleID, получение публичного ключа (включая fallback-константу), сборка цепочки сертификатов и формирование зашифрованного blob, отправляемого на сервер.

Надеюсь этот доклад сделал чуточку понятнее, почему девайсчек такая фиготень. В теори это можно обойти(как минимум юзать свои полученные серты), но мне лень это делать. 


## Работу выполнил [andrd3v](https://github.com/andrd3v). Помощь предоставил [whoeevee](https://github.com/whoeevee). Спасибо, что прочли!

