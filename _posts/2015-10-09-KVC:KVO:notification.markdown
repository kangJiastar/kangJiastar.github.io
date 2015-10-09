---
layout: post
title:  "KVC、KVO、通知机制"
date:   2015-10-09 10:37:30
categories: jekyll update
---


##KVO(监听某个值的变化)

{% highlight objc %}
- (void)testKvo
{
    HMPerson *p = [[HMPerson alloc] init];
    
    p.age = 20;
    
    [p addObserver:self forKeyPath:@"age" options:NSKeyValueObservingOptionOld | NSKeyValueObservingOptionNew context:nil];
    
    p.age = 30;
    p.age = 40;
    
    self.p = p;
}


/**
 *  当监控的某个属性的值改变了就会调用
 *
 *  @param keyPath 属性名（哪个属性改了？）
 *  @param object  哪个对象的属性被改了？
 *  @param change  属性的修改情况（属性原来的值、属性最新的值）
 *  @param context void * == id
 */
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context
{
    NSLog(@"%@对象的%@属性改变了：%@", object, keyPath, change);
}

//释放
- (void)dealloc
{
    [self.p removeObserver:self forKeyPath:@"age"];
}
{% endhighlight %}

##KVC（间接通过字符串类型的key取出对应的属性值）

# KVC的价值
>1.可以访问私有成员变量的值
 2.可以间接修改私有成员变量的值（替换系统自带的导航栏、tabbar）

通过下面两个方法
`- (id)valueForKey:(NSString *)key;`
`- (id)valueForKeyPath:(NSString *)keyPath;`

其中使用`keyPath`是一个路径，对象之间关系的路径，假如一个`Person`对象拥有一条`Dog`,想要访问`Dog`的`name`这个时候可以这样访问
`NSString *name =  [p valueForKeyPath:@"dog.name"];`
所以`keyPath`更加常用一些

#一些并不常用但比较有意思的方法
{% highlight objc %}
 HMBook *b1 = [[HMBook alloc] init];
    b1.name = @"kuihua";
    b1.price = 100.6;
    
    HMBook *b2 = [[HMBook alloc] init];
    b2.name = @"pixie";
    b2.price = 5.6;
    
    HMBook *b3 = [[HMBook alloc] init];
    b3.name = @"jiuyin";
    b3.price = 50.6;
    
    p.books = @[b1, b2, b3];
    
    
    // 计算数组的长度
    NSNumber *num = [p valueForKeyPath:@"books.@count"];
 
    //取出所有的输名字
    NSArray *names = [p valueForKeyPath:@"books.name"];
    NSArray *names = [p.books valueForKeyPath:@"name"];
    NSLog(@"%@", names);
    
    //计算所有书的价值总和
   double sumPrice = [[p valueForKeyPath:@"books.@sum.price"] doubleValue];
    NSLog(@"%f", sumPrice);

{% endhighlight %}

##通知

> * 注意：通知处理事件在哪个线程，取决于发送通知在哪个线程，在主线程发送的通知在主线程执行，在异步线程发送的通知在异步线程执行

一个完整的通知一般包含3个属性：
`name;` // 通知的名称
`object`; // 通知发布者(是谁要发布通知)
`userInfo`; // 一些额外的信息(通知发布者传递给通知接收者的信息内容)


初始化一个通知（NSNotification）对象
{% highlight objc %}
+ (instancetype)notificationWithName:(NSString *)aName object:(id)anObject;
+ (instancetype)notificationWithName:(NSString *)aName object:(id)anObject userInfo:(NSDictionary *)aUserInfo;
- (instancetype)initWithName:(NSString *)name object:(id)object userInfo:(NSDictionary *)userInfo;
{% endhighlight %}

#发布通知

通知中心`NSNotificationCenter`提供了相应的方法来帮助发布通知

`- (void)postNotification:(NSNotification *)notification;`

发布一个`notification`通知，可在`notification`对象中设置通知的名称、通知发布者、额外信息等

`- (void)postNotificationName:(NSString *)aName object:(id)anObject;`

发布一个名称为`aName`的通知，`anObject`为这个通知的发布者

`- (void)postNotificationName:(NSString *)aName object:(id)anObject userInfo:(NSDictionary *)aUserInfo;`


发布一个名称为`aName`的通知，`anObject`为这个通知的发布者，`aUserInfo`为额外信息



#监听通知

通知中心`NSNotificationCenter`提供了方法来注册一个监听通知的监听器`Observer`

`- (void)addObserver:(id)observer selector:(SEL)aSelector name:(NSString *)aName object:(id)anObject;`

