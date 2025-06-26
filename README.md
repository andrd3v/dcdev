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

  //я думал что тут можно хукнуть и атаковать , но так как это демон, то атаковать нельзя, также нельзя атаковать методы из интерналфреймворка и мы не можем создать свой демон чтобы подменить коннектион в DeviceCheck - мы можем только полностью эмулировать его работу и все
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


- (NSData *)_encryptData:(NSData *)data
        serverSyncedDate:(NSDate *)serverDate
                    error:(NSError **)error
{
    NSData   *plainData      = [data retain];
    NSDate   *syncedDate     = [serverDate retain];
    NSData   *clientAppIDRaw = [[[self clientAppID] dataUsingEncoding:NSUTF8StringEncoding] retain];
    NSError  *localError     = nil;

    os_log_t logHandle = [OSLog subsystem:@"com.apple.devicecheck" category:@"DCCertificateGenerator"];
    if (os_log_type_enabled(logHandle, OS_LOG_TYPE_DEFAULT)) {
        os_log(logHandle, "Encrypting data...");
    }

    // Получаем байты и длины
    const void *plainBytes = [plainData bytes];
    size_t      plainLen   = [plainData length];
    const void *appIDBytes = [clientAppIDRaw bytes];
    size_t      appIDLen   = [clientAppIDRaw length];


    // --- ECDH: получаем и печатаем публичный ключ ---
    // Буфер для публичного ключа (длина – 65 байт) (ВАЖНО: тм ам есть проврека на длинну ключа, что он 65 - я опустил ее тут)
    uint8_t pubKeyBuf[65] = {0};
    aks_ref_key_get_public_key(self->_refKey, pubKeyBuf);
    printf("%-25.25s = ", "random_pubkey");
    for (int i = 0; i < 65; i++) {
        printf("%02x", pubKeyBuf[i]);
    }
    putchar('\n');

    // --- Вычисляем общий секрет ECDH ---
    uint8_t  *sharedSecret = NULL;
    size_t    sharedLen    = 0;
    int ecdhOK = aks_ref_key_compute_key(
        pubKeyBuf,              // remote pubkey
        0, 0,
        (uint8_t *)pubKeyBuf,    // локальный приватный ключ храним внутри refKey
        &sharedSecret, &sharedLen
    );
    if (!ecdhOK) {
        _DCLogSystem();
        localError = [NSError errorWithDomain:@"DCCertificateGenerator"
                                         code:-1
                                     userInfo:@{NSLocalizedDescriptionKey: @"ECDH failed"}];
        goto cleanup;
    }

    printf("%-25.25s = ", "ECDH shared key");
    for (size_t i = 2; i < sharedLen; i++) { // первые 2 байта – заголовок
        printf("%02x", sharedSecret[i]);
    }
    putchar('\n');

    // --- HKDF (SHA256) ---
    uint8_t derivedKey[32] = {0};
    if (cchkdf(ccsha256_ltc_di_ptr,
               sharedLen - 2,
               sharedSecret + 2,
               0, 0, 0, 0,
               sizeof(derivedKey),
               derivedKey) != 0)
    {
        _DCLogSystem();
        localError = [NSError errorWithDomain:@"DCCertificateGenerator"
                                         code:-2
                                     userInfo:@{NSLocalizedDescriptionKey: @"HKDF failed"}];
        goto cleanup;
    }

    printf("%-25.25s = ", "HKDF derived key");
    for (int i = 0; i < 32; i++) {
        printf("%02x", derivedKey[i]);
    }
    putchar('\n');

    // Извлекаем IV (12 байт) – вторую половину HKDF
    uint8_t derivedIV[12] = {0};
    memcpy(derivedIV, derivedKey + 32, sizeof(derivedIV)); // в оригинале v13[128] байтов, здесь берём первые 12

    printf("%-25.25s = ", "HKDF derived iv");
    for (int i = 0; i < 12; i++) {
        printf("%02x", derivedIV[i]);
    }
    putchar('\n');

    // общая длина «зашивки» (appID + plain + 81)
    uint32_t envelopePayloadLen = (uint32_t)(appIDLen + plainLen + 81);

    // allocate header+payload
    size_t envelopeTotalLen = envelopePayloadLen + 154; // 235 ≈ 154 + 81
    uint8_t *envelope = calloc(1, envelopeTotalLen);
    if (!envelope) {
        localError = [NSError errorWithDomain:@"DCCertificateGenerator"
                                         code:-3
                                     userInfo:@{NSLocalizedDescriptionKey: @"calloc failed"}];
        goto cleanup;
    }

    envelope[0] = 2;

    memcpy(envelope + 5, pubKeyBuf, sizeof(pubKeyBuf));
    *(uint32_t *)(envelope + 150) = envelopePayloadLen;

    uint8_t *payload = calloc(1, envelopePayloadLen);
    if (!payload) {
        free(envelope);
        localError = [NSError errorWithDomain:@"DCCertificateGenerator"
                                         code:-4
                                     userInfo:@{NSLocalizedDescriptionKey: @"calloc payload failed"}];
        goto cleanup;
    }

    NSTimeInterval ts = [syncedDate timeIntervalSince1970];
    *(uint64_t *)(payload + 65) = (uint64_t)ts;

    *(uint32_t *)(payload + 73) = (uint32_t)appIDLen;
    memcpy(payload + 81, appIDBytes, appIDLen);

    *(uint32_t *)(payload + 77) = (uint32_t)plainLen;
    memcpy(payload + 81 + appIDLen, plainBytes, plainLen);

    // --- AES-GCM шифрование всего payload ---
    if (ccgcm_one_shot(sharedSecret,
                       sizeof(derivedKey),
                       derivedKey,
                       sizeof(derivedIV),
                       derivedIV,
                       0, 0,
                       envelopePayloadLen,
                       payload,
                       16,
                       envelope + 4) != 0)
    {
        _DCLogSystem();
        free(envelope);
        free(payload);
        localError = [NSError errorWithDomain:@"DCCertificateGenerator"
                                         code:-5
                                     userInfo:@{NSLocalizedDescriptionKey: @"AES-GCM failed"}];
        goto cleanup;
    }

    printf("%-25.25s = ", "tag");
    for (int i = 4; i < 20; i++) {
        printf("%02x", envelope[i]);
    }
    putchar('\n');

    fprintf(stderr, "encrypted_data_len: %u\n", envelopePayloadLen);

    // NSData для возврата
    NSData *result = [[[NSData alloc] initWithBytes:envelope
                                             length:envelopeTotalLen] autorelease];

    free(envelope);
    free(payload);

    _DCLogSystem();

cleanup:
    if (sharedSecret)      aks_ref_key_free(&sharedSecret);
    [plainData release];
    [clientAppIDRaw release];
    [syncedDate release];

    if (localError)
    {
        if (error) *error = localError;
        return nil;
    }
    return result;
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

# Цепочка генерации токена DeviceCheck

1. **App → `DCDevice.generateToken`**  
   Клиентский код инициирует запрос на генерацию токена.

2. **XPC → `DCClientHandler.fetchOpaqueBlob`**  
   `DCDeviceMetadataDaemonConnection` пересылает вызов демону `devicecheckd`.

3. **`DCDDeviceMetadata.generateEncryptedBlob`**  
   Формирует внутренний блок и проксирует в криптослой.

4. **`DCCryptoProxyImpl.fetchOpaqueBlob`**  
   Запускает логирование и передает контекст и публичный ключ в генератор сертификатов.

5. **`DCCertificateGenerator.generateEncryptedCertificateChain`**  
   Асинхронно собирает «сырую» цепочку сертификатов.

6. **`_generateCertificateChain…` → `__74__…block_invoke` → `_encryptData`**  
   Сериализует данные (CBOR), выполняет ECDH+HKDF, шифрует AES-GCM и возвращает зашифрованный blob.

7. **`completion(rawChain, nil)` → `innerBlock(rawChain, nil)` → `completionBlock(encryptedBlob, nil)`**  
   Внутренние блоки прокидывают результат до исходного XPC-блока.

8. **XPC → App receives `NSData` token**  
   Демон отправляет зашифрованный blob обратно в приложение по XPC.

