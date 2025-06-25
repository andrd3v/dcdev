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

При получении соедниения он вызвает DCClientHandler initWithConnection: в своем листенере DCXPCListener listener:shouldAcceptNewConnection:
во первых этот метод вызывает 
```if ( -[DCClientHandler _isSupported](self, "_isSupported") ) ```
значение этой переменной жестко закодирвоанно в DeviceIdentityIsSupported из приватного фреймворка
```__int64 DeviceIdentityIsSupported_1()
{
  return 1LL;
}
```

*** предположу, что если дсдевайс недоступен на девайсе, то там будет другая сборка фреймворка и там будет закодирован 0 ***

после проверки поддрежки вызывается DCClientHandler _generateAppIDFromCurrentConnection
этот метод получает

```team_id_and_bundle = objc_claimAutoreleasedReturnValue(
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
```if (team_id && [team_id length] && ![team_id isEqualToString:@"0000000000"]) {
    appID = [NSString stringWithFormat:@"%@.%@", team_id, bundle_id];
} else {
    appID = bundle_id;
}
```
вернет строку которая получилась в итоге, если она не пустая 
```return [appID length] ? appID : nil;```


