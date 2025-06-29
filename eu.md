
# dcdev

**📅 June 25, 2025**

When creating a dcdevice token, the method ` [DCDevice generateTokenWithCompletionHandler:]` is used. Internally, it calls `DCDeviceMetadataDaemonConnection`.

This method establishes a connection to the iPhone daemon `devicecheckd`:

```objc
v4 = (NSXPCConnection *)objc_msgSend(
                              objc_alloc((Class)&OBJC_CLASS___NSXPCConnection),
                              "initWithMachServiceName:options:",
                              CFSTR("com.apple.devicecheckd"),
                              0LL);
```

---

We move into `devicecheckd`.

Upon receiving a connection, it calls `DCClientHandler initWithConnection:` → `DCClientHandler fetchOpaqueBlobWithCompletion` in its listener `DCXPCListener listener:shouldAcceptNewConnection:`.

First, this method invokes:

```objc
if ( -[DCClientHandler _isSupported](self, "_isSupported") )
```

The value of this check is hard-coded in `DeviceIdentityIsSupported` from the private framework:

```c
__int64 DeviceIdentityIsSupported_1()
{
  return 1LL;
}
```

***I assume that if dcdevice is unavailable on the device, a different build of the framework will encode `0` here.***

After verifying support, it calls `DCClientHandler _generateAppIDFromCurrentConnection`, which obtains:

```objc
team_id_and_bundle = objc_claimAutoreleasedReturnValue(
    -[DCClientHandler _stringValueForEntitlement:]
        (self, "_stringValueForEntitlement:",
         CFSTR("application-identifier")));
// ABCDE12345.com.example.myApp
// "<TeamID>.<BundleIdentifier>"
```

If that fails to yield a bundle ID, it falls back:

```objc
if (![team_id_and_bundle length]) {
    // entitlement is empty → fallback

    CFStringRef team_id   = SecTaskCopyTeamIdentifier(task,   NULL);
    CFStringRef bundle_id = SecTaskCopySigningIdentifier(task, NULL);
}
```

If `team_id` is valid and not `"0000000000"`, it joins them with a dot; otherwise, it uses only the bundle ID:

```objc
if (team_id && [team_id length] && ![team_id isEqualToString:@"0000000000"]) {
    appID = [NSString stringWithFormat:@"%@.%@", team_id, bundle_id];
} else {
    appID = bundle_id;
}
```

It returns the resulting string if non-empty:

```objc
return [appID length] ? appID : nil;
```

**📅 June 26, 2025**

In `DCClientHandler fetchOpaqueBlobWithCompletion` it calls:

```objc
[DCDDeviceMetadata initWithContext:cryptoProxy:…];
```

// I thought this might be hookable and vulnerable, but since this is a daemon, we cannot attack it, nor methods in the internal framework, nor can we create our own daemon to replace the DeviceCheck connection—we can only fully emulate its behavior.

```objc
DCContext_class = objc_alloc_init((Class)&OBJC_CLASS___DCContext);
objc_msgSend(DCContext_class, "setClientAppID:", app_id);
DCDDeviceMetadata = objc_alloc((Class)&OBJC_CLASS___DCDDeviceMetadata);
DCCryptoProxyImpl = objc_alloc_init((Class)&OBJC_CLASS___DCCryptoProxyImpl);
// create two helper classes
v11 = objc_msgSend(DCDDeviceMetadata, "initWithContext:cryptoProxy:", DCContext_class, DCCryptoProxyImpl);
// pass our classes into DeviceCheckInternal
```

Returning to the call in the daemon (not the private framework):

```objc
objc_msgSend(init_DCDDeviceMetadata, "generateEncryptedBlobWithCompletion:", v4);
```

This method initiates asynchronous generation of the "encrypted blob" (token), proxying the request to the cryptographic layer (`DCCryptoProxy`) and passing the client completion handler.

```objc
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
                                completion:innerBlock];

    [cb release];
}
```

`DCCryptoProxyImpl` → `DCCertificateGenerator`:

`DCCryptoProxyImpl` essentially only triggers logging/signaling; the core logic resides in the block:

