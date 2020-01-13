# Advanced Usage

Alamofire is built on top of `URLSession` and the Foundation URL Loading System. To make the most of this framework, it is recommended that you be familiar with the concepts and capabilities of the underlying networking stack.

**Recommended Reading**

- [URL Loading System Programming Guide](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/URLLoadingSystem/URLLoadingSystem.html)
- [`URLSession` Class Reference](https://developer.apple.com/reference/foundation/urlsession)
- [`URLCache` Class Reference](https://developer.apple.com/reference/foundation/urlcache)
- [`URLAuthenticationChallenge` Class Reference](https://developer.apple.com/reference/foundation/urlauthenticationchallenge)

## `Session`
Alamofire’s `Session` is roughly equivalent in responsibility to the `URLSession` instance it maintains: it provides API to produce the various `Request` subclasses encapsulating different `URLSessionTask` subclasses, as well as encapsulating a variety of configuration applied to all `Request`s produced by the instance.

`Session` provides a `default` singleton instance which powers the top-level API from the `AF` enum namespace. As such, the following two statements are equivalent:

```swift
AF.request("https://httpbin.org/get")
```

```swift
let session = Session.af.default
session.request("https://httpbin.org/get")
```

### Creating Custom `Session` Instances
Most applications will need to customize the behavior of their `Session` instances in a variety of ways. The easiest way to accomplish this is to use the following convenience initializer and store the result in a singleton used throughout the app.

```swift
public convenience init(configuration: URLSessionConfiguration = URLSessionConfiguration.af.default,
                        delegate: SessionDelegate = SessionDelegate(),
                        rootQueue: DispatchQueue = DispatchQueue(label: "org.alamofire.session.rootQueue"),
                        startRequestsImmediately: Bool = true,
                        requestQueue: DispatchQueue? = nil,
                        serializationQueue: DispatchQueue? = nil,
                        interceptor: RequestInterceptor? = nil,
                        serverTrustManager: ServerTrustManager? = nil,
                        redirectHandler: RedirectHandler? = nil,
                        cachedResponseHandler: CachedResponseHandler? = nil,
                        eventMonitors: [EventMonitor] = [])
```

This initializer allows the customization of all fundamental `Session` behaviors.

#### Creating a `Session` With a `URLSessionConfiguration`
To customize the behavior of the underlying `URLSesion`, a customized `URLSessionConfiguration` instance can be provided. Starting from the `URLSession.af.default` instance is recommended, as it adds the default `Accept-Encoding`, `Accept-Language`, and `User-Agent` headers provided by Alamofire, but any `URLSessionConfiguration` can be used.

```swift
let configuration = URLSessionConfiguration.af.default
configuration.allowsCellularAccess = false

let session = Session(configuration: configuration)
```

> `URLSessionConfiguration` is **not** the recommended location to set `Authorization` or `Content-Type` headers. Instead, add them to `Request`s using the provided `headers` APIs, using `ParameterEncoder`s, or a `RequestAdapter`.

> As Apple states in their [documentation](https://developer.apple.com/documentation/foundation/urlsessionconfiguration), mutating `URLSessionConfiguration` properties after the instance has been added to a `URLSession` (or, in Alamofire’s case, used to initialize a`Session`) has no effect.

### `SessionDelegate`
A `SessionDelegate` instance encapsulates all handling of the various `URLSessionDelegate` and related protocols callbacks. `SessionDelegate` also acts as the `SessionStateDelegate` for every `Request` produced by Alamofire, allow the `Request` to indirectly important state from the `Session` instance that produced them. `SessionDelegate` can be customized with a specific `FileManager` instance, which will be used for any disk access, like accessing file’s to be uploaded by `UploadRequest`s or files downloaded by `DownloadRequest`s.

```swift
let delelgate = SessionDelegate(fileManager: .default)
```

### `startRequestsImmediately`
By default, `Session` will call `resume()` on a `Request` as soon as it has added at least one response handler. Setting `startRequestsImmediately` to `false` requires that all `Request`s have `resume()` called manually.

```swift
let session = Session(startRequestsImmediately: false)
```

### A `Session`’s `DispatchQueue`s
By default, `Session` instances use a single `DispatchQueue` for all asynchronous work. This includes the `underlyingQueue` of the `URLSession`’s `delegate` `OperationQueue`, for all `URLRequest` creation, all response serialization work, and all internal `Session` and `Request` state mutation. If performance analysis shows a particular bottleneck around `URLRequest` creation or response serialization, `Session` can be provided with separate `DispatchQueue`s for each area of work.

```swift
let rootQueue = DispatchQueue(label: "com.app.session.rootQueue")
let requestQueue = DispatchQueue(label: "com.app.session.requestQueue")
let serializationQueue = DispatchQueue(label: "com.app.session.serializationQueue")

let session = Session(rootQueue: rootQueue, 
                      requestQueue: requestQueue, 
                      serializationQueue: serializationQueue)
```

Any custom `rootQueue` provided **MUST** be a serial queue, but `requestQueue` and `serializationQueue` can be either serial or parallel queues. Serial queues are the recommended default unless performance analysis shows work being delayed, in which case making the queues parallel may help overall performance.

### Adding a `RequestInterceptor`
Alamofire’s `RequestInterceptor` protocol (`RequestAdapter & RequestRetrier`) provides important and powerful request adaptation and retry features. It can be applied at both the `Session` and `Request` level. For more details on `RequestInterceptor` and the various implementations Alamofire includes, like `RetryPolicy`, see [below](#requestinterceptor).

```swift
let policy = RetryPolicy()
let session = Session(interceptor: policy) 
```

### Adding a `ServerTrustManager`
Alamofire’s `ServerTrustManager` class encapsulates mappings between domains and instances of `ServerTrustEvaluating`-conforming types, which provide the ability to customize a `Session`’s handling of TLS security. This includes the use of certificate and public key pinning as well as certificate revocation checking. For more information, see the section about the `ServerTrustManager` and `ServerTrustEvaluating`. Initializing a `ServerTrustManger` is as simple as providing a mapping between the domain and the type of evaluation to be performed:

```swift
let manager = ServerTrustManager(evaluators: ["httpbin.org": PinnedCertificatesTrustEvaluator()])
let session = Session(serverTrustManager: manager)
```

### Adding a `RedirectHandler`
Alamofire’s `RedirectHandler` protocol customizes the handling of HTTP redirect responses. It can be applied at both the `Session` and `Request` level. Alamofire includes the `Redirector` type which conforms to `RedirectHandler` and offers simple control over redirects. For more details on `RedirectHandler`, see the detailed documentation below. 

```swift
let redirector = Redirector(behavior: .follow)
let session = Session(redirectHandler: redirector)
```

### Adding a `CachedResponseHandler`
Alamofire’s `CachedResponseHandler` protocol customizes the caching of responses and can be applied at both the `Session` and `Request` level. Alamofire includes the `ResponseCacher` type which conforms to `CachedResponseHandler` and offers simple control over response caching. For more details, see the detailed documentation below. Link.

```swift
let cacher = ResponseCacher(behavior: .cache)
let session = Session(cachedResponseHandler: cacher)
```

### Adding `EventMonitor`s
Alamofire’s `EventMonitor` protocol provides powerful insight into Alamofire’s internal events. It can be used to provide logging and other event-based features. `Session` accepts an array of `EventMonitor`-conforming instances at initialization time. 

```swift
let monitor = ClosureEventMonitor()
monitor.requestDidCompleteTaskWithError = { (request, task, error) in
    debugPrint(request)
}
let session = Session(eventMonitors: [monitor])
```

### Creating Instances From `URLSession`s
In addition to the `convenience` initializer mentioned previously, `Session`s can be initialized directly from `URLSession`s. However, there are several requirements to keep in mind when using this initializer, so using the convenience initializer is recommended. These include:
* Alamofire does not support `URLSession`s configured for background use. This will lead to a runtime error when the `Session` is initialized.
* A `SessionDelegate` instance must be created and used as the `URLSession`’s `delegate`, as well as passed to the `Session` initializer.
* A custom `OperationQueue` must be passed as the `URLSession`’s `delegateQueue`. This queue must be a serial queue, it must have a backing `DispatchQueue`, and that `DispatchQueue` must be passed to the `Session` as its `rootQueue`.

```swift
let rootQueue = DispatchQueue(label: "org.alamofire.customQueue")
let queue = OperationQueue()
queue.maxConcurrentOperationCount = 1
queue.underlyingQueue = rootQueue
let delegate = SessionDelegate()
let configuration = URLSessionConfiguration.af.default
let urlSession = URLSession(configuration: configuration,
                            delegate: delegate,
                            delegateQueue: queue)
let session = Session(session: urlSession, delegate: delegate, rootQueue: rootQueue)
```

## Requests
Each request performed by Alamofire is encapsulated by particular class, `DataRequest`, `UploadRequest`, and `DownloadRequest`. Each of these classes encapsulate functionality unique to each type of request, but `DataRequest` and `DownloadRequest` inherit from a common superclass, `Request` (`UploadRequest` inherits from `DataRequest`).

### `Request`
Although `Request` doesn’t encapsulate any particular type of request, it contains the state and functionality common to all requests Alamofire performs. This includes:

#### State
All `Request` types include the notion of state, indicating the major events in the `Request`’s lifetime.

```swift
public enum State {
    case initialized
    case resumed
    case suspended
    case cancelled
    case finished
}
```

`Request`s start in the `.initialized` state after their creation. `Request`s can be suspended, resumed, and cancelled by calling the appropriate lifetime method.
- `resume()` resumes, or starts, a `Request`’s network traffic. If `startRequestsImmediately` is `true`, this is called automatically once a response handler has been added to the `Request`.
- `suspend()` suspends, or pauses the `Request` and its network traffic. `Request`s in this state can be resumed, but only `DownloadRequests` may be able continue transferring data. Other `Request`s will start over.
- `cancel()` cancels a `Request`. Once in this state, a `Request` cannot be resumed or suspended. When `cancel()` is called, the `Request`’s `error` property will be set with an `AFError.explicitlyCancelled` instance.
If a `Request` is resumed and isn’t later cancelled, it will reach the `.finished` state once all response validators and response serializers have been run. However, if additional response serializers are added to the `Request` after it has reached the `.finished` state, it will transition back to the `.resumed` state and perform the network request again.

#### Progress
In order to track the progress of a request, `Request` offers a both  `uploadProgress` and `downloadProgress` properties as well as closure-based `uploadProgress` and `downloadProgress` methods. Like all closure-based `Request` APIs, the progress APIs can be chained off of the `Request`. Like the other closure-based APIs, they should be added to a request *before* calling any response handler, like `responseDecodable`.

```swift
AF.request(...)
    .uploadProgress { progress in
        print(progress)
    }
    .downloadProgress { progress in
        print(progress)
    }
    .responseDecodable(of: SomeType.self) { response in
        debugPrint(response)
    }
```
 
Importantly, not all `Request` subclasses are able to report their progress accurately, or may have other dependencies to do so.
- For upload progress, progress can be determined in the following ways:
	- The length of the `Data` object provided as the upload body to an `UploadRequest`.
	- The length of a file on disk provided as the upload body of an `UploadRequest`.
	- The value of the `Content-Length` header on the request, if it has been manually set.
- For download progress, there is a single requirement:
	- The server response must contain a `Content-Length` header.
Unfortunately there may be other, undocumented requirements for progress reporting from `URLSession` which prevents accurate progress reporting.

#### Handling Redirects
Alamofire’s `RedirectHandler` protocol provides control and customization of redirect handling for `Request`. In addition to per-`Session` `RedirectHandler`s, each `Request` can be given its own `RedirectHandler` which overrides any provided by the `Session`. 

```swift
let redirector = Redirector(behavior: .follow)
AF.request(...)
    .redirect(using: redirector)
    .responseDecodable(of: SomeType.self) { response in 
        debugPrint(response)
    }
```

> Note: Only one `RedirectHandler` can be set on a `Request`. Attempting to set more than one will result in a runtime exception.

#### Customizing Caching
Alamofire’s `CachedResponseHandler` protocol provides control and customization over the caching of responses. In addition to per-`Session` `CachedResponseHandler`s, each `Request` can be given its own `CachedResponseHandler` which overrides any provided by the `Session`.

```swift
let cacher = Cacher(behavior: .cache)
AF.request(...)
    .cacheResponse(using: cacher)
    .responseDecodable(of: SomeType.self) { response in 
        debugPrint(response)
    }
```

> Note: Only one `CachedResponseHandler` can be set on a `Request`. Attempting to set more than one will result in a runtime exception.

#### Credentials
In order to take advantage of the automatic credential handling provided by `URLSession`, Alamofire provides per-`Request` API to allow the automatic addition of `URLCredential` instances to requests. These include both convenience API for HTTP authentication using a username and password, as well as any `URLCredential` instance. 

Adding a credential to automatically reply to any HTTP authentication challenges is straightforward:
```swift
AF.request(...)
    .authenticate(username: "user@example.domain", password: "password")
    .responseDecodable(of: SomeType.self) { response in 
        debugPrint(response)
    }
```
> Note: This mechanism only supports HTTP authentication prompts. If a request requires an `Authentication` header for all requests, it should be provided directly.

Additionally, adding a raw `URLCredential` is just as easy:
```swift
let credential = URLCredential(...)
AF.request(...)
    .authenticate(using: credential)
    .responseDecodable(of: SomeType.self) { response in 
        debugPrint(response)
    }
```

#### A `Request`’s `URLRequest`s
Each network request issued by a `Request` is ultimately encapsulated in a `URLRequest` created from the various parameters passed to one of the `Session` request methods. `Request` will keep a copy of these `URLRequest` in its `requests` property.
#### Response

#### `URLSessionTask`s

#### Metrics

### `DataRequest`

### `UploadRequest`

### `DownloadRequest`

### Adapting and Retrying Requests

Most web services these days are behind some sort of authentication system. One of the more common ones today is OAuth. This generally involves generating an access token authorizing your application or user to call the various supported web services. While creating these initial access tokens can be laborious, it can be even more complicated when your access token expires and you need to fetch a new one. There are many thread-safety issues that need to be considered.

The `RequestAdapter` and `RequestRetrier` protocols were created to make it much easier to create a thread-safe authentication system for a specific set of web services.

#### RequestAdapter

The `RequestAdapter` protocol allows each `Request` made on a `SessionManager` to be inspected and adapted before being created. One very specific way to use an adapter is to append an `Authorization` header to requests behind a certain type of authentication.

```swift
class AccessTokenAdapter: RequestAdapter {
    private let accessToken: String

    init(accessToken: String) {
        self.accessToken = accessToken
    }

    func adapt(_ urlRequest: URLRequest) throws -> URLRequest {
        var urlRequest = urlRequest

        if let urlString = urlRequest.url?.absoluteString, urlString.hasPrefix("https://httpbin.org") {
            urlRequest.setValue("Bearer " + accessToken, forHTTPHeaderField: "Authorization")
        }

        return urlRequest
    }
}
```

```swift
let sessionManager = SessionManager()
sessionManager.adapter = AccessTokenAdapter(accessToken: "1234")

sessionManager.request("https://httpbin.org/get")
```

#### RequestRetrier

The `RequestRetrier` protocol allows a `Request` that encountered an `Error` while being executed to be retried. When using both the `RequestAdapter` and `RequestRetrier` protocols together, you can create credential refresh systems for OAuth1, OAuth2, Basic Auth and even exponential backoff retry policies. The possibilities are endless. Here's an example of how you could implement a refresh flow for OAuth2 access tokens.

> **DISCLAIMER:** This is **NOT** a global `OAuth2` solution. It is merely an example demonstrating how one could use the `RequestAdapter` in conjunction with the `RequestRetrier` to create a thread-safe refresh system.

> To reiterate, **do NOT copy** this sample code and drop it into a production application. This is merely an example. Each authentication system must be tailored to a particular platform and authentication type.

```swift
class OAuth2Handler: RequestAdapter, RequestRetrier {
    private typealias RefreshCompletion = (_ succeeded: Bool, _ accessToken: String?, _ refreshToken: String?) -> Void

    private let sessionManager: SessionManager = {
        let configuration = URLSessionConfiguration.default
        configuration.httpAdditionalHeaders = SessionManager.defaultHTTPHeaders

        return SessionManager(configuration: configuration)
    }()

    private let lock = NSLock()

    private var clientID: String
    private var baseURLString: String
    private var accessToken: String
    private var refreshToken: String

    private var isRefreshing = false
    private var requestsToRetry: [RequestRetryCompletion] = []

    // MARK: - Initialization

    public init(clientID: String, baseURLString: String, accessToken: String, refreshToken: String) {
        self.clientID = clientID
        self.baseURLString = baseURLString
        self.accessToken = accessToken
        self.refreshToken = refreshToken
    }

    // MARK: - RequestAdapter

    func adapt(_ urlRequest: URLRequest) throws -> URLRequest {
        if let urlString = urlRequest.url?.absoluteString, urlString.hasPrefix(baseURLString) {
            var urlRequest = urlRequest
            urlRequest.setValue("Bearer " + accessToken, forHTTPHeaderField: "Authorization")
            return urlRequest
        }

        return urlRequest
    }

    // MARK: - RequestRetrier

    func should(_ manager: SessionManager, retry request: Request, with error: Error, completion: @escaping RequestRetryCompletion) {
        lock.lock() ; defer { lock.unlock() }

        if let response = request.task?.response as? HTTPURLResponse, response.statusCode == 401 {
            requestsToRetry.append(completion)

            if !isRefreshing {
                refreshTokens { [weak self] succeeded, accessToken, refreshToken in
                    guard let strongSelf = self else { return }

                    strongSelf.lock.lock() ; defer { strongSelf.lock.unlock() }

                    if let accessToken = accessToken, let refreshToken = refreshToken {
                        strongSelf.accessToken = accessToken
                        strongSelf.refreshToken = refreshToken
                    }

                    strongSelf.requestsToRetry.forEach { $0(succeeded, 0.0) }
                    strongSelf.requestsToRetry.removeAll()
                }
            }
        } else {
            completion(false, 0.0)
        }
    }

    // MARK: - Private - Refresh Tokens

    private func refreshTokens(completion: @escaping RefreshCompletion) {
        guard !isRefreshing else { return }

        isRefreshing = true

        let urlString = "\(baseURLString)/oauth2/token"

        let parameters: [String: Any] = [
            "access_token": accessToken,
            "refresh_token": refreshToken,
            "client_id": clientID,
            "grant_type": "refresh_token"
        ]

        sessionManager.request(urlString, method: .post, parameters: parameters, encoding: JSONEncoding.default)
            .responseJSON { [weak self] response in
                guard let strongSelf = self else { return }

                if 
                    let json = response.result.value as? [String: Any], 
                    let accessToken = json["access_token"] as? String, 
                    let refreshToken = json["refresh_token"] as? String 
                {
                    completion(true, accessToken, refreshToken)
                } else {
                    completion(false, nil, nil)
                }

                strongSelf.isRefreshing = false
            }
    }
}
```

```swift
let baseURLString = "https://some.domain-behind-oauth2.com"

let oauthHandler = OAuth2Handler(
    clientID: "12345678",
    baseURLString: baseURLString,
    accessToken: "abcd1234",
    refreshToken: "ef56789a"
)

let sessionManager = SessionManager()
sessionManager.adapter = oauthHandler
sessionManager.retrier = oauthHandler

let urlString = "\(baseURLString)/some/endpoint"

sessionManager.request(urlString).validate().responseJSON { response in
    debugPrint(response)
}
```

Once the `OAuth2Handler` is applied as both the `adapter` and `retrier` for the `SessionManager`, it will handle an invalid access token error by automatically refreshing the access token and retrying all failed requests in the same order they failed.

> If you needed them to execute in the same order they were created, you could sort them by their task identifiers.

The example above only checks for a `401` response code which is not nearly robust enough, but does demonstrate how one could check for an invalid access token error. In a production application, one would want to check the `realm` and most likely the `www-authenticate` header response although it depends on the OAuth2 implementation.

Another important note is that this authentication system could be shared between multiple session managers. For example, you may need to use both a `default` and `ephemeral` session configuration for the same set of web services. The example above allows the same `oauthHandler` instance to be shared across multiple session managers to manage the single refresh flow.

### Security

Using a secure HTTPS connection when communicating with servers and web services is an important step in securing sensitive data. By default, Alamofire will evaluate the certificate chain provided by the server using Apple's built in validation provided by the Security framework. While this guarantees the certificate chain is valid, it does not prevent man-in-the-middle (MITM) attacks or other potential vulnerabilities. In order to mitigate MITM attacks, applications dealing with sensitive customer data or financial information should use certificate or public key pinning provided by the `ServerTrustPolicy`.

#### ServerTrustPolicy

The `ServerTrustPolicy` enumeration evaluates the server trust generally provided by an `URLAuthenticationChallenge` when connecting to a server over a secure HTTPS connection.

```swift
let serverTrustPolicy = ServerTrustPolicy.pinCertificates(
    certificates: ServerTrustPolicy.certificates(),
    validateCertificateChain: true,
    validateHost: true
)
```

There are many different cases of server trust evaluation giving you complete control over the validation process:

* `performDefaultEvaluation`: Uses the default server trust evaluation while allowing you to control whether to validate the host provided by the challenge.
* `pinCertificates`: Uses the pinned certificates to validate the server trust. The server trust is considered valid if one of the pinned certificates match one of the server certificates.
* `pinPublicKeys`: Uses the pinned public keys to validate the server trust. The server trust is considered valid if one of the pinned public keys match one of the server certificate public keys.
* `disableEvaluation`: Disables all evaluation which in turn will always consider any server trust as valid.
* `customEvaluation`: Uses the associated closure to evaluate the validity of the server trust thus giving you complete control over the validation process. Use with caution.

#### Server Trust Policy Manager

The `ServerTrustPolicyManager` is responsible for storing an internal mapping of server trust policies to a particular host. This allows Alamofire to evaluate each host against a different server trust policy.

```swift
let serverTrustPolicies: [String: ServerTrustPolicy] = [
    "test.example.com": .pinCertificates(
        certificates: ServerTrustPolicy.certificates(),
        validateCertificateChain: true,
        validateHost: true
    ),
    "insecure.expired-apis.com": .disableEvaluation
]

let sessionManager = SessionManager(
    serverTrustPolicyManager: ServerTrustPolicyManager(policies: serverTrustPolicies)
)
```

> Make sure to keep a reference to the new `SessionManager` instance, otherwise your requests will all get cancelled when your `sessionManager` is deallocated.

These server trust policies will result in the following behavior:

- `test.example.com` will always use certificate pinning with certificate chain and host validation enabled thus requiring the following criteria to be met to allow the TLS handshake to succeed:
	- Certificate chain MUST be valid.
	- Certificate chain MUST include one of the pinned certificates.
	- Challenge host MUST match the host in the certificate chain's leaf certificate.
- `insecure.expired-apis.com` will never evaluate the certificate chain and will always allow the TLS handshake to succeed.
- All other hosts will use the default evaluation provided by Apple.

##### Subclassing Server Trust Policy Manager

If you find yourself needing more flexible server trust policy matching behavior (i.e. wildcarded domains), then subclass the `ServerTrustPolicyManager` and override the `serverTrustPolicyForHost` method with your own custom implementation.

```swift
class CustomServerTrustPolicyManager: ServerTrustPolicyManager {
    override func serverTrustPolicy(forHost host: String) -> ServerTrustPolicy? {
        var policy: ServerTrustPolicy?

        // Implement your custom domain matching behavior...

        return policy
    }
}
```

#### Validating the Host

The `.performDefaultEvaluation`, `.pinCertificates` and `.pinPublicKeys` server trust policies all take a `validateHost` parameter. Setting the value to `true` will cause the server trust evaluation to verify that hostname in the certificate matches the hostname of the challenge. If they do not match, evaluation will fail. A `validateHost` value of `false` will still evaluate the full certificate chain, but will not validate the hostname of the leaf certificate.

> It is recommended that `validateHost` always be set to `true` in production environments.

#### Validating the Certificate Chain

Pinning certificates and public keys both have the option of validating the certificate chain using the `validateCertificateChain` parameter. By setting this value to `true`, the full certificate chain will be evaluated in addition to performing a byte equality check against the pinned certificates or public keys. A value of `false` will skip the certificate chain validation, but will still perform the byte equality check.

There are several cases where it may make sense to disable certificate chain validation. The most common use cases for disabling validation are self-signed and expired certificates. The evaluation would always fail in both of these cases, but the byte equality check will still ensure you are receiving the certificate you expect from the server.

> It is recommended that `validateCertificateChain` always be set to `true` in production environments.

#### App Transport Security

With the addition of App Transport Security (ATS) in iOS 9, it is possible that using a custom `ServerTrustPolicyManager` with several `ServerTrustPolicy` objects will have no effect. If you continuously see `CFNetwork SSLHandshake failed (-9806)` errors, you have probably run into this problem. Apple's ATS system overrides the entire challenge system unless you configure the ATS settings in your app's plist to disable enough of it to allow your app to evaluate the server trust.

If you run into this problem (high probability with self-signed certificates), you can work around this issue by adding the following to your `Info.plist`.

```xml
<dict>
    <key>NSAppTransportSecurity</key>
    <dict>
        <key>NSExceptionDomains</key>
        <dict>
            <key>example.com</key>
            <dict>
                <key>NSExceptionAllowsInsecureHTTPLoads</key>
                <true/>
                <key>NSExceptionRequiresForwardSecrecy</key>
                <false/>
                <key>NSIncludesSubdomains</key>
                <true/>
                <!-- Optional: Specify minimum TLS version -->
                <key>NSTemporaryExceptionMinimumTLSVersion</key>
                <string>TLSv1.2</string>
            </dict>
        </dict>
    </dict>
</dict>
```

Whether you need to set the `NSExceptionRequiresForwardSecrecy` to `NO` depends on whether your TLS connection is using an allowed cipher suite. In certain cases, it will need to be set to `NO`. The `NSExceptionAllowsInsecureHTTPLoads` MUST be set to `YES` in order to allow the `SessionDelegate` to receive challenge callbacks. Once the challenge callbacks are being called, the `ServerTrustPolicyManager` will take over the server trust evaluation. You may also need to specify the `NSTemporaryExceptionMinimumTLSVersion` if you're trying to connect to a host that only supports TLS versions less than `1.2`.

> It is recommended to always use valid certificates in production environments.

#### Using Self-Signed Certificates with Local Networking

If you are attempting to connect to a server running on your localhost, and you are using self-signed certificates, you will need to add the following to your `Info.plist`.

```xml
<dict>
    <key>NSAppTransportSecurity</key>
    <dict>
        <key>NSAllowsLocalNetworking</key>
        <true/>
    </dict>
</dict>
```

According to [Apple documentation](https://developer.apple.com/library/content/documentation/General/Reference/InfoPlistKeyReference/Articles/CocoaKeys.html#//apple_ref/doc/uid/TP40009251-SW35), setting `NSAllowsLocalNetworking` to `YES` allows loading of local resources without disabling ATS for the rest of your app.

### Network Reachability

The `NetworkReachabilityManager` listens for reachability changes of hosts and addresses for both WWAN and WiFi network interfaces.

```swift
let manager = NetworkReachabilityManager(host: "www.apple.com")

manager?.listener = { status in
    print("Network Status Changed: \(status)")
}

manager?.startListening()
```

> Make sure to remember to retain the `manager` in the above example, or no status changes will be reported.
> Also, do not include the scheme in the `host` string or reachability won't function correctly.

There are some important things to remember when using network reachability to determine what to do next.

- **Do NOT** use Reachability to determine if a network request should be sent.
	- You should **ALWAYS** send it.
- When Reachability is restored, use the event to retry failed network requests.
	- Even though the network requests may still fail, this is a good moment to retry them.
- The network reachability status can be useful for determining why a network request may have failed.
	- If a network request fails, it is more useful to tell the user that the network request failed due to being offline rather than a more technical error, such as "request timed out."

### Routing Requests

As apps grow in size, it's important to adopt common patterns as you build out your network stack. An important part of that design is how to route your requests. The Alamofire `URLConvertible` and `URLRequestConvertible` protocols along with the `Router` design pattern are here to help.

#### URLConvertible

Types adopting the `URLConvertible` protocol can be used to construct URLs, which are then used to construct URL requests internally. `String`, `URL`, and `URLComponents` conform to `URLConvertible` by default, allowing any of them to be passed as `url` parameters to the `request`, `upload`, and `download` methods:

```swift
let urlString = "https://httpbin.org/post"
Alamofire.request(urlString, method: .post)

let url = URL(string: urlString)!
Alamofire.request(url, method: .post)

let urlComponents = URLComponents(url: url, resolvingAgainstBaseURL: true)!
Alamofire.request(urlComponents, method: .post)
```

Applications interacting with web applications in a significant manner are encouraged to have custom types conform to `URLConvertible` as a convenient way to map domain-specific models to server resources.

##### Type-Safe Routing

```swift
extension User: URLConvertible {
    static let baseURLString = "https://example.com"

    func asURL() throws -> URL {
    	let urlString = User.baseURLString + "/users/\(username)/"
        return try urlString.asURL()
    }
}
```

```swift
let user = User(username: "mattt")
Alamofire.request(user) // https://example.com/users/mattt
```

#### URLRequestConvertible

Types adopting the `URLRequestConvertible` protocol can be used to construct URL requests. `URLRequest` conforms to `URLRequestConvertible` by default, allowing it to be passed into `request`, `upload`, and `download` methods directly (this is the recommended way to specify custom HTTP body for individual requests):

```swift
let url = URL(string: "https://httpbin.org/post")!
var urlRequest = URLRequest(url: url)
urlRequest.httpMethod = "POST"

let parameters = ["foo": "bar"]

do {
    urlRequest.httpBody = try JSONSerialization.data(withJSONObject: parameters, options: [])
} catch {
    // No-op
}

urlRequest.setValue("application/json", forHTTPHeaderField: "Content-Type")

Alamofire.request(urlRequest)
```

Applications interacting with web applications in a significant manner are encouraged to have custom types conform to `URLRequestConvertible` as a way to ensure consistency of requested endpoints. Such an approach can be used to abstract away server-side inconsistencies and provide type-safe routing, as well as manage authentication credentials and other state.

##### API Parameter Abstraction

```swift
enum Router: URLRequestConvertible {
    case search(query: String, page: Int)

    static let baseURLString = "https://example.com"
    static let perPage = 50

    // MARK: URLRequestConvertible

    func asURLRequest() throws -> URLRequest {
        let result: (path: String, parameters: Parameters) = {
            switch self {
            case let .search(query, page) where page > 0:
                return ("/search", ["q": query, "offset": Router.perPage * page])
            case let .search(query, _):
                return ("/search", ["q": query])
            }
        }()

        let url = try Router.baseURLString.asURL()
        let urlRequest = URLRequest(url: url.appendingPathComponent(result.path))

        return try URLEncoding.default.encode(urlRequest, with: result.parameters)
    }
}
```

```swift
Alamofire.request(Router.search(query: "foo bar", page: 1)) // https://example.com/search?q=foo%20bar&offset=50
```

##### CRUD & Authorization

```swift
import Alamofire

enum Router: URLRequestConvertible {
    case createUser(parameters: Parameters)
    case readUser(username: String)
    case updateUser(username: String, parameters: Parameters)
    case destroyUser(username: String)

    static let baseURLString = "https://example.com"

    var method: HTTPMethod {
        switch self {
        case .createUser:
            return .post
        case .readUser:
            return .get
        case .updateUser:
            return .put
        case .destroyUser:
            return .delete
        }
    }

    var path: String {
        switch self {
        case .createUser:
            return "/users"
        case .readUser(let username):
            return "/users/\(username)"
        case .updateUser(let username, _):
            return "/users/\(username)"
        case .destroyUser(let username):
            return "/users/\(username)"
        }
    }

    // MARK: URLRequestConvertible

    func asURLRequest() throws -> URLRequest {
    	let url = try Router.baseURLString.asURL()

        var urlRequest = URLRequest(url: url.appendingPathComponent(path))
        urlRequest.httpMethod = method.rawValue

        switch self {
        case .createUser(let parameters):
            urlRequest = try URLEncoding.default.encode(urlRequest, with: parameters)
        case .updateUser(_, let parameters):
            urlRequest = try URLEncoding.default.encode(urlRequest, with: parameters)
        default:
            break
        }

        return urlRequest
    }
}
```

```swift
Alamofire.request(Router.readUser("mattt")) // GET https://example.com/users/mattt
```

### Custom Response Serialization

Alamofire provides built-in response serialization for data, strings, JSON, and property lists:

```swift
Alamofire.request(...).responseData { (resp: DataResponse<Data>) in ... }
Alamofire.request(...).responseString { (resp: DataResponse<String>) in ... }
Alamofire.request(...).responseJSON { (resp: DataResponse<Any>) in ... }
Alamofire.request(...).responsePropertyList { (resp: DataResponse<Any>) in ... }
```

Those responses wrap deserialized *values* (Data, String, Any) or *errors* (network, validation errors), as well as *meta-data* (URL request, HTTP headers, status code, [metrics](#statistical-metrics), ...).

You have several ways to customize all of those response elements:

- [Response Mapping](#response-mapping)
- [Handling Errors](#handling-errors)
- [Creating a Custom Response Serializer](#creating-a-custom-response-serializer)
- [Generic Response Object Serialization](#generic-response-object-serialization)

#### Response Mapping

Response mapping is the simplest way to produce customized responses. It transforms the value of a response, while preserving eventual errors and meta-data. For example, you can turn a json response `DataResponse<Any>` into a response that holds an application model, such as `DataResponse<User>`. You perform response mapping with the `DataResponse.map` method:

```swift
Alamofire.request("https://example.com/users/mattt").responseJSON { (response: DataResponse<Any>) in
    let userResponse = response.map { json in
        // We assume an existing User(json: Any) initializer
        return User(json: json)
    }

    // Process userResponse, of type DataResponse<User>:
    if let user = userResponse.value {
        print("User: { username: \(user.username), name: \(user.name) }")
    }
}
```

When the transformation may throw an error, use `flatMap` instead:

```swift
Alamofire.request("https://example.com/users/mattt").responseJSON { response in
    let userResponse = response.flatMap { json in
        try User(json: json)
    }
}
```

Response mapping is a good fit for your custom completion handlers:

```swift
@discardableResult
func loadUser(completionHandler: @escaping (DataResponse<User>) -> Void) -> Alamofire.DataRequest {
    return Alamofire.request("https://example.com/users/mattt").responseJSON { response in
        let userResponse = response.flatMap { json in
            try User(json: json)
        }

        completionHandler(userResponse)
    }
}

loadUser { response in
    if let user = response.value {
        print("User: { username: \(user.username), name: \(user.name) }")
    }
}
```

When the map/flatMap closure may process a big amount of data, make sure you execute it outside of the main thread:

```swift
@discardableResult
func loadUser(completionHandler: @escaping (DataResponse<User>) -> Void) -> Alamofire.DataRequest {
    let utilityQueue = DispatchQueue.global(qos: .utility)

    return Alamofire.request("https://example.com/users/mattt").responseJSON(queue: utilityQueue) { response in
        let userResponse = response.flatMap { json in
            try User(json: json)
        }

        DispatchQueue.main.async {
            completionHandler(userResponse)
        }
    }
}
```

`map` and `flatMap` are also available for [download responses](#downloading-data-to-a-file).

#### Handling Errors

Before implementing custom response serializers or object serialization methods, it's important to consider how to handle any errors that may occur. There are two basic options: passing existing errors along unmodified, to be dealt with at response time; or, wrapping all errors in an `Error` type specific to your app.

For example, here's a simple `BackendError` enum which will be used in later examples:

```swift
enum BackendError: Error {
    case network(error: Error) // Capture any underlying Error from the URLSession API
    case dataSerialization(error: Error)
    case jsonSerialization(error: Error)
    case xmlSerialization(error: Error)
    case objectSerialization(reason: String)
}
```

#### Creating a Custom Response Serializer

Alamofire provides built-in response serialization for strings, JSON, and property lists, but others can be added in extensions on `Alamofire.DataRequest` and / or `Alamofire.DownloadRequest`.

For example, here's how a response handler using [Ono](https://github.com/mattt/Ono) might be implemented:

```swift
extension DataRequest {
    static func xmlResponseSerializer() -> DataResponseSerializer<ONOXMLDocument> {
        return DataResponseSerializer { request, response, data, error in
            // Pass through any underlying URLSession error to the .network case.
            guard error == nil else { return .failure(BackendError.network(error: error!)) }

            // Use Alamofire's existing data serializer to extract the data, passing the error as nil, as it has
            // already been handled.
            let result = Request.serializeResponseData(response: response, data: data, error: nil)

            guard case let .success(validData) = result else {
                return .failure(BackendError.dataSerialization(error: result.error! as! AFError))
            }

            do {
                let xml = try ONOXMLDocument(data: validData)
                return .success(xml)
            } catch {
                return .failure(BackendError.xmlSerialization(error: error))
            }
        }
    }

    @discardableResult
    func responseXMLDocument(
        queue: DispatchQueue? = nil,
        completionHandler: @escaping (DataResponse<ONOXMLDocument>) -> Void)
        -> Self
    {
        return response(
            queue: queue,
            responseSerializer: DataRequest.xmlResponseSerializer(),
            completionHandler: completionHandler
        )
    }
}
```

#### Generic Response Object Serialization

Generics can be used to provide automatic, type-safe response object serialization.

```swift
protocol ResponseObjectSerializable {
    init?(response: HTTPURLResponse, representation: Any)
}

extension DataRequest {
    func responseObject<T: ResponseObjectSerializable>(
        queue: DispatchQueue? = nil,
        completionHandler: @escaping (DataResponse<T>) -> Void)
        -> Self
    {
        let responseSerializer = DataResponseSerializer<T> { request, response, data, error in
            guard error == nil else { return .failure(BackendError.network(error: error!)) }

            let jsonResponseSerializer = DataRequest.jsonResponseSerializer(options: .allowFragments)
            let result = jsonResponseSerializer.serializeResponse(request, response, data, nil)

            guard case let .success(jsonObject) = result else {
                return .failure(BackendError.jsonSerialization(error: result.error!))
            }

            guard let response = response, let responseObject = T(response: response, representation: jsonObject) else {
                return .failure(BackendError.objectSerialization(reason: "JSON could not be serialized: \(jsonObject)"))
            }

            return .success(responseObject)
        }

        return response(queue: queue, responseSerializer: responseSerializer, completionHandler: completionHandler)
    }
}
```

```swift
struct User: ResponseObjectSerializable, CustomStringConvertible {
    let username: String
    let name: String

    var description: String {
        return "User: { username: \(username), name: \(name) }"
    }

    init?(response: HTTPURLResponse, representation: Any) {
        guard
            let username = response.url?.lastPathComponent,
            let representation = representation as? [String: Any],
            let name = representation["name"] as? String
        else { return nil }

        self.username = username
        self.name = name
    }
}
```

```swift
Alamofire.request("https://example.com/users/mattt").responseObject { (response: DataResponse<User>) in
    debugPrint(response)

    if let user = response.result.value {
        print("User: { username: \(user.username), name: \(user.name) }")
    }
}
```

The same approach can also be used to handle endpoints that return a representation of a collection of objects:

```swift
protocol ResponseCollectionSerializable {
    static func collection(from response: HTTPURLResponse, withRepresentation representation: Any) -> [Self]
}

extension ResponseCollectionSerializable where Self: ResponseObjectSerializable {
    static func collection(from response: HTTPURLResponse, withRepresentation representation: Any) -> [Self] {
        var collection: [Self] = []

        if let representation = representation as? [[String: Any]] {
            for itemRepresentation in representation {
                if let item = Self(response: response, representation: itemRepresentation) {
                    collection.append(item)
                }
            }
        }

        return collection
    }
}
```

```swift
extension DataRequest {
    @discardableResult
    func responseCollection<T: ResponseCollectionSerializable>(
        queue: DispatchQueue? = nil,
        completionHandler: @escaping (DataResponse<[T]>) -> Void) -> Self
    {
        let responseSerializer = DataResponseSerializer<[T]> { request, response, data, error in
            guard error == nil else { return .failure(BackendError.network(error: error!)) }

            let jsonSerializer = DataRequest.jsonResponseSerializer(options: .allowFragments)
            let result = jsonSerializer.serializeResponse(request, response, data, nil)

            guard case let .success(jsonObject) = result else {
                return .failure(BackendError.jsonSerialization(error: result.error!))
            }

            guard let response = response else {
                let reason = "Response collection could not be serialized due to nil response."
                return .failure(BackendError.objectSerialization(reason: reason))
            }

            return .success(T.collection(from: response, withRepresentation: jsonObject))
        }

        return response(responseSerializer: responseSerializer, completionHandler: completionHandler)
    }
}
```

```swift
struct User: ResponseObjectSerializable, ResponseCollectionSerializable, CustomStringConvertible {
    let username: String
    let name: String

    var description: String {
        return "User: { username: \(username), name: \(name) }"
    }

    init?(response: HTTPURLResponse, representation: Any) {
        guard
            let username = response.url?.lastPathComponent,
            let representation = representation as? [String: Any],
            let name = representation["name"] as? String
        else { return nil }

        self.username = username
        self.name = name
    }
}
```

```swift
Alamofire.request("https://example.com/users").responseCollection { (response: DataResponse<[User]>) in
    debugPrint(response)

    if let users = response.result.value {
        users.forEach { print("- \($0)") }
    }
}
```