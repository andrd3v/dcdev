# dcdev
**📅 25 июня 2025**


при создании токена дсдевайс используется ```[DCDevice generateTokenWithCompletionHandler:]```
внутри себя она вызывает ```DCDeviceMetadataDaemonConnection```

этот метод создает соеднинение с демоном айфона devicecheckd
```
v4 = (NSXPCConnection *)objc_msgSend(
                              objc_alloc((Class)&OBJC_CLASS___NSXPCConnection),
                              "initWithMachServiceName:options:",
                              CFSTR("com.apple.devicecheckd"),
                              0LL);
```


Перемещаемся в devicecheckd

При получении соедниения он вызвает ```DCClientHandler initWithConnection:``` - > ```DCClientHandler fetchOpaqueBlobWithCompletion``` в своем листенере ```DCXPCListener listener:shouldAcceptNewConnection:```
во первых этот метод вызывает 
```if ( -[DCClientHandler _isSupported](self, "_isSupported") ) ```
значение этой переменной жестко закодирвоанно в ```DeviceIdentityIsSupported``` из приватного фреймворка
```
__int64 DeviceIdentityIsSupported_1()
{
  return 1LL;
}
```

***предположу, что если дсдевайс недоступен на девайсе, то там будет другая сборка фреймворка и там будет закодирован 0***

после проверки поддрежки вызывается ```DCClientHandler _generateAppIDFromCurrentConnection```
этот метод получает

```
team_id_and_bundle = objc_claimAutoreleasedReturnValue(
    -[DCClientHandler _stringValueForEntitlement:]
        (self, "_stringValueForEntitlement:",
         CFSTR("application-identifier")));
// ABCDE12345.com.example.myApp
// "<TeamID>.<BundleIdentifier>"
```
если так не получилось получить бандлайди то идем по фаллбеку
```
if (![team_id_and_bundle length]) {
    // entitlement пустое → fallback

    CFStringRef team_id   = SecTaskCopyTeamIdentifier(task,   NULL);
    CFStringRef bundle_id = SecTaskCopySigningIdentifier(task, NULL);
}
```

Если team_id валидный и не "0000000000", объединяет через точку, иначе использует только bundle_id
```
if (team_id && [team_id length] && ![team_id isEqualToString:@"0000000000"]) {
    appID = [NSString stringWithFormat:@"%@.%@", team_id, bundle_id];
} else {
    appID = bundle_id;
}
```
вернет строку которая получилась в итоге, если она не пустая 
```return [appID length] ? appID : nil;```

**📅 26 июня 2025**

В `DCClientHandler fetchOpaqueBlobWithCompletion` вызывается
   ```objc
   [DCDDeviceMetadata initWithContext:cryptoProxy:…];


   DCContext_class = objc_alloc_init((Class)&OBJC_CLASS___DCContext); // тут нельзя хукнуть методы так как они из фреймворка интернал 
   objc_msgSend(DCContext_class, "setClientAppID:", app_id); // выставляем в классе наш "<TeamID>.<BundleIdentifier>"
   DCDDeviceMetadata = objc_alloc((Class)&OBJC_CLASS___DCDDeviceMetadata);
   DCCryptoProxyImpl = objc_alloc_init((Class)&OBJC_CLASS___DCCryptoProxyImpl);
   // создаем два вспомогательных класса
   v11 = objc_msgSend(DCDDeviceMetadata, "initWithContext:cryptoProxy:", DCContext_class, DCCryptoProxyImpl);
   // передаем наши классы в DeviceCheckInternal
```

итак, вернемся к вызову `DCDDeviceMetadata generateEncryptedBlobWithCompletion:` в демоне(не в приваетфреймворке!)
после инициализации паблик ключа в `DCDDeviceMetadata initWithContext:cryptoProxy:`
вызывается из приват фреймворка```objc_msgSend(init_DCDDeviceMetadata, "generateEncryptedBlobWithCompletion:", v4);``` <- `в демоне!`

`generateEncryptedBlobWithCompletion` инициирует асинхронную генерацию «зашифрованного блоба» (token), проксируя запрос к криптографическому слою (DCCryptoProxy) и передавая клиентский блок-обработчик.

