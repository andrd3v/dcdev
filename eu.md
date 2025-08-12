My English is not the best, so if something is not clear, then refer to the Russian documentation and translate it

### Work done by [andrd3v](https://github.com/andrd3v). Help provided by [whoeevee](https://github.com/whoeevee). Thanks for reading!
reworking my old private report on dcdevice

# Chapter 1
### DeviceCheck.framework

When creating a token from DCDevice, the method `-[DCDevice generateTokenWithCompletionHandler:]` is used – inside it calls `DCDeviceMetadataDaemonConnection`. This method creates a connection to the iPhone’s devicecheckd daemon:
```objc
NSXPCConnection *xpc_connection = (NSXPCConnection *)objc_msgSend(
                              objc_alloc((Class)&OBJC_CLASS___NSXPCConnection),
                              "initWithMachServiceName:options:",
                              CFSTR("com.apple.devicecheckd"),
                              0LL);
```

`devicecheckd`

After connecting to the devicecheckd daemon via XPC, the following chain of calls starts. When a connection is received, the daemon invokes `-[DCXPCListener listener:shouldAcceptNewConnection:]` -> `-[DCClientHandler initWithConnection:]`, and after the client makes an RPC call -> `-[DCClientHandler fetchOpaqueBlobWithCompletion]`. First of all, `-[DCClientHandler fetchOpaqueBlobWithCompletion]` calls:
```objc
if ( -[DCClientHandler _isSupported](self, "_isSupported") )
The value of this variable is hard-coded in DeviceIdentityIsSupported from DeviceIdentity.framework:
__int64 DeviceIdentityIsSupported_1()
{
  return 1LL;
}
```
I assume that if DCDevice is unavailable on the device, then there will be a different build of the framework where it is hard-coded to 0. After checking support, it calls `-[DCClientHandler _generateAppIDFromCurrentConnection]` – this method obtains <TeamID>.<BundleIdentifier> (format: ABCDE12345.com.example.myApp) using entitlements (`-[NSXPCConnection valueForEntitlement:]`), and if that fails, it uses `SecTaskCopyTeamIdentifier` and `SecTaskCopySigningIdentifier`. If team_id is valid and not "0000000000", the method combines the team_id and bundle_id with a dot; otherwise it uses only the bundle_id. It returns an appID (<TeamID>.<BundleIdentifier> or <BundleIdentifier>):
`return [appID length] ? appID : nil;`
Returning to `-[DCClientHandler fetchOpaqueBlobWithCompletion]` (important note: I will not consider the fallback cases where the code goes into the else branch for error handling):
```objc
if ( app_id )
{
  DCContext_class = objc_alloc_init((Class)&OBJC_CLASS___DCContext);
  objc_msgSend(DCContext_class, "setClientAppID:", app_id);  // we set our "<TeamID>.<BundleIdentifier>" in the class

  // allocate DCDDeviceMetadata and initialize DCCryptoProxyImpl from DeviceCheckInternal.framework
  DCDDeviceMetadata = objc_alloc((Class)&OBJC_CLASS___DCDDeviceMetadata);
  DCCryptoProxyImpl = objc_alloc_init((Class)&OBJC_CLASS___DCCryptoProxyImpl);

  // initialize DCDDeviceMetadata from DeviceCheckInternal.framework
  init_DCDDeviceMetadata = objc_msgSend(
                             DCDDeviceMetadata,
                             "initWithContext:cryptoProxy:",
                             DCContext_class,
                             DCCryptoProxyImpl);

  objc_msgSend(init_DCDDeviceMetadata, "generateEncryptedBlobWithCompletion:", v4);
}
```
In summary: the devicecheckd daemon returns an encrypted token (opaque blob) to DeviceCheck.framework via XPC in the completionHandler block passed through `-[DCDDeviceMetadata generateEncryptedBlobWithCompletion:]`. This blob is later used as the token parameter in `-[DCDDeviceMetadata generateTokenWithCompletionHandler:]`.
Breaking Down DCContext, DCDDeviceMetadata, and DCCryptoProxyImpl in fetchOpaqueBlobWithCompletion
`DeviceCheckInternal.framework`