```objc
void __noreturn __59__DCCryptoProxyImpl_fetchOpaqueBlobWithContext_completion___block_invoke(
        int64_t a1, void *publicKey)
{
    // 1) Capture publicKey
    DCCertificateGenerator *gen = [[DCCertificateGenerator alloc]
        initWithContext:(DCContext *)*(uint64_t *)(a1 + 32)
               publicKey:(id)objc_retain(publicKey)];

    // 2) Generate chain and encrypt
    [gen generateEncryptedCertificateChainWithCompletion:
        (void (^)(NSData *blob, NSError *error))*(uint64_t *)(a1 + 40)];

    objc_release(gen);
}
```

## `DCCertificateGenerator` Methods

```objc
- (instancetype)initWithContext:(DCContext *)context
                     publicKey:(id)publicKey;
- (void)generateEncryptedCertificateChainWithCompletion:
         (void (^)(NSData *encryptedBlob, NSError *error))completion;
```

### Initialization:

```objc
id __cdecl -[DCCertificateGenerator initWithContext:publicKey:](...)
{
    // 1. Retain passed parameters
    _publicKey = [publicKey retain];
    _context   = [context retain];
    // 2. Call base init
    self = [super init];
    return self;
}
```

Stores in instance fields:

* `self->_publicKey` – the app’s public key
* `self->_context` – the `DCContext` session parameters

### Generating the chain:

In `generateEncryptedCertificateChainWithCompletion:`, it retains the callback block, creates a stack block, and calls the private raw-chain generator:

```objc
void __cdecl -[DCCertificateGenerator generateEncryptedCertificateChainWithCompletion:](...)
{
    id completion = [a3 retain];
    void (^block)(NSData *rawChain, NSDate *serverDate) = ^(NSData *rawChain, NSDate *serverDate) {
        // see below
    };
    [self _generateCertificateChainWithCompletion:block];
    [completion release];
}
```

The block (`__74__…block_invoke`) handles rawChain encryption:

```objc
void __fastcall __74__…block_invoke(int64_t block_ptr,
                                     int64_t rawChain,      // a2
                                     int64_t serverDate)     // a3
{
    if (rawChain) {
        void *encryptor = *(void **)(block_ptr + 32);

        //    - rawChain        : NSData *
        //    - serverDate      : NSDate *
        //    - &errorPointer   : NSError **
        NSError *error = nil;
        NSData *encryptedBlob = [encryptor _encryptData:rawChain
                                     serverSyncedDate:serverDate
                                                 error:&error]; 

        completion(encryptedBlob, error);

        [encryptedBlob release];
        [error release];
    } else {
        NSError *error = [NSError errorWithDomain:@"com.apple.devicecheck.cryptoerror"
                                             code:0
                                         userInfo:nil];

        completion(nil, error);

        [error release];
    }
}
```

#### Encryption Overview:

* Serialize data using CBOR
* Perform ephemeral ECDH to derive a shared secret
* Use HKDF to derive a symmetric key
* Encrypt with AES-GCM
* Assemble final blob

A reconstructed (pseudocode) version of `_encryptData:serverSyncedDate:error:`:

