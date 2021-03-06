---
layout: post
title:  "上传/下载"
date:   2015-02-10 10:37:30
categories: jekyll update
---

使用`NSURLSession`进行上传和下载任务，`NSURLSession`是iOS7之后推出的,具有`downLoadTask`和`upLoadTask`功能，实现任务时，会先将任务挂起，这样可以实现断点续传功能，使用时候要注意`resume`

###下载
> * 最简单的下载

{% highlight objc %}
- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event
{
    // 1. URL
    NSString *urlString = @"http://192.168.26.201/0805空缺数据 2.xls.zip";
    // 添加 % 转义
    urlString = [urlString stringByAddingPercentEscapesUsingEncoding:NSUTF8StringEncoding];
    NSURL *url = [NSURL URLWithString:urlString];
    
    // NSURLSession,sharedSession是一个全局共享的单例,苹果为了方便程序员使用简单的网络任务
//    NSURLSession *session = [NSURLSession sharedSession];
    
    // 2. 由sharedSession发起网络会话任务
    [[[NSURLSession sharedSession] downloadTaskWithURL:url completionHandler:^(NSURL *location, NSURLResponse *response, NSError *error) {
        
        NSLog(@"%@", [NSBundle mainBundle].bundlePath);
        NSLog(@"%@", location);
    }] resume];
}
{% endhighlight %}

> * 下载解压缩

{% highlight objc %}
- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event
{
    NSString *urlString = @"http://192.168.26.201/0805空缺数据 2.xls.zip";
    // 添加 % 转义
    urlString = [urlString stringByAddingPercentEscapesUsingEncoding:NSUTF8StringEncoding];
    
    NSURL *url = [NSURL URLWithString:urlString];
    
    // NSURLSession,sharedSession是一个全局共享的单例,苹果为了方便程序员使用简单的网络任务
    NSURLSession *session = [NSURLSession sharedSession];
    
    // 由session发起下载任务,在URLSession中,所有的任务都是由session发起的
    // 下载任务会保存在tmp文件夹中,如果没有处理,下载的文件会被删除!
    /**
     从网络上下载小说
     
     1. 下载这个小说的zip
     2. 下载完成之后,可以解压缩,原有的zip包就不需要了
     3. URLSession会自动帮我们删除这个zip包
     但是如果调用了这个方法不会调用相应的代理方法,原因不详,望赐教
     */
    NSURLSessionDownloadTask *task = [session downloadTaskWithURL:url completionHandler:^(NSURL *location, NSURLResponse *response, NSError *error) {
        
        NSLog(@"%@", [NSBundle mainBundle].bundlePath);
        NSLog(@"%@", location);
        
        // 将下载的文件解压缩到cache目录 - 重新启动不会删除
        NSString *cacheDir = [NSSearchPathForDirectoriesInDomains(NSCachesDirectory, NSUserDomainMask, YES) lastObject];
        
        // 解压缩 path属性可以拿到URL中对应的路径
        //使用了解压缩框架ZipArchive
        [SSZipArchive unzipFileAtPath:location.path toDestination:cacheDir];
    }];
    
    // 所有的任务,默认都是挂起的,需要resume
    [task resume];
}
{% endhighlight %}

> * 三个代理方法

{% highlight objc %}
#pragma mark - NSURLSession下载代理方法
/** 下载完成 */
- (void)URLSession:(NSURLSession *)session downloadTask:(NSURLSessionDownloadTask *)downloadTask didFinishDownloadingToURL:(NSURL *)location
{
    //这个操作本身是在异步线程中完成的，如果想要在这个线程中完成下载解压操作，需要在异步线程完成。但是更新UI的操作如collectionView的reloadData要放到主线程去进行
    //如果要在这个方法中使用下载解压操作，要放到
    NSLog(@"下载完成 %@", location);
}

/** 断点续传使用 */
- (void)URLSession:(NSURLSession *)session downloadTask:(NSURLSessionDownloadTask *)downloadTask didResumeAtOffset:(int64_t)fileOffset expectedTotalBytes:(int64_t)expectedTotalBytes
{
    // 什么也不写
}

/**
 bytesWritten                   本次下载的字节数
 totalBytesWritten              已经下载的字节数
 totalBytesExpectedToWrite      下载文件总字节数
 */
