
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
                                     int64_t rawChain,
                                     int64_t serverDate)
{
    if (rawChain) {
        void *encryptor = *(void **)(block_ptr + 32);
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

    // Logging…

    const void *plainBytes = [plainData bytes];
    size_t      plainLen   = [plainData length];
    const void *appIDBytes = [clientAppIDRaw bytes];
    size_t      appIDLen   = [clientAppIDRaw length];

    // ECDH: get public key
    uint8_t pubKeyBuf[65] = {0};
    aks_ref_key_get_public_key(self->_refKey, pubKeyBuf);
    // print pubKeyBuf…

    // Compute shared secret with ECDH
    uint8_t  *sharedSecret = NULL;
    size_t    sharedLen    = 0;
    int ecdhOK = aks_ref_key_compute_key(
        pubKeyBuf, pubKeyBuf, &sharedSecret, &sharedLen);
    if (!ecdhOK) { goto cleanup; }

    // HKDF (SHA256)
    uint8_t derivedKey[32] = {0};
    cchkdf(..., sharedSecret+2, sharedLen-2, derivedKey);

    // Extract IV (12 bytes) from derivedKey
    uint8_t derivedIV[12]; memcpy(derivedIV, derivedKey+32, 12);

    // Build envelope and payload buffers
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

    // AES-GCM encrypt payload into envelope+tag
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
void __cdecl -[DCCertificateGenerator generateEncryptedCertificateChainWithCompletion:](...)
{
  // prepare block struct …
  objc_msgSend(self, "_generateCertificateChainWithCompletion:", blockStruct);
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
void (^generateCertificateChainBlock)(NSArray *) = ^(NSArray *certificates) {
    if (!certificates || certificates.count != 2) { … }
    NSMutableData *certificateChainData = [NSMutableData data];
    for (NSUInteger i = 0; i < certificates.count; i++) {
        SecCertificateRef cert = certificates[i];
        NSData *derData = SecCertificateCopyData(cert);
        NSString *pemHeader = @"-----BEGIN CERTIFICATE-----\n";
        NSString *pemFooter = @"\n-----END CERTIFICATE-----";
        NSData *base64Data = [derData base64EncodedDataWithOptions:NSDataBase64Encoding64CharacterLineLength];
        NSString *base64String = [[NSString alloc] initWithData:base64Data encoding:NSUTF8StringEncoding];
        NSString *pemEntry = [NSString stringWithFormat:@"%@%@%@", pemHeader, base64String, pemFooter];
        [certificateChainData appendData:[pemEntry dataUsingEncoding:NSUTF8StringEncoding]];
        if (i > 0) [certificateChainData appendData:[@"\n" dataUsingEncoding:NSUTF8StringEncoding]];
    }
    NSDate *validityDate = [NSDate date];
    completionBlock(certificateChainData, validityDate, nil);
};
```

Finally, `DeviceIdentityIssueClientCertificateWithCompletion_0` is invoked to issue the client certificate:

```objc
void DeviceIdentityIssueClientCertificateWithCompletion_0(
    dispatch_queue_t queue,
    id identityOptions,
    DeviceIdentityCompletionBlock completionBlock)
{
    BOOL supported = isSupportedDeviceIdentityClient(NULL);
    if (supported) {
        dispatch_queue_t serialQ = copyDeviceIdentitySerialQueue();
        dispatch_async(serialQ, ^{
            // Missing actual logic in disassembly
            NSData *certData = ...;
            blk(certData, genError);
            dispatch_release(serialQ);
        });
    } else {
        NSError *err = createMobileActivationError(...);
        blk(nil, err);
    }
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
