---
title: HTTPS(SSL/TLS)握手过程及AFNetWorking对HTTPS握手的实现
date: 2018-03-19 11:51:40
description: 
categories: https
tags:
---

### HTTPS(SSL/TLS)握手过程
整个SSL握手过程就是客户端,服务器端双方验证身份,以及生成对称加密密钥的过程.里面涉及到数字证书,对称加密、RSA非对称加密等一些知识.网上这方面的介绍也挺多的.可以参考[SSL/TLS协议运行机制的概述](http://www.ruanyifeng.com/blog/2014/02/ssl_tls.html)

详细过程:
![SSL握手过程](http://www.ruanyifeng.com/blogimg/asset/201402/bg2014020502.png)


#### 预备知识
* 各种加解密算法如对称加密、RSA非对称加密,摘要算法.
* 数字证书


### AFNetWorking对HTTPS握手过程的实现
AFNetWorking在HTTPS握手过程中,只是帮助我们完成了对服务器端证书的验证.

> AF的默认安全策略  

self.securityPolicy = [AFSecurityPolicy defaultPolicy];  
SSLPinningMode默认AFSSLPinningModeNone.  
validatesDomainName默认YES,验证域名.  
allowInvalidCertificates默认NO.不允许无效证书.

在进行SSL握手时,系统会调用代理方法`URLSession:didReceiveChallenge:completionHandler:`.AFNetWorking在该代理方法中已经帮我们做了默认的处理,如果需要自定义处理则只需要给`sessionDidReceiveAuthenticationChallenge`Block赋值即可.

```
- (void)URLSession:(NSURLSession *)session
didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge
 completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential *credential))completionHandler
{
    NSURLSessionAuthChallengeDisposition disposition = NSURLSessionAuthChallengePerformDefaultHandling;
    __block NSURLCredential *credential = nil;

    if (self.sessionDidReceiveAuthenticationChallenge) {
        disposition = self.sessionDidReceiveAuthenticationChallenge(session, challenge, &credential);
    } else {
        if ([challenge.protectionSpace.authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust]) {
            if ([self.securityPolicy evaluateServerTrust:challenge.protectionSpace.serverTrust forDomain:challenge.protectionSpace.host]) {
                credential = [NSURLCredential credentialForTrust:challenge.protectionSpace.serverTrust];
                if (credential) {
                    disposition = NSURLSessionAuthChallengeUseCredential;
                } else {
                    disposition = NSURLSessionAuthChallengePerformDefaultHandling;
                }
            } else {
                disposition = NSURLSessionAuthChallengeCancelAuthenticationChallenge;
            }
        } else {
            disposition = NSURLSessionAuthChallengePerformDefaultHandling;
        }
    }

    if (completionHandler) {
        completionHandler(disposition, credential);
    }
}
```
#### 证书的验证
先看一下SSLPinningMode的三种模式:

```
typedef NS_ENUM(NSUInteger, AFSSLPinningMode) {
    AFSSLPinningModeNone,
    AFSSLPinningModePublicKey,
    AFSSLPinningModeCertificate,
};
```
AFSSLPinningModeNone:不使用pinned证书.  
该模式下是否信任服务器端取决于allowInvalidCertificates=YES,或者SecTrustEvaluate()函数对证书验证是否通过.
  
AFSSLPinningModePublicKey:验证PublicKey.使用工程里的证书和服务器端返回的证书对比PublicKey是否一致.

AFSSLPinningModeCertificate:验证证书.使用工程里的证书和服务器端返回的证书对比二者是否一致.比上述只验证PublicKey要更加严格.

先解释下pinnedCertificates,它表示的是我们拖入工程里的证书.可能是自签证书,也有可能是CA权威机构颁发的证书.

AF主要帮我们实现了对服务器端证书的验证.具体的验证过程在方法`evaluateServerTrust: forDomain:`里.该方法会继续调用系统Security框架里的方法对证书进行验证.

先来看下`evaluateServerTrust: forDomain:`的部分实现:

```
if (self.SSLPinningMode == AFSSLPinningModeNone) {
    return self.allowInvalidCertificates || AFServerTrustIsValid(serverTrust);
} else if (!AFServerTrustIsValid(serverTrust) && !self.allowInvalidCertificates) {
    return NO;
}
```
如果SSLPinningMode等于AFSSLPinningModeNone.当allowInvalidCertificates=YES则表明证书验证通过,或者`AFServerTrustIsValid()`验证通过也表明证书验证通过.

`AFServerTrustIsValid()`的实现如下: 

```
static BOOL AFServerTrustIsValid(SecTrustRef serverTrust) {
    BOOL isValid = NO;
    SecTrustResultType result;
    __Require_noErr_Quiet(SecTrustEvaluate(serverTrust, &result), _out);

    isValid = (result == kSecTrustResultUnspecified || result == kSecTrustResultProceed);

_out:
    return isValid;
}
```
它调用的是系统的函数
`OSStatus SecTrustEvaluate(SecTrustRef trust, SecTrustResultType * __nullable result)`.

笔者发现在iOS<10.3.3时(iOS10.2.1 iPhone6),该函数对中间人Charles的证书验证结果是kSecTrustResultUnspecified即通过验证,而iOS>=10.3.3时验证结果是kSecTrustResultRecoverableTrustFailure验证失败.

接下来就是对AFSSLPinningModeCertificate和AFSSLPinningModePublicKey两种模式的处理:

```
switch (self.SSLPinningMode) {
        case AFSSLPinningModeNone:
        default:
            return NO;
        case AFSSLPinningModeCertificate: {
            NSMutableArray *pinnedCertificates = [NSMutableArray array];
            for (NSData *certificateData in self.pinnedCertificates) {
                [pinnedCertificates addObject:(__bridge_transfer id)SecCertificateCreateWithData(NULL, (__bridge CFDataRef)certificateData)];
            }
            SecTrustSetAnchorCertificates(serverTrust, (__bridge CFArrayRef)pinnedCertificates);

            if (!AFServerTrustIsValid(serverTrust)) {
                return NO;
            }

            // obtain the chain after being validated, which *should* contain the pinned certificate in the last position (if it's the Root CA)
            NSArray *serverCertificates = AFCertificateTrustChainForServerTrust(serverTrust);
            
            for (NSData *trustChainCertificate in [serverCertificates reverseObjectEnumerator]) {
                if ([self.pinnedCertificates containsObject:trustChainCertificate]) {
                    return YES;
                }
            }
            
            return NO;
        }
        case AFSSLPinningModePublicKey: {
            NSUInteger trustedPublicKeyCount = 0;
            NSArray *publicKeys = AFPublicKeyTrustChainForServerTrust(serverTrust);

            for (id trustChainPublicKey in publicKeys) {
                for (id pinnedPublicKey in self.pinnedPublicKeys) {
                    if (AFSecKeyIsEqualToKey((__bridge SecKeyRef)trustChainPublicKey, (__bridge SecKeyRef)pinnedPublicKey)) {
                        trustedPublicKeyCount += 1;
                    }
                }
            }
            return trustedPublicKeyCount > 0;
        }
    }
```
#### AFSSLPinningModeCertificate模式处理
由于有可能使用的是自签证书,所以先将工程里的证书设置为锚点证书.然后调用SecTrustEvaluate()函数验证服务器端的证书,验证通过后继续验证证书链.如果证书链中包含pinnedCertificates证书则验证通过.否则验证失败.

#### AFSSLPinningModePublicKey模式处理
pinnedPublicKeys和服务器端返回证书(若被中间人攻击这里的证书其实是中间人的证书)的PublicKey比较.

最后验证完成.

### 抓包HTTPS数据
通过上述分析,想要抓包iPhone上APP的HTTPS数据,若APP使用AFNetworking,则在iOS<10.3.3下,AFNetworking的SSLPinningMode需要设置为AFSSLPinningModeNone,就可以抓到.

在iOS>=10.3.3,则需要SSLPinningMode需要设置为AFSSLPinningModeNone,且allowInvalidCertificates=YES,且validatesDomainName=NO,且info.plist需要设置NSAllowsArbitraryLoads=YES,才能使用中间人攻击的方式抓到数据.

在使用HTTPS,iOS>=10.3.3时,系统验证服务端证书是根据系统安装的CA证书进行验证的,对于自签证书默认验证是失败的.因此对于自签证书的验证需要调用SecTrustSetAnchorCertificates()设置锚点证书后再进行验证.

伪代码:  

```
NSString * cerPath = [[NSBundle mainBundle] pathForResource:@"Charles Proxy CA" ofType:@"cer"]; //证书的路径
NSData * cerData = [NSData dataWithContentsOfFile:cerPath];
SecCertificateRef certificate = SecCertificateCreateWithData(NULL, (__bridge CFDataRef)(cerData));
if (certificate != NULL) {
    self.trustedCertificates = @[CFBridgingRelease(certificate)];
}
... 
...

SecTrustRef trust = challenge.protectionSpace.serverTrust;
SecTrustResultType result;
SecTrustSetAnchorCertificates(trust, (__bridge CFArrayRef)self.trustedCertificates);
OSStatus status = SecTrustEvaluate(trust, &result);
if (status == errSecSuccess &&
        (result == kSecTrustResultProceed ||
         result == kSecTrustResultUnspecified)) {

    //验证成功，生成NSURLCredential凭证cred，告知challenge使用这个凭证来继续连接
    NSURLCredential *credential = [NSURLCredential credentialForTrust:challenge.protectionSpace.serverTrust];
    completionHandler(NSURLSessionAuthChallengeUseCredential, credential);
} else {
    //验证失败，取消这次验证流程
completionHandler(NSURLSessionAuthChallengeCancelAuthenticationChallenge, nil);
}
```
打印SecCertificateRef对象:

```
NSString * cerPath = [[NSBundle mainBundle] pathForResource:@"Charles Proxy CA" ofType:@"cer"]; //证书的路径
NSData * cerData = [NSData dataWithContentsOfFile:cerPath];
SecCertificateRef certificate = SecCertificateCreateWithData(NULL, (__bridge CFDataRef)(cerData));
```
这样创建出来的SecCertificateRef证书对象,打印日志为:

```
<cert(0x7ff365c07d20) s: Charles Proxy CA (25 九月 2016, appledeMacBook-Pro.local) i: Charles Proxy CA (25 九月 2016, appledeMacBook-Pro.local)>
```

### 适配ATS
The NSAppTransportSecurity key is supported in iOS 9.0 and later and in macOS 10.11 and later, and is available in both apps and app extensions.

Starting in iOS 10.0 and later and in macOS 10.12 and later, the following subkeys are supported:

* NSAllowsArbitraryLoadsForMedia

* NSAllowsArbitraryLoadsInWebContent

* NSRequiresCertificateTransparency

* NSAllowsLocalNetworking

#### ATS Configuration Basics
App Transport Security (ATS) is enabled by default for apps linked against the iOS 9.0 or macOS 10.11 SDKs or later, as indicated by the default Boolean value of NO for the NSAllowsArbitraryLoads key. This key is at the root level of the NSAppTransportSecurity dictionary.

ATS在iOS9.0 SDK及以后是默认开启的,即NSAllowsArbitraryLoads的值默认是NO.
通俗一点就是:iOS9以后,APP将访问不了HTTP资源.比如你想加载图片`http://sjbz.fd.zol-img.com.cn/t_s1080x1920c/g5/M00/0C/05/ChMkJlp5PRmIHSpjAAIImNBCzd8AAkr9gNyC5sAAgiw300.jpg`,但是系统会提示:

```
App Transport Security has blocked a cleartext HTTP (http://) resource load since it is insecure. Temporary exceptions can be configured via your app's Info.plist file.
Error Domain=NSURLErrorDomain Code=-1022 "The resource could not be loaded because the App Transport Security policy requires the use of a secure connection." 
```

如果APP内有需要访问HTTP资源的,有两种办法:

1. 在info.plist文件中设置NSAllowsArbitraryLoads=YES.
2. 在info.plist文件中添加白名单,即设置Exception Domains.

	```
	<dict>
	<key>sjbz.fd.zol-img.com.cn</key>
	<dict>
		<key>NSIncludesSubdomains</key>
		<true/>
		<key>NSTemporaryExceptionAllowsInsecureHTTPLoads</key>
		<true/>
		<key>NSTemporaryExceptionMinimumTLSVersion</key>
		<string>TLSv1.1</string>
	</dict>
	</dict>
	```

完整NSAppTransportSecurity字典结构如下:

```
NSAppTransportSecurity : Dictionary {

    NSAllowsArbitraryLoads : Boolean

    NSAllowsArbitraryLoadsForMedia : Boolean

    NSAllowsArbitraryLoadsInWebContent : Boolean

    NSAllowsLocalNetworking : Boolean

    NSExceptionDomains : Dictionary {

        <domain-name-string> : Dictionary {

            NSIncludesSubdomains : Boolean

            NSExceptionAllowsInsecureHTTPLoads : Boolean

            NSExceptionMinimumTLSVersion : String

            NSExceptionRequiresForwardSecrecy : Boolean   // Default value is YES

            NSRequiresCertificateTransparency : Boolean

        }

    }

}
```


### 通过抓包软件观察实际SSL握手情况
待续...


### 参考
[SSL/TLS协议运行机制的概述](http://www.ruanyifeng.com/blog/2014/02/ssl_tls.html)


[iOS安全系列之一：HTTPS](http://oncenote.com/2014/10/21/Security-1-HTTPS/)

[iOS应用网络安全之HTTPS](http://io.diveinedu.com/2016/01/09/iOS应用网络安全之HTTPS.html)

[AFSecurityPolicy 网络安全策略（五）](http://piglikeyoung.com/2017/01/26/AFNetworking-AFSecurityPolicy-5/)

[SSL/TLS 握手过程详解](https://www.jianshu.com/p/7158568e4867)

[NSAppTransportSecurity Reference](https://developer.apple.com/library/content/documentation/General/Reference/InfoPlistKeyReference/Articles/CocoaKeys.html#//apple_ref/doc/uid/TP40009251-SW33)

