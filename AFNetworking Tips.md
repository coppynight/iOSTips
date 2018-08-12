@(相关技术)[iOS, AFNetworking, 个人笔记]

在ATA上闲逛看到一篇介绍AFNetworking的[文章](https://www.atatech.org/articles/60311)，里面详细介绍了其结构，并简单浏览了源代码。本文的侧重点将放在AFNetworking中体现出的代码技巧与OC语言中一些有意思的事情。

这个库算是所有iOS开发第一个认识的亲人了，虽然日常开发工作中已经不用了，但里面还是有挺多有意思的地方（也可能是我代码写的太少了......），我将自己觉得有意思的地方整理了出来，与大家分享。

## Agenda
按照如下几个核心类整理了其中的奇技淫巧，#LineXX#代表该块代码出现的行数，本文描述的AFNetworking版本为3.1
+ URLSessionManager (HTTPSessionManager)
+ RequestSerialization
+ ResponseSerialization
+ ReachabilityManager && SecurityPolicy

## URLSessionManager

### `__unused`不用的参数做个标记 #Line250#
```objectivec
- (void)URLSession:(__unused NSURLSession *)session
              task:(NSURLSessionTask *)task
didCompleteWithError:(NSError *)error
```

### NSFoundationVersionNumber && iOS8 Sync Bug #Line41#

对于特定版本Bug的修复可以采用这种方式

```objectivec
static void url_session_manager_create_task_safely(dispatch_block_t block) {
    if (NSFoundationVersionNumber < NSFoundationVersionNumber_With_Fixed_5871104061079552_bug) {
        // Fix of bug
        // Open Radar:http://openradar.appspot.com/radar?id=5871104061079552 (status: Fixed in iOS8)
        // Issue about:https://github.com/AFNetworking/AFNetworking/issues/2093
        dispatch_sync(url_session_manager_creation_queue(), block);
    } else {
        block();
    }
}
```

### Class Cluster中的MethodSwizzle #Line381#

AFNetworking场景中需要给NSURLSessionTask的resume方法添加自定义的通知，c常见的思路是替换方法，加入自定义的通知事件。由于NSURLSessionTask在实现上是一个类簇，单纯的替换基类方法并不能解决问题。
在iOS7下，`NSURLSessionDataTask` -> `__NSCFLocalDataTask`->`__NSCFLocalSessionTask`->`__NSCFURLSessionTask`
在iOS8下，`NSURLSessionDataTask` -> `__NSCFLocalDataTask`->`__NSCFLocalSessionTask`->`NSURLSessionTask`
在iOS7下，`__NSCFLocalSessionTask`和`__NSCFURLSessionTask`是两个类，各自实现了`resume`与`suspend`方法，意味这两个类方法都需要替换
在iOS8下，只有`NSURLSessionTask`实现了`resume`与`suspend`，只需要替换一个类
大致思路就是选取继承链路上两个版本相同的类，此处选的是`NSURLSessionDataTask`，向上找父类，如果该类和父类的`resume`方法实现不同，并且没有被替换过，就替换掉当前类的`resume`方法，并指向父类继续找。

```objectivec
IMP originalAFResumeIMP = method_getImplementation(class_getInstanceMethod([self class], @selector(af_resume)));
Class currentClass = [localDataTask class];

while (class_getInstanceMethod(currentClass, @selector(resume))) {
Class superClass = [currentClass superclass];
IMP classResumeIMP = method_getImplementation(class_getInstanceMethod(currentClass, @selector(resume)));
IMP superclassResumeIMP = method_getImplementation(class_getInstanceMethod(superClass, @selector(resume)));
if (classResumeIMP != superclassResumeIMP &&
originalAFResumeIMP != classResumeIMP) {
	[self swizzleResumeAndSuspendMethodForClass:currentClass];
            }
	currentClass = [currentClass superclass];
}
```

## RequestSerialization

### 指针的小技巧 #Line187#

AFNetworking中大量使用了通知来处理状态变化，怎样让不同类的通知不相互影响，每个类只只处理自己发送的通知呢，使用了C语言中的一些小技巧
```objectivec
/// 类似一个上下文的hook，很想Fomulae中传递的objectId
static void *AFHTTPRequestSerializerObserverContext = &AFHTTPRequestSerializerObserverContext;

/// OC本身作为C语言的超集，任何C语言的技巧都可以很好的体现
[self addObserver:self forKeyPath:keyPath options:NSKeyValueObservingOptionNew context:AFHTTPRequestSerializerObserverContext];

```

### 解析URL参数时的一处bug #Line502#

在阅读代码时，发现了解析URL参数时的Bug
https://github.com/AFNetworking/AFNetworking/pull/3944
```objectivec
/// URL.query在传入“？”结尾的url时，值为“”，按照目前的逻辑会直接拼接成“?&key=value”，造成参数错误
if (query && query.length > 0) {
    mutableRequest.URL = [NSURL URLWithString:[[mutableRequest.URL absoluteString] stringByAppendingFormat:mutableRequest.URL.query ? @"&%@" : @"?%@", query]];
}
```

### 正确解析包含特殊字符的Url#Line35#

里面还涉及到正确遍历`NSString`的方法，不要去遍历每一个字符，而是应该遍历每一个subString，因为有某一些字符如emoji是由多个字符组成的，核心方法`rangeOfComposedCharacterSequencesForRange`

```objectivec
NSString * AFPercentEscapedStringFromString(NSString *string) {
    static NSString * const kAFCharactersGeneralDelimitersToEncode = @":#[]@"; // does not include "?" or "/" due to RFC 3986 - Section 3.4
    static NSString * const kAFCharactersSubDelimitersToEncode = @"!$&'()*+,;=";

    NSMutableCharacterSet * allowedCharacterSet = [[NSCharacterSet URLQueryAllowedCharacterSet] mutableCopy];
    [allowedCharacterSet removeCharactersInString:[kAFCharactersGeneralDelimitersToEncode stringByAppendingString:kAFCharactersSubDelimitersToEncode]];

	// FIXME: https://github.com/AFNetworking/AFNetworking/pull/3028
    // return [string stringByAddingPercentEncodingWithAllowedCharacters:allowedCharacterSet];

    static NSUInteger const batchSize = 50;

    NSUInteger index = 0;
    NSMutableString *escaped = @"".mutableCopy;

    while (index < string.length) {
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wgnu"
        NSUInteger length = MIN(string.length - index, batchSize);
#pragma GCC diagnostic pop
        NSRange range = NSMakeRange(index, length);

        // To avoid breaking up character sequences such as 👴🏻👮🏽
        range = [string rangeOfComposedCharacterSequencesForRange:range];

        NSString *substring = [string substringWithRange:range];
        NSString *encoded = [substring stringByAddingPercentEncodingWithAllowedCharacters:allowedCharacterSet];
        [escaped appendString:encoded];

        index += range.length;
    }

	return escaped;
}
```

### 帮你设置好的UserAgent #Line220#
看看对前端同事很重要的UA吧
```objectivec
NSString *userAgent = nil;
#if TARGET_OS_IOS
// User-Agent Header; see http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.43
userAgent = [NSString stringWithFormat:@"%@/%@ (%@; iOS %@; Scale/%0.2f)", [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleExecutableKey] ?: [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleIdentifierKey], [[NSBundle mainBundle] infoDictionary][@"CFBundleShortVersionString"] ?: [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleVersionKey], [[UIDevice currentDevice] model], [[UIDevice currentDevice] systemVersion], [[UIScreen mainScreen] scale]];
....// 非iOS的被我删了
if (userAgent) {
    ...
    [self setValue:userAgent forHTTPHeaderField:@"User-Agent"];
}
```

### 数据流的使用 #Line418 && Line842#
陌生的`NSInputStrea`与`NSOutputStream`，可以看出是受runloopmode影响的，发送数据文件时可以尝试
```objectivec
    [inputStream scheduleInRunLoop:[NSRunLoop currentRunLoop] forMode:NSDefaultRunLoopMode];
    [outputStream scheduleInRunLoop:[NSRunLoop currentRunLoop] forMode:NSDefaultRunLoopMode];
    [inputStream open];
    [outputStream open];
    while ([inputStream hasBytesAvailable] && [outputStream hasSpaceAvailable]) {
        ...// append data
    }
    [outputStream close];
    [inputStream close];
    ...
```
### 自定义Stream需要复写一些关键方法，详见#Line842#
```objectivec
- (NSInteger)read:(uint8_t *)buffer maxLength:(NSUInteger)len;
- (BOOL)getBuffer:(uint8_t * _Nullable * _Nonnull)buffer length:(NSUInteger *)len;
```

### 编写自定义的RequestSerializer #Line1254#

```objectivec
if (![mutableRequest valueForHTTPHeaderField:@"Content-Type"]) {
    [mutableRequest setValue:@"application/json" forHTTPHeaderField:@"Content-Type"];
}
```

## ResponseSerialization
占个坑，说实话没request那么有意思了

## ReachabilityManager && SecurityPolicy
占坑
ReachabilityManager不写了
SecurityPolicy还是很有意思的