The first thing that happens in `-[DCClientHandler fetchOpaqueBlobWithCompletion]` is the initialization of DCContext and assignment of our <TeamID>.<BundleIdentifier> or <BundleIdentifier> to it:
```objc
DCContext_class = objc_alloc_init((Class)&OBJC_CLASS___DCContext);
objc_msgSend(DCContext_class, "setClientAppID:", app_id);
Since DCContext has no -init method of its own, it simply inherits the implementation from NSObject. Memory is allocated for the object and a pointer to the DCContext class is set. Fields (for example _clientAppID) are zero-initialized. Then the -init selector is sent to the DCContext object. Because DCContext does not override this method, the standard -[NSObject init] is called, which simply returns self without additional logic. After that, the method -[DCContext setClientAppID:] is called, which sets self->_clientAppID to our <TeamID>.<BundleIdentifier> or <BundleIdentifier>:
id __cdecl __noreturn -[DCContext clientAppID](DCContext *self, SEL a2)
{
  return objc_getProperty_33(self, a2, 8LL, 1);
}
```
Skipping allocation of DCDDeviceMetadata, we look at:
```objc
DCCryptoProxyImpl = objc_alloc_init((Class)&OBJC_CLASS___DCCryptoProxyImpl);
Similarly, memory is allocated for the object, a pointer to the DCCryptoProxyImpl class is set, and then -[NSObject init] is called. Now our DCDDeviceMetadata (which was just allocated) is initialized:
init_DCDDeviceMetadata = objc_msgSend(
                           DCDDeviceMetadata,
                           "initWithContext:cryptoProxy:",
                           DCContext_class, // our class holding <TeamID>.<BundleIdentifier> or <BundleIdentifier>
                           DCCryptoProxyImpl_class_arg);
```
What happens during the initialization of DCDDeviceMetadata:
```objc
// we are actually working with the instance's memory area, not with a C-structure per se
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
    // set up fields in DCDDeviceMetadata
    j__objc_storeStrong((id *)&dc_device_metadata->_cryptoProxy, DCCryptoProxyImpl_class_arg);
    j__objc_storeStrong((id *)&dc_device_metadata->_context, DCContext_class_arg);  // our context with our <TeamID>.<BundleIdentifier> or <BundleIdentifier>
  }

  return (id *)dc_device_metadata;
}
```
Great, all classes have been initialized and now the devicecheckd daemon calls objc_msgSend(init_DCDDeviceMetadata, "generateEncryptedBlobWithCompletion:", v4); – a method that will begin generating the token.

# Chapter 2
`DeviceCheckInternal.framework`

Important note: almost all methods are rewritten by me to make them easier to read and understand. Our daemon invoked the method `objc_msgSend(init_DCDDeviceMetadata, "generateEncryptedBlobWithCompletion:", v4);` – this is the start of token creation.
```objc
void __cdecl -[DCDDeviceMetadata generateEncryptedBlobWithCompletion:](DCDDeviceMetadata *self, SEL a2, id completion_arg)
{
  id v4 = objc_retain(completion_arg); // retain the completion argument

  // exactly what was set up in -[DCDDeviceMetadata initWithContext:cryptoProxy:]
  DCCryptoProxy *cryptoProxy = self->_cryptoProxy;
  DCContext *context = self->_context; // our context with our <TeamID>.<BundleIdentifier> or <BundleIdentifier>

  [cryptoProxy fetchOpaqueBlobWithContext:context
                              completion:^(NSData *blob, NSError *error) {
      // this block corresponds to __57__DCDDeviceMetadata_generateEncryptedBlobWithCompletion___block_invoke
      // take pointer to the original XPC completion block saved at the time of the call.
      // if argument a2 (data) is non-zero, call completion(data, nil).
      // if a2 is zero, create an NSError with code 0 and call completion(nil, error).
  }];
}
```

Excellent, the method `-[DCCryptoProxy fetchOpaqueBlobWithContext:completion:]` is now called, to which our context (with <TeamID>.<BundleIdentifier> or <BundleIdentifier>) and the completion handler are passed. IDA decompiles the code poorly, so it will be slightly rewritten for clarity:
```objc
void __cdecl -[DCCryptoProxyImpl fetchOpaqueBlobWithContext:completion:](
        DCCryptoProxyImpl *self,
        SEL a2,
        id DCContext_argDCContext_arg,
        id completion_arg)
{
    // hold onto context (<TeamID>.<BundleIdentifier> or <BundleIdentifier>) and completion
    id retainedContext = [DCContext_argDCContext_arg retain];
    void (^copiedCompletion)(NSData *, NSError *) = [completion_arg copy];
  
    if (os_log_type_enabled(self.logger, OS_LOG_TYPE_DEFAULT))
    {
      os_log(self.logger, "Generating certificate...");
    }
  
    __block id blockContext = retainedContext;
    __block void (^blockCompletion)(NSData *, NSError *) = copiedCompletion;

    [self _fetchPublicKey:^(NSData *publicKey) {
          DCCertificateGenerator *generator = [[DCCertificateGenerator alloc]
              initWithContext:blockContext // our context with our <TeamID>.<BundleIdentifier> or <BundleIdentifier>
                     publicKey:publicKey]; // our key
  
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

### Getting the publicKey
First, the publicKey is obtained, and then it is passed on. How do we get the public key:
```objc
void __cdecl -[DCCryptoProxyImpl _fetchPublicKey:](DCCryptoProxyImpl *self, SEL a2, id completion)
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