```
- (void)generateEncryptedBlobWithCompletion:
        (void (^)(NSData *encryptedBlob, NSError *error))completion
{
    void (^cb)(NSData*,NSError*) = [completion retain];
    DCCryptoProxy *cryptoProxy = self->_cryptoProxy;
    DCContext      *context     = self->_context;

    //    __57__DCDDeviceMetadata_generateEncryptedBlobWithCompletion___block_invoke // <- !!!!!!!
    void (^innerBlock)(NSData *rawChain, NSError *error) = ^(NSData *rawChain, NSError *error) {
        if (rawChain) {
            completion(rawChain, nil);
        } else {
            NSError *err = [NSError dc_errorWithCode:0];
            completion(nil, err);
            [err release];
        }
    };

    [cryptoProxy fetchOpaqueBlobWithContext:context
                                completion:innerBlock]; // DCCryptoProxyImpl

    [cb release];
}
```



`DCCryptoProxyImpl` → `DCCertificateGenerator`

```DCCryptoProxyImpl``` фактически лишь запускает подсистему логирования/сигнализации (через _DCLogSystem_0) и не содержит остальной логики прямо в этом месте. Весь «мозг» перенесён в блок

```objc
void __noreturn __59__DCCryptoProxyImpl_fetchOpaqueBlobWithContext_completion___block_invoke(
        int64_t a1, void *publicKey)
{
    // 1) Захват publicKey
    DCCertificateGenerator *gen = [[DCCertificateGenerator alloc]
        initWithContext:(DCContext *)*(uint64_t *)(a1 + 32)
               publicKey:(id)objc_retain(publicKey)];

    // 2) Генерация цепочки и шифрование
    [gen generateEncryptedCertificateChainWithCompletion:
        (void (^)(NSData *blob, NSError *error))*(uint64_t *)(a1 + 40)];

    objc_release(gen);
}
```


## Методы `DCCertificateGenerator`

```objc
- (instancetype)initWithContext:(DCContext *)context
                     publicKey:(id)publicKey;
- (void)generateEncryptedCertificateChainWithCompletion:
         (void (^)(NSData *encryptedBlob, NSError *error))completion;
```

## Первый создает паблик ключ
```
id __cdecl -[DCCertificateGenerator initWithContext:publicKey:](...)
{
    // 1. Ретейним переданные параметры
    _publicKey = [publicKey retain];
    _context   = [context retain];
    // 2. Вызываем базовый init
    self = [super init];
    return self;
}
```

Сохраняет в инстанс поля:
```self->_publicKey``` – публичный ключ приложения
```self->_context``` – объект DCContext с параметрами сессии

## а далее второй метод в вызове в метадате generateEncryptedBlobWithCompletion инициирует асинхронную генерацию «сырых» (нешифрованных) сертификатов, передавая внутреннему методу _generateCertificateChainWithCompletion: свой блок-обработчик.

```objc
void __cdecl -[DCCertificateGenerator generateEncryptedCertificateChainWithCompletion:](...)
{
    // 1. Ретейним callback-блок
    id completion = [a3 retain];

    // 2. Создаём стэк-блок __74__…_block_invoke
    void (^block)(NSData *rawChain, NSDate *serverDate) = ^(NSData *rawChain, NSDate *serverDate) {
        // этот код описан ниже
    };

    // 3. Передаём блок в приватный метод генерации сырой цепочки
    [self _generateCertificateChainWithCompletion:block];

    // 4. Освобождаем блок
    [completion release];
}
```


далее в `[DCCertificateGenerator generateEncryptedCertificateChainWithCompletion:]` генериурется сертификат с помощью блок инвойса 

```objc
void __fastcall __74__…block_invoke(int64_t block_ptr,
                                     int64_t rawChain,      // a2
                                     int64_t serverDate)     // a3
{
    if (rawChain) {
        // 1. Получаем self (зашифровщик) из 
        void *encryptor = *(void **)(block_ptr + 32);

        // 2. Вызываем приватный метод шифрования:
        //    - rawChain        : NSData *
        //    - serverDate      : NSDate *
        //    - &errorPointer   : NSError **
        NSError *error = nil;
        NSData *encryptedBlob = [encryptor _encryptData:rawChain
                                     serverSyncedDate:serverDate
                                                 error:&error]; // что тут происходит описано ниже

        // 3. Вызываем исходный completion(encryptedBlob, error)
        completion(encryptedBlob, error);

        // 4. Освобождаем временные объекты
        [encryptedBlob release];
        [error release];
    } else {
        // 1. Если сырой chain отсутствует — формируем ошибку
        NSError *error = [NSError errorWithDomain:@"com.apple.devicecheck.cryptoerror"
                                             code:0
                                         userInfo:nil];

        // 2. Вызываем completion(nil, error)
        completion(nil, error);

        // 3. Освобождаем ошибку
        [error release];
    }
}
```