```objc
- (NSData *)_encryptData:(NSData *)data
        serverSyncedDate:(NSDate *)serverDate
                    error:(NSError **)error
{
    NSData   *plainData      = [data retain];
    NSDate   *syncedDate     = [serverDate retain];
    NSData   *clientAppIDRaw = [[[self clientAppID] dataUsingEncoding:NSUTF8StringEncoding] retain];
    NSError  *localError     = nil;


    //  v27 = (void *)objc_claimAutoreleasedReturnValue_5(objc_msgSend(*(id *)(v25 + 16), "clientAppID"));
    //  v28 = (void *)objc_claimAutoreleasedReturnValue_5(objc_msgSend(v27, "dataUsingEncoding:", 4LL));   
    const void *plainBytes = [plainData bytes];
    size_t      plainLen   = [plainData length];
    const void *appIDBytes = [clientAppIDRaw bytes];
    size_t      appIDLen   = [clientAppIDRaw length];

    uint8_t pubKeyBuf[65] = {0};
    aks_ref_key_get_public_key(self->_refKey, pubKeyBuf);
    // print pubKeyBuf…

    uint8_t  *sharedSecret = NULL;
    size_t    sharedLen    = 0;
    int ecdhOK = aks_ref_key_compute_key(
        pubKeyBuf, pubKeyBuf, &sharedSecret, &sharedLen);
    if (!ecdhOK) { goto cleanup; }

    uint8_t derivedKey[32] = {0};
    cchkdf(..., sharedSecret+2, sharedLen-2, derivedKey);

    uint8_t derivedIV[12]; memcpy(derivedIV, derivedKey+32, 12);

    uint32_t envelopePayloadLen = appIDLen + plainLen + 81;
    size_t envelopeTotalLen = envelopePayloadLen + 154;
    uint8_t *envelope = calloc(1, envelopeTotalLen);
    envelope[0] = 2;
    memcpy(envelope+5, pubKeyBuf, 65);
    *(uint32_t *)(envelope+150) = envelopePayloadLen;

    uint8_t *payload = calloc(1, envelopePayloadLen);
    NSTimeInterval ts = [syncedDate timeIntervalSince1970];
    *(uint64_t *)(payload+65) = (uint64_t)ts;
    *(uint32_t *)(payload+73) = appIDLen;
    memcpy(payload+81, appIDBytes, appIDLen);
    *(uint32_t *)(payload+77) = plainLen;
    memcpy(payload+81+appIDLen, plainBytes, plainLen);

    ccgcm_one_shot(sharedSecret, derivedKey, derivedIV, envelopePayloadLen, payload, 16, envelope+4);

    NSData *result = [[[NSData alloc] initWithBytes:envelope
                                             length:envelopeTotalLen] autorelease];
    free(envelope);
    free(payload);
    _DCLogSystem();

cleanup:
    if (sharedSecret) aks_ref_key_free(&sharedSecret);
    [plainData release];
    [clientAppIDRaw release];
    [syncedDate release];
    if (localError) { if (error) *error = localError; return nil; }
    return result;
}
```