- (void)URLSession:(NSURLSession *)session downloadTask:(NSURLSessionDownloadTask *)downloadTask didWriteData:(int64_t)bytesWritten totalBytesWritten:(int64_t)totalBytesWritten totalBytesExpectedToWrite:(int64_t)totalBytesExpectedToWrite
{
    // 进度 = 已经下载 / 总大小
    float progress = (float)totalBytesWritten / totalBytesExpectedToWrite;
    
    NSLog(@"%f %@", progress, [NSThread currentThread]);
    
    // 主线程更新UI GCD
//    dispatch_async(dispatch_get_main_queue(), ^ { self.progressView.progress = progress;});
    // NSOperation
[[NSOperationQueue mainQueue] addOperationWithBlock: ^ { 
        //进度条
        self.pView.progress = progress;
    }];
}
{% endhighlight %}

> * 断点续传

主要的是关于任务开始和释放
(注意)

使用下面这个办法注册`NSURLSession`对象时,需要注意到队列,可以使用主队列,这样下载任务只在主队列进行,使用自己定义的队列或者使用`nil`的时候,为异步并发队列

{% highlight objc %}
+ (NSURLSession *)sessionWithConfiguration:(NSURLSessionConfiguration *)configuration delegate:(id <NSURLSessionDelegate>)delegate delegateQueue:(NSOperationQueue *)queue;
{% endhighlight %}

{% highlight objc %}
- (NSURLSession *)session
{
    if (_session == nil) {
        NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
        _session = [NSURLSession sessionWithConfiguration:config delegate:self delegateQueue:[[NSOperationQueue alloc] init]];
    }
    return _session;
}

// 开始
- (IBAction)start
{
    self.pView.progress = 0.0f;
    
    // 1. URL
    NSString *urlString = @"http://192.168.26.201/2014苹果发布者大会.mp4";
    urlString = [urlString stringByAddingPercentEscapesUsingEncoding:NSUTF8StringEncoding];
    NSURL *url = [NSURL URLWithString:urlString];
    
    // 2. session
//    NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
//    NSURLSession *session = [NSURLSession sessionWithConfiguration:config delegate:self delegateQueue:[[NSOperationQueue alloc] init]];
    
    // 3. 下载任务
    self.downloadTask = [self.session downloadTaskWithURL:url];
    [self.downloadTask resume];
}

// 暂停
- (IBAction)pause
{
    // 如果已经暂停,应该不能再一次暂停
    NSLog(@"%s", __func__);
    
    [self.downloadTask cancelByProducingResumeData:^(NSData *resumeData) {
        NSLog(@"====> %zd", resumeData.length);
        // 需要把resumeData保存下来,才能保证后续的续传能够执行
        self.resumeData = resumeData;
        
        // 释放下载任务!
        self.downloadTask = nil;
    }];
}

// 继续
- (IBAction)resume
{
    // 如果已经继续,应该不能再次继续
    if (self.resumeData == nil) {
        // 说明前一次没有被暂停的下载任务
        return;
    }
    
    // 发起续传的网络下载任务
//    [[self.session downloadTaskWithResumeData:self.resumeData] resume];
    self.downloadTask = [self.session downloadTaskWithResumeData:self.resumeData];
    
    // 释放resumeData
    self.resumeData = nil;
    
    [self.downloadTask resume];
}
{% endhighlight %}


> * PUT上传、删除文件

{% highlight objc %}
@implementation CZViewController
/**
 
 PUT 方法可以上传任意大小的文件
 
 // 设置HTTP请求头属性
 // ****** 注意:不同的网络环境变化,通常都是设置HTTP的请求头字段,通过请求头字段可以告诉服务器很多内容!
 // 譬如:上传文件,授权信息等等,具体如何设置,公司的后端程序员会告诉前端
 
 1. 需要身份验证
 NSString *authStr = @"admin:123456";
 NSString *authBase64 = [@"BASIC " stringByAppendingString:[self base64:authStr]];
 [request setValue:authBase64 forHTTPHeaderField:@"Authorization"];
 
 * DELETE 删除服务器上的文件
 
 request.HTTPMethod = @"DELETE";
 
 身份验证和PUT一致!
 */
// PUT上传
- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event
{
    [self putUpload];
}

/** 删除服务器上的文件 Web-DAV */
- (void)deleteFile
{
    // 1. URL 中需要指定PUT上传到服务器上的文件名
    NSURL *url = [NSURL URLWithString:@"http://192.168.26.201/uploads/123.mp4"];
    
    // 2. request
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"DELETE";
    
    // 2.1 设置用户名和密码
    NSString *authStr = @"admin:123456";
    // 结果: "BASIC YWRtaW46MTIzNDU2"
    NSString *authBase64 = [@"BASIC " stringByAppendingString:[self base64:authStr]];
    // 设置HTTP请求头属性
    // ****** 注意:不同的网络环境变化,通常都是设置HTTP的请求头字段,通过请求头字段可以告诉服务器很多内容!
    // 譬如:上传文件,授权信息等等,具体如何设置,公司的后端程序员会告诉前端
    [request setValue:authBase64 forHTTPHeaderField:@"Authorization"];
    
    // 3. sesssion 只能使用request进行DELETE删除
    [[[NSURLSession sharedSession] dataTaskWithRequest:request completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        
        NSLog(@"完成!");
        
    }] resume];
}