как же получается блоб 
`        NSData *encryptedBlob = [encryptor _encryptData:rawChain
                                     serverSyncedDate:serverDate
                                                 error:&error];  
`

Основной путь:
Сериализация данных → `CBOR`
Эфемерный `ECDH` → `shared secret`
`HKDF` → симметричный ключ
`AES-GCM` шифрование
Сборка финального `blob-а`


вся это функция ассинхронная и раскидана по `колдам`
вот ее восстановленный вариант
```objc
- (NSData *)_encryptData:(NSData *)rawChain
        serverSyncedDate:(NSDate *)serverDate
                    error:(NSError **)outError
{
    // 1. Проверяем входные данные
    if (rawChain == nil) {
        *outError = [NSError dc_errorWithCode:0];
        return nil;
    }

    // 2. Извлекаем публичный ключ из self->_publicKey
    NSData *pubKeyData = self->_publicKey;  // ожид. 65 байт
    if (pubKeyData.length != 65) {
        *outError = [NSError errorWithDomain:@"com.apple.devicecheck.cryptoerror" code:0 userInfo:nil];
        return nil;
    }

    // 3. Формируем CBOR-контекст:
    //    { "chain": rawChain.bytes,
    //      "date": serverDate.timeIntervalSince1970,
    //      "appID": context.appID,
    //      "entitlements": context.entitlements,
    //      … }
    NSMutableData *cborPayload = CBOREncode({
        "raw_chain": rawChain,
        "server_date": @(serverDate.timeIntervalSince1970),
        "app_id": self->_context.appID, // вот тут хранится наш тим айди и бандл
        "entitlements": self->_context.entitlementsDict // и тут лежат енты приложения
    });

    // 4. Генерируем ephemeral ключ (EC P-256)
    ECKeyPair *ephemeral = ECCreateKeyPair(curve: P256);

    // 5. Выполняем ECDH между ephemeral.private и pubKeyData
    NSData *sharedSecret = ECDH(ephemeral.privateKey, pubKeyData);

    // 6. Из sharedSecret и serverDate деривим симметричный ключ
    NSData *symKey = HKDF(sharedSecret,
                          salt: serverDate.timeIntervalSince1970,
                          info: "DeviceCheck",
                          length: 32);

    // 7. Шифруем CBOR-пакет AES-GCM
    NSData *iv         = RandomBytes(12);
    (NSData *cipherText, NSData *tag) = AESGCM_Encrypt(symKey, iv, cborPayload);

    // 8. Финальный формат (например, CBOR-массив):
    //    [ ephemeral.publicKey || iv || tag || cipherText ]
    NSMutableData *outBlob = CBOREncodeArray(@[
        ephemeral.publicKeyData,
        iv,
        tag,
        cipherText
    ]);

    return outBlob;
}
```

В DCClientHandler, в случае успеха app_id != nil
```
objc_msgSend(init_DCDDeviceMetadata,
             "generateEncryptedBlobWithCompletion:",
             v4);
...
objc_release(init_DCDDeviceMetadata);
```

v4 здесь и есть блок, который XPC-демон вызовет, чтобы отправить ответ клиенту. Когда в DCDDeviceMetadata этот v4 вызовут (как показано в __57__…_block_invoke), данные улетают обратно в мой процесс.




что же в итоге

```
App → DCDevice.generateToken
   → XPC → DCClientHandler.fetchOpaqueBlob
      → DCDDeviceMetadata.generateEncryptedBlob
         → DCCryptoProxyImpl.fetchOpaqueBlob
            → DCCertificateGenerator.generateEncryptedCertificateChain
               → _generateCertificateChain…
                  → __74__…block_invoke → _encryptData…
                     → completion(rawChain, nil)
               ← completion(encryptedBlob, nil)
            ← innerBlock(rawChain, nil)
         ← completionBlock(encryptedBlob, nil)
      ← XPC → App receives NSData token
```