observer：监听器，即谁要接收这个通知

aSelector：收到通知后，回调监听器的这个方法，并且把通知对象当做参数传入

aName：通知的名称。如果为nil，那么无论通知的名称是什么，监听器都能收到这个通知

anObject：通知发布者。如果为anObject和aName都为nil，监听器都收到所

`- (id)addObserverForName:(NSString *)name object:(id)obj queue:(NSOperationQueue *)queue usingBlock:(void (^)(NSNotification *note))block;`

name：通知的名称
obj：通知发布者
block：收到对应的通知时，会回调这个block
queue：决定了block在哪个操作队列中执行，如果传nil，默认在当前操作队列中同步执行


#释放通知

通知中心不会保留`retain`监听器对象，在通知中心注册过的对象，必须在该对象释放前取消注册。否则，当相应的通知再次出现时，通知中心仍然会向该监听器发送消息。因为相应的监听器对象已经被释放了，所以可能会导致应用崩溃

通知中心提供了相应的方法来取消注册监听器

`- (void)removeObserver:(id)observer;`

`- (void)removeObserver:(id)observer name:(NSString *)aName object:(id)anObject;`

一般在监听器销毁之前取消注册（如在监听器中加入下列代码）：
{% highlight objc %}
- (void)dealloc {
	//[super dealloc];  非ARC中需要调用此句
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}
{% endhighlight %}

UIDevice通知

UIDevice类提供了一个单粒对象，它代表着设备，通过它可以获得一些设备相关的信息，比如电池电量值(batteryLevel)、电池状态(batteryState)、设备的类型(model，比如iPod、iPhone等)、设备的系统(systemVersion)

通过[UIDevice currentDevice]可以获取这个单粒对象

UIDevice对象会不间断地发布一些通知，下列是UIDevice对象所发布通知的名称常量：
`UIDeviceOrientationDidChangeNotification // 设备旋转`
`UIDeviceBatteryStateDidChangeNotification // 电池状态改变`
`UIDeviceBatteryLevelDidChangeNotification // 电池电量改变`
`UIDeviceProximityStateDidChangeNotification // 近距离传感器(比如设备贴近了使用者的脸部)`


##通知小案例
在`main.m`中
{% highlight objc %}
int main(int argc, const char * argv[])
{

    @autoreleasepool {

        // 1. 初始化新闻机构
        NewsCompany *tx = [[NewsCompany alloc] init];
        tx.name = @"腾讯新闻";
        
        NewsCompany *sina = [[NewsCompany alloc] init];
        sina.name = @"新浪新闻";
        
        // 2. 初始化人对象
        Person *zhangsan = [[Person alloc] init];
        zhangsan.name = @"张三";
        
        Person *lisi = [[Person alloc] init];
        lisi.name = @"李四";
        
        Person *wangwu = [[Person alloc] init];
        wangwu.name = @"王五";
        
        // 拿到通知中心对象
        NSNotificationCenter *center = [NSNotificationCenter defaultCenter];
        
        // 3. 收听新闻， 添加监听器
        //接收所有通知
        [center addObserver:zhangsan selector:@selector(newsCome:) name:nil object:nil];
        //只接受yule_news的通知
        [center addObserver:lisi selector:@selector(newsCome:) name:@"yule_news" object:tx];
        
        
        // 4. 发布通知
        // tx发布一条新闻
        [center postNotificationName:@"junshi_news" object:tx userInfo:@{@"title" : @"巴以冲突", @"text" : @"巴以冲突升级。。。。。"}];
        
        [center postNotificationName:@"yule_news" object:sina userInfo:@{@"title" : @"xx星吸毒被抓", @"text" : @"xx星吸毒被抓...."}];
        
    }
    return 0;
}
{% endhighlight %}
在`Person.m`中
{% highlight objc %}
@implementation Person
//实现相应的方法
- (void)newsCome:(NSNotification *)note
{
    NSLog(@"%@收到新闻, 新闻的内容%@", self.name, note);
}

- (void)dealloc
{
//    [super dealloc];
    // 把人监听的所有通知都移除掉
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

@end

{% endhighlight %}

###通知和代理的联系和区别
> * 共同点
利用通知和代理都能完成对象之间的通信
(比如A对象告诉D对象发生了什么事情, A对象传递数据给D对象)

> * 不同点
代理 : 一对一关系(1个对象只能告诉另1个对象发生了什么事情)
通知 : 多对多关系(1个对象能告诉N个对象发生了什么事情, 1个对象能得知N个对象发生了什么事情)




