After this, the raw-chain generator `_generateCertificateChainWithCompletion:` is called:

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
  -[DCCertificateGenerator _generateCertificateChainWithCompletion:](v5, "_generateCertificateChainWithCompletion:", v4); // <------ !!!!!!
  objc_release(v6);
  objc_release(v3);
}
```

Inside, `[DCCryptoUtilities identityCertificateOptions]` → `[DCCryptoUtilities generateTTL]` generates a TTL:

```objc
unsigned int +[DCCryptoUtilities generateTTL](...) {
  return arc4random_uniform(0x40561u) + 262080;
}
```

In the block `__66__…block_invoke`, certificates are processed to PEM chain:

```objc
void (^generateCertificateChainBlock)(NSArray *) = ^(NSArray *certificates) 
{
    os_log_t logger = os_log_create("com.apple.devicecheck", "DCCertificateGenerator");
    if (os_log_type_enabled(logger, OS_LOG_TYPE_INFO)) {
        os_log_info(logger, "Certificate issued, processing..");
    }

    NSDate *currentDate = [NSDate date];

    if (!certificates || certificates.count != 2) {
        NSDictionary *userInfo = @{
            NSLocalizedDescriptionKey: @"Invalid inputs."
        };
        NSError *error = [NSError errorWithDomain:@"com.apple.devicecheck.cryptoerror"
                                             code:0
                                         userInfo:userInfo];
        
        if (os_log_type_enabled(logger, OS_LOG_TYPE_ERROR)) {
            os_log_error(logger, "Invalid certificate chain: %{public}@", error);
        }
        
        completionBlock(nil, nil, error);
        return;
    }

    NSMutableData *certificateChainData = [NSMutableData data];
    NSError *processingError = nil;

    for (NSUInteger i = 0; i < certificates.count; i++) {
        SecCertificateRef certificate = (__bridge SecCertificateRef)certificates[i];
        NSData *derData = (__bridge_transfer NSData *)SecCertificateCopyData(certificate);
        
        if (!derData) {
            NSDictionary *userInfo = @{
                NSLocalizedDescriptionKey: @"Failed to convert certificate."
            };
            processingError = [NSError errorWithDomain:@"com.apple.devicecheck.cryptoerror"
                                                  code:0
                                              userInfo:userInfo];
            break;
        }

        NSString *pemHeader = @"-----BEGIN CERTIFICATE-----\n";
        NSString *pemFooter = @"\n-----END CERTIFICATE-----";
        
        NSData *base64Data = [derData base64EncodedDataWithOptions:NSDataBase64Encoding64CharacterLineLength];
        NSString *base64String = [[NSString alloc] initWithData:base64Data encoding:NSUTF8StringEncoding];
        
        if (!base64String) {
            NSDictionary *userInfo = @{
                NSLocalizedDescriptionKey: @"Base64 encoding failed."
            };
            processingError = [NSError errorWithDomain:@"com.apple.devicecheck.cryptoerror"
                                                  code:0
                                              userInfo:userInfo];
            break;
        }

        NSString *pemEntry = [NSString stringWithFormat:@"%@%@%@", pemHeader, base64String, pemFooter];
        NSData *pemData = [pemEntry dataUsingEncoding:NSUTF8StringEncoding];
        
        if (i > 0) {
            [certificateChainData appendData:[@"\n" dataUsingEncoding:NSUTF8StringEncoding]];
        }
        [certificateChainData appendData:pemData];
    }

    NSDate *validityDate = currentDate;
    if (certificates.count >= 3) {
        id potentialDate = certificates[2];
        if ([potentialDate isKindOfClass:[NSDate class]]) {
            validityDate = potentialDate;
            if (os_log_type_enabled(logger, OS_LOG_TYPE_DEBUG)) {
                os_log_debug(logger, "Using synced timestamp");
            }
        } else {
            NSDictionary *userInfo = @{
                NSLocalizedDescriptionKey: @"Expected date field, failing..."
            };
            processingError = [NSError errorWithDomain:@"com.apple.devicecheck.cryptoerror"
                                                  code:0
                                              userInfo:userInfo];
        }
    }

    if (processingError) {
        if (os_log_type_enabled(logger, OS_LOG_TYPE_ERROR)) {
            os_log_error(logger, "Certificate processing failed: %{public}@", processingError);
        }
        completionBlock(nil, nil, processingError);
    } else {
        if (os_log_type_enabled(logger, OS_LOG_TYPE_INFO)) {
            os_log_info(logger, "Certificate processed successfully. Size: %lu", (unsigned long)certificateChainData.length);
        }
        completionBlock(certificateChainData, validityDate, nil);
    }
};
```

Finally, `DeviceIdentityIssueClientCertificateWithCompletion_0` is invoked to issue the client certificate:

```objc
void DeviceIdentityIssueClientCertificateWithCompletion_0(
    dispatch_queue_t queue,
    id identityOptions,
    DeviceIdentityCompletionBlock completionBlock)
{
    if (queue) dispatch_retain(queue);
    id opts = [identityOptions retain];
    DeviceIdentityCompletionBlock blk = [completionBlock copy];

    BOOL supported = isSupportedDeviceIdentityClient(NULL);

    if (supported) {
        dispatch_queue_t serialQ = copyDeviceIdentitySerialQueue();
        dispatch_async(serialQ, ^{
            // --- There should be logic for building and obtaining a chain certificate, but damn it, it's not there.---
            CFDataRef cfCert = 
            NSData *certData = (__bridge_transfer NSData *)cfCert;
            NSError *error = nil;

            blk(certData, genError);

            [opts release];
            [blk release];
            dispatch_release(serialQ);
        });
        dispatch_release(serialQ);
    } else {
        NSError *err = createMobileActivationError(
            "DeviceIdentityIssueClientCertificateWithCompletion",
            860,      
            -1,       
            NULL,
            CFSTR("Client is not supported.")
        );
        blk(nil, err);

        [blk release];
        [opts release];
    }

    if (queue) dispatch_release(queue);
}
```

These results propagate back to `[DCCertificateGenerator generateEncryptedCertificateChainWithCompletion:]` and, in `DCClientHandler`, on success (`app_id != nil`):

```objc
objc_msgSend(init_DCDDeviceMetadata,
             "generateEncryptedBlobWithCompletion:",
             v4);
objc_release(init_DCDDeviceMetadata);
```

Here, `v4` is the block that the XPC daemon will call to send the response back to my process.