This method calls `-[DCAssetFetcher fetchPublicKeyAssetWithCompletion:]`.
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

For now I won't comment further; we continue down the chain:
```objc
void __cdecl -[DCAssetFetcher _fetchAssetWithContext:completionHandler:](
        DCAssetFetcher *self,
        SEL a2,
        (DCAssetFetcherContext *)context, // another context, without team or bundle ID
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
Now things get interesting – `_queryMetadataWithContext:`

```objc
void __cdecl __noreturn -[DCAssetFetcher _queryMetadataWithContext:completion:](
        DCAssetFetcher *self,
        SEL a2,
        (DCAssetFetcherContext *)context, // another context, without team or bundle ID
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
    NSUInteger resultCode = [assetQuery queryMetaDataSync]; // this is all from libobjc.A

    // Branch: skip cache or absent (ignoreCachedMetadata || resultCode == 2)
    if ([retainedContext ignoreCachedMetadata] || resultCode == 2)
    {
        [self _handleMissingMetadataWithContext:retainedContext
                                   completion:completionBlock];
    } else {
      if (resultCode != 0)
      {
          // Error: generate NSError and return immediately
          NSError *error = [NSError errorWithDomain:@"com.apple.twobit.fetcherror"
                                               code:0xFFFF_FFFF_FFFF_F448
                                           userInfo:nil];
          completionBlock(nil, error);
          [assetQuery release];
          return;
      }

      // Success: pass data upward
      [self _handleSuccessForQuery:assetQuery
                         completion:completionBlock];
    }

    [assetQuery release];
    [completionBlock release];
    [retainedContext release];
}
```
.........
```objc
- (void)_handleMissingMetadataWithContext:(DCAssetFetchContext *)context
                               completion:(void (^)(DCAsset *asset, NSError *error))completion {
    NSLog(@"[DCAssetFetcher] Query sync result indicated missing asset catalog");

    if (context.allowCatalogRefresh) {
        context.allowCatalogRefresh = NO;
        context.ignoreCachedMetadata = NO;

        // trigger asset catalog refresh and retry the request after completion
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

....

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

......

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

......
```objc
+ (DCAsset *)assetWithMobileAsset:(NSDictionary *)mobileAsset {
    NSNumber *version = mobileAsset[@"com.apple.MobileAsset.AssetVersion"];
    if (![version isKindOfClass:[NSNumber class]] || version.integerValue != 1) {
        NSLog(@"[DCAsset] Unknown asset version: %@", version);
        return nil;
    }

    NSData *pubKeyData = mobileAsset[@"com.apple.devicecheck.pubvalue"]; // assetProperty:
    if (![pubKeyData isKindOfClass:[NSData class]] || pubKeyData.length == 0) {
        NSLog(@"[DCAsset] No public key found in asset");
        return nil;
    }

    DCAsset *asset = [[DCAsset alloc] init];
    asset.version = 1;
    asset.publicKey = pubKeyData;

    NSNumber *refreshInterval = mobileAsset[@"com.apple.devicecheck.refreshtimer"]; // assetProperty:
    if ([refreshInterval isKindOfClass:[NSNumber class]]) {
        asset.publicKeyRefreshInterval = refreshInterval.doubleValue;
    }

    return asset;
}
```

....

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

And the funniest part – none of this was needed and it’s the wrong path: the correct key is in `__37__DCCryptoProxyImpl__fetchPublicKey___block_invoke`:
```+[NSData dataWithBytes:length:](&OBJC_CLASS___NSData, "dataWithBytes:length:", &fallback_server_pubkey, 65LL);```
It is hard-coded and the same in all versions of macOS, iOS, and iPadOS:
`0450d934fa67bcf6f2dfbf96629e0a7238e9205d75f28cfcd84f35a6592bbe058a9c0f8edbca2acb67efb774971ca45f7d856a694fb1b9c40b94fb2e7a5a9498b0`
This is 130 bytes of key – the key itself is 65 characters:
```
╭─    ~ ······························································································································· ✔  at 12:04:28 
╰─ unhex 0450d934fa67bcf6f2dfbf96629e0a7238e9205d75f28cfcd84f35a6592bbe058a9c0f8edbca2acb67efb774971ca45f7d856a694fb1b9c40b94fb2e7a5a9498b0
P�4�g���߿�b�
r8� ]u���O5�Y+������*�g�t��_}�jiO���
                                    ��.zZ���%
```
and that’s our public key.
# Chapter 3
`DeviceCheckInternal.framework`
Important note: almost all methods are rewritten by me to make them easier to read and understand. Returning to `fetchOpaqueBlobWithContext:`
```objc
void __cdecl -[DCCryptoProxyImpl fetchOpaqueBlobWithContext:completion:](
        DCCryptoProxyImpl *self,
        SEL a2,
        id DCContext_argDCContext_arg,
        id completion_arg)
{
    // hold onto context (<TeamID>.<BundleIdentifier> or <BundleIdentifier>) and completion
    id retainedContext = [DCContext_argDCContext_arg retain];
    void (^copiedCompletion)(NSData *, NSError *) = [completion_arg copy];
  
    if (os_log_type_enabled(self.logger, OS_LOG_TYPE_DEFAULT))
    {
      os_log(self.logger, "Generating certificate...");
    }
  
    __block id blockContext = retainedContext;
    __block void (^blockCompletion)(NSData *, NSError *) = copiedCompletion;

    [self _fetchPublicKey:^(NSData *publicKey) // here we receive our key
      {
          DCCertificateGenerator *generator = [[DCCertificateGenerator alloc]
              initWithContext:blockContext // our context with our <TeamID>.<BundleIdentifier> or <BundleIdentifier>
                     publicKey:publicKey]; // our key
  
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
Now let’s look at the initialization of generator:
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
    j__objc_storeStrong((id *)&v9->_publicKey, publicKey_arg); // our obtained public key
    j__objc_storeStrong((id *)&v9->_context, context_arg);     // our context with our <TeamID>.<BundleIdentifier> or <BundleIdentifier>
  }
  return (id *)v9;
}
```

After successful initialization, generateEncryptedCertificateChainWithCompletion is called – creation of the token:
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
First – we need to get the certificate via _generateCertificateChainWithCompletion, and then pass it to __74__DCCertificateGenerator_generateEncryptedCertificateChainWithCompletion___block_invoke – where it will be passed to _encryptData, the place where the token is created. In short, all this is about obtaining two root certificates from the keychain – in theory this can be reversed or the logic replicated (which I very much doubt, since it involves the keychain), but obtaining the certificate happens in __DeviceIdentityIssueClientCertificateWithCompletion_block_invoke. Here are example certificates from the argument:
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
Q0ExMRMwEQYDVQQKDApBcHBsZSBJbmMuMRMwEQYDVQQIDApDYWxpZm9ybmlhMBZ
MBMGByqGSM49AgEGCCqGSM49AwEHA0IABOY3WyNGuRO+wcrpt2Yb6ARssM0g5GFC
y302nZQ3p/DPxR3cG4wqLK73zYEiKjUXU1Uv0bgbV61CmTSvPkd0KmejggFkMIIB
YDAMBgNVHRMBAf8EAjAAMA4GA1UdDwEB/wQEAwIE8DCCAT4GCSqGSIb3Y2QKAQSC
AS8EggErMYIBJ/+EmqGSUA0wCxYEQ0hJUAIDAIAV/4SqjZJEETAPFgRFQ0lEAgcK
TVwAaAAu/4aTtcJjGzAZFgRibWFjBBFkNDphMzozZDozNDo2YTowMP+Gy7XKaRkw
FxYEaW1laQQPMzU5NDA0MDgyMDM4NzA1/4ebydxtFjAUFgRzcm5tBAxGSzJWTURE
VUpDTDj/h6uR0mQyMDAWBHVkaWQEKDU1MzcwZjUyYWQwOWRiYmEyYjZhYTcwZDM4
ZjZmMjc5NzExYmYxNmX/h7u1wmMbMBkWBHdtYWMEEWQ0OmEzOjNkOjM0OjY5OmRm
/4ebldJkOjA4FgRzZWlkBDAwNDI0MzEyQjFFNDM4MDAxNzIxOTE0MTgyNTk0Mjg0
NDdCRjZFMzJCQzNCN0YwQ0YwCgYIKoZIzj0EAwIDRwAwRAIgLNjRS4pu3925qko
ourOxUM3L+8hwVYbT/35bIW9q5KQCICf4Xpvuvdx/DMBbr5Wp9GXn0bxP69/8S99
Z9x+t0ow0
-----END CERTIFICATE-----
```
or:
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
We skip all of these details and go to _encryptData – the creation of the token. Let’s create the token:
```objc
- (NSData *)_encryptData:(NSData *)data // certificates
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
# Chapter 4. Conclusion

In this repository I attempted to show the full path of generating a DeviceCheck token: from calling DCDevice in the app, through XPC and the devicecheckd daemon, to the final encryption of the payload in _encryptData. Key stages are covered – initializing the context with TeamID.BundleID, obtaining the public key (including the fallback constant), assembling the certificate chain, and forming the encrypted blob sent to the server. I hope this report made it a bit clearer why DeviceCheck is such a pain. In theory it can be bypassed (at least by using your own obtained certificates), but I'm too lazy to do that.

### Work done by [andrd3v](https://github.com/andrd3v). Help provided by [whoeevee](https://github.com/whoeevee). Thanks for reading!
