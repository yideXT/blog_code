---
title: iOS HTTPS同步请求数据
date: 2016-01-02 12:20:04
tags: [iOS]
---

### iOS HTTPS同步请求数据

我们平时用的比较多的iOS请求框架就是AFNetworking和NSURLSession、NSURLConnection等。一般请求数据都推荐使用异步请求，这样可以提升程序运行效率。但是有些特殊的情况，我们需要请求到数据之后再执行下一步操作，这时候我们就需要使用同步请求。在iOS7之前，我们可以使用NSURLConnection方法:
```
[NSURLConnection sendSynchronousRequest: returningResponse: error:]
```
但是https请求就不能使用这个方法了，因为它不支持证书的校验。我们可以使用NSURLSession,使用信号量来控制其进行同步请求。

##### 创建https请求
```
首先让对象遵守NSURLSessionDelegate协议,然后把session的delegate置为self。

//创建一个信号量,信号值为0
    dispatch_semaphore_t sema = dispatch_semaphore_create(0);
    
    __block NSData *responseData;
    //创建一个https请求任务
    NSURLSessionConfiguration *configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
    NSURLSession *session = [NSURLSession sessionWithConfiguration:configuration delegate:self delegateQueue:[NSOperationQueue mainQueue]];
    NSURLRequest *request = [NSURLRequest requestWithURL:@"https://xxxx/xxxx"];
    NSURLSessionDataTask *task = [session dataTaskWithURL:request completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
        responseData = data;
        //发送一个信号，信号总量+1;
        dispatch_semaphore_signal(sema);
    }];
    
    //(DISPATCH_TIME_FOREVER)线程一直等待，直到信号量大于0才执行下面的代码;
    dispatch_semaphore_wait(sema, DISPATCH_TIME_FOREVER);

    //执行下面的代码,处理数据......
    NSLog(@"responseData === %@",responseData);
```

##### https服务器证书验证
```
//Session层次收到了授权，证书等问题
-(void)URLSession:(NSURLSession *)session didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential *credential))completionHandler {
    
    //获取服务器的trust object
    SecTrustRef serverTrust = challenge.protectionSpace.serverTrust;
    
    //如果服务器没有正式的证书，默认认证成功，创建一个凭证返回给服务器
    NSURLSessionAuthChallengeDisposition disposition = NSURLSessionAuthChallengeUseCredential;
    NSURLCredential *credential = [NSURLCredential credentialForTrust:serverTrust];
    //回调凭证，传递给服务器
    if(completionHandler){
        completionHandler(disposition, credential);
    }
}

//task层次收到了授权，证书等问题
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential *credential))completionHandler {
    
    //获取服务器的trust object
    SecTrustRef serverTrust = challenge.protectionSpace.serverTrust;
    
    //如果服务器没有正式的证书，默认认证成功，创建一个凭证返回给服务器
    NSURLSessionAuthChallengeDisposition disposition = NSURLSessionAuthChallengeUseCredential;
    NSURLCredential *credential = [NSURLCredential credentialForTrust:serverTrust];
    //回调凭证，传递给服务器
    if(completionHandler){
        completionHandler(disposition, credential);
    }
}
```

这样之后，就能够使一个异步请求变成同步了。