- (void)putUpload
{
    // 1. URL 中需要指定PUT上传到服务器上的文件名
    NSURL *url = [NSURL URLWithString:@"http://192.168.26.201/uploads/123.png"];
    
    // 2. request
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"PUT";
    
    // 2.1 设置用户名和密码
    NSString *authStr = @"admin:123456";
    // 结果: "BASIC YWRtaW46MTIzNDU2"
    NSString *authBase64 = [@"BASIC " stringByAppendingString:[self base64:authStr]];
    // 设置HTTP请求头属性
    // ****** 注意:不同的网络环境变化,通常都是设置HTTP的请求头字段,通过请求头字段可以告诉服务器很多内容!
    // 譬如:上传文件,授权信息等等,具体如何设置,公司的后端程序员会告诉前端
    [request setValue:authBase64 forHTTPHeaderField:@"Authorization"];
    
    // 3. sesssion 只能使用request进行PUT上传
    // 参数fromFile -> 本地要上传文件的完成URL
    NSURL *localURL = [[NSBundle mainBundle] URLForResource:@"02.网络应用层次结构.mp4" withExtension:nil];
    
    // 4. 跟踪上传进度
    NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
    
    NSURLSession *session = [NSURLSession sessionWithConfiguration:config delegate:self delegateQueue:nil];
    
    [[session uploadTaskWithRequest:request fromFile:localURL completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        
        NSString *result = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
        
        // 提示,跟踪的时候一定不要只输出result
        // 提示:如果是覆盖文件,NSData没有内容
        NSLog(@"====> %@", result);
    }] resume];
}

#pragma mark - 上传文件进度代理方法
/**
 bytesSent                  本次上传字节数
 totalBytesSent             总共上传字节数
 totalBytesExpectedToSend   要上传的字节数
 */
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didSendBodyData:(int64_t)bytesSent totalBytesSent:(int64_t)totalBytesSent totalBytesExpectedToSend:(int64_t)totalBytesExpectedToSend
{
    float progress = (float)totalBytesSent / totalBytesExpectedToSend;
    
    NSLog(@"%f %@", progress, [NSThread currentThread]);
}

/** 
 将指定的字符串,进行base64编码
 
 BASE64 - 是网络上最常用的编码方式之一(可以编码->看不懂,可以解码->又看的懂了)
 
 BASE64是很多加密算法中的基础算法!
 
 能够达到的效果,将二进制文件转换成ASCII码,从而可以将二进制文件的结果直接当成URL的参数传递
 */
- (NSString *)base64:(NSString *)string
{
    // 1. 将字符串转换成二进制数据
    NSData *data = [string dataUsingEncoding:NSUTF8StringEncoding];
    
    // 2. 进行base64编码
    return [data base64EncodedStringWithOptions:0];
}

- (void)viewDidLoad
{
    [super viewDidLoad];
    
    NSLog(@"%@", [self base64:@"admin:123456"]);
}
{% endhighlight %}

> * post上传
用到分类
链接: http://pan.baidu.com/s/1mgCKndm 密码: ghq4

{% highlight objc %}
// POST 上传
- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event
{
    // 1. URL,是真正负责上传的脚本(php)文件
    NSURL *url = [NSURL URLWithString:@"http://localhost/post/upload.php"];
    
    // 2. request
    /**
     1> 负责上传文件的URL
     2> 上传的文件名:保存在服务器上的文件名称
     3> 本地要上传文件的完整路径
     */
    NSString *localPath = [[NSBundle mainBundle] pathForResource:@"001.png" ofType:nil];
    NSURLRequest *request = [NSMutableURLRequest requestWithUploadURL:url uploadFileName:@"123.png" localFilePath:localPath];
    
    // 3. 上传
    [[[NSURLSession sharedSession] dataTaskWithRequest:request completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        
        NSString *result = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
        NSLog(@"%@", result);
    }] resume];
}
{% endhighlight %}





[多种方式实现文件的上传/下载][jekyll - DB]
 

[jekyll - DB]:     http://www.cocoachina.com/ios/20151012/13621.html

	


