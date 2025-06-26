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


