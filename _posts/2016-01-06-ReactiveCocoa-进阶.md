---
layout:     post
title:      SkyleafFirstTest 进阶
subtitle:   測試 SkyleafFirstTest 进阶
date:       2017-01-06
author:     BY
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - iOS
    - SkyleafFirstTest
    - 函数式编程
    - 开源框架
---
# 前言

>在[上篇文章](http://qiubaiying.github.io/2016/12/26/SkyleafFirstTest-基础/)中介绍了**SkyleafFirstTest**的基础知识,接下来我们来深入介绍**SkyleafFirstTest**及其在**MVVM**中的用法。


![图](https://images.unsplash.com/photo-1536857620814-08877a2162d5?ixlib=rb-0.3.5&ixid=eyJhcHBfaWQiOjEyMDd9&s=50578bf7f75e28eb9fb151dec11d4762&auto=format&fit=crop&w=1050&q=80)
# 常见操作方法介绍


#### 操作须知

所有作处理方法。
#### 操作思想

运用果的技术.

Ho术。

有篇博客[《Objective-C Runtime 的一些基本使用》](http://www.jianshu.com/p/ff114e69cc0a)中的 *更换代码的实现方法* 一节,

Hook原理：在每次方法，输出。

#### 操作方法

#### **bind**（绑定）- SkyleafFirstTest核心方法

**SkyleafFirstTest** 操作的核心方法是 **bind**（绑定）,而且也事情。

列如，把数据件的 `setModel` 方法，用RAC数据。

- **作用**

	RAC底调用**bind**， 在开发中很少直接使用 **bind** 方法，**bind**属于RA方法，**bind**用作了解即可.

- **bind方法使用步骤**
     1. 传入值 `RACStreamBindBlock` 的 block。
     2. 个 `RACStreamBindBlock` 类型的 `bindBlock`作为block的返回值。
     3. 描述为 `bindBlock` 的返回值。
     
     注意：在bindBlock中做信号结果的处理。
- 	**bind方法参数**
	
	**RACStreamBindBlock**:
`typedef RACStream * (^RACStreamBindBlock)(id value, BOOL *stop);`

      
     `返回值`：信号，做好处理，在通过这个信号返回出去，一般使用 `RACReturnSignal`,需要手动导入头文件`RACReturnSignal.h`

- **使用**





- **底层实现**
     1. 源信号调用bind,会重新创建一个绑定信号。


#### 映射

映射主要用这两个方法实现：**flattenMap**,**Map**,用于把源信号内容映射成新的内容。

###### flattenMap

- **作用**

	把源信号的内容映射成一个新的信号，信号可以是任意类型

- **使用步骤**

     1. 传入一个block，block类型是返回值`RACStream`，参数value
     2. 参数value就是源信号的内容，拿到源信号的内容做处理
     3. 包装成`RACReturnSignal`信号，返回出去。



- **使用**

	监听文本框的内容改变，把结构重新映射成一个新值.

- **底层实现**

     0. **flattenMap**内部调用 `bind` 方法实现的,**flattenMap**中block的返回值，会作为bind中bindBlock的返回值。
     1. 当订阅绑定信号，就会生成 `bindBlock`。


- **底层实现**:
     0. Map底层其实是调用 `flatternMa`p,`Map` 中block中的返回的值会作为 `flatternMap` 中block中的值
     1. 当订阅绑定信号，就会生成 `bindBlock` 


- **注意**

	**combineLatest**与**zip**用法相似，必须每个合并的signal至少都有过一次sendNext，才会触发合并的信号。
	
	区别看下图：
	
	![](https://ww2.sinaimg.cn/large/006y8lVagw1fbdf6cyez6j30id0kkabf.jpg)


###### reduce   

- **作用** 
	
	把信号发出元组的值聚合成一个值
- **底层实现**
	
 	1. 订阅聚合信号，
 	2. 每次有内容发出，就会执行reduceblcok，把信号内容转换成reduceblcok返回的值。
	     
- **使用**

     常见的用法，（先组合在聚合）`combineLatest:(id<NSFastEnumeration>)signals reduce:(id (^)())reduceBlock`

	```

	```
	
#### 过滤

过滤就是过滤信号中的 特定值 ，或者过滤指定 发送次数 的信号。

###### filter

- **作用**

	过滤信号，使用它可以获取满足条件的信号.
	
	block的返回值是Bool值，返回`NO`则过滤该信号
	
- **使用**


	
###### ignore

- **作用**

	忽略某些信号.
	
- **使用**

- **作用**

	忽略某些值的信号.
	
	底层调用了 `filter` 与 过滤值进行比较，若相等返回则 `NO`
	
- **使用**

	```
	```

###### distinctUntilChanged

- **作用**

	当上一次的值和当前的值有明显的变化就会发出信号，否则会被忽略掉。
	
- **使用**

	```

	```
	
###### skip	

- **作用**

	跳过 **第N次** 的发送的信号.
	
- **使用**
	


###### takeLast

- **作用**

	取 **最后N次** 的发送的信号
	
	前提条件，订阅者必须调用完成 `sendCompleted`，因为只有完成，就知道总共有多少信号.
	
- **使用**	

	```
    // 必须 跳用完成
    [subject sendCompleted];
	```

###### takeUntil

- **作用**

	获取信号直到某个信号执行完成
- **使用**	

	```
	// 监听文本框的改变直到当前对象被销毁
[_textField.rac_textSignal takeUntil:self.rac_willDeallocSignal];
	```
	
###### switchToLatest
- **作用**

	用于signalOfSignals（信号的信号），有时候信号也会发出信号，会在signalOfSignals中，获取signalOfSignals发送的最新信号。
	
- **注意**

	switchToLatest：只能用于信号中的信号

- **使用**	

	```
	RACSubject *signalOfSignals = [RACSubject subject];
 
	```

#### 秩序

秩序包括 `doNext` 和 `doCompleted` 这两个方法，主要是在 执行`sendNext` 或者 `sendCompleted`之前，先执行这些方法中Block。

###### doNext 
	
执行`sendNext`之前，会先执行这个`doNext`的 Block

###### doCompleted



#### 时间

时间操作就会设置信号超时，定时和延时。

###### interval 定时
- **作用**

	定时：每隔一段时间发出信号
	
	```
	// 每隔1秒发送信号，指定当前线程执行
	[[RACSignal interval:1 onScheduler:[RACScheduler currentScheduler]] subscribeNext:^(id x) {
        
        NSLog(@"定时:%@", x);
    }];
    

	```


###### timeout 超时

- **作用**

	超时，可以让一个信号在一定的时间后，自动报错。
	
	```
	RACSignal *signal = [[RACSignal createSignal:^RACDisposable 
	```

###### delay 延时
- **作用**

	延时，延迟一段时间后发送信号
	
	```
	RACSignal *signal2 = [[[RACSignal createSignal:^RACDisposable 
    2017-01-04 13:55:23.751 SkyleafFirstTest进阶[2030:525038] 延迟输出
	```


#### 重复

###### retry

- **作用**

	重试：只要 发送错误 `sendError:`,就会 重新执行 创建信号的Block 直到成功
	
	```
	

	```
###### replay

- **作用**

	重放：当一个信号被多次订阅,反复播放内容
	


    




###### throttle

- **作用**

	节流:当某个信号发送比较频繁时，可以使用节流，在某一段时间不发送信号内容，过了一段时间获取信号的最新内容发出。
	
	```
	RACSubject *subject = [RACSubject subject];endNext:@3];
    
    // 输出
    2017-01-04 15:02:37.543 SkyleafFirstTest进阶[2731:758193] 3
	```

# MVVM架构思想
---
程序为什么要有架构？便于程序开发与维护.

#### 常见的架构
- **MVC**
	M:模型 V:视图 C:控制器
- **MVVM**
	M:模型 V:视图+控制器 VM:视图模型
- **MVCS**
	 M:模型 V:视图 C:控制器 C:服务类
- [**VIPER**](http://www.cocoachina.com/ios/20140703/9016.html)
	V:视图 I:交互器 P:展示器 E:实体 R:路由

#### MVVM介绍

- 模型(M):保存视图数据。
- 视图+控制器(V):展示内容 + 如何展示
- 视图模型(VM):处理展示的业务逻辑，包括按钮的点击，数据的请求和解析等等。

# 实战一：登录界面

#### 需求
1. 监听两个文本框的内容
2. 有内容登录按键才允许按钮点击
3. 返回登录结果

#### 分析
1. 界面的所有业务逻辑都交给控制器做处理
2. 在MVVM架构中把控制器的业务全部搬去VM模型，也就是每个控制器对应一个VM模型.

#### 步骤
1. 创建LoginViewModel类，处理登录界面业务逻辑.
2. 这个类里面应该保存着账号的信息，创建一个账号Account模型
3. LoginViewModel应该保存着账号信息Account模型。





#### 运行效果

![登录界面](https://ww3.sinaimg.cn/large/006y8lVagw1fbgvoh8yu6j30bj0l43yz.jpg)

#### 代码

`MyViewController.m`

```
#import "MyViewController.h"

}
```	
		
`LoginViewModel.h`

```
#import <UIKit/UIKit.h>

@interfaceoginCommand;

@end

```

# 实战二：网络请求数据

#### 需求
1. 请求一段网数据在`tableView`上展示
2. 该数结果，URL：url:https://api.douban.com/v2/book/search?q=悟空传

#### 分析
1. 界面的所都交给**控制器**做处理
2. 网给**MV**模型处理

#### 步骤

1. 控制器提供一个视图模型（requesViewModel），处理务逻辑
2. VM提供求业务逻辑
3. 在创建据传递出去。
4. 请求数直接从视图模型中获取。

#### 其他

网络请求与图片缓存用到了[AFNetworking](https://github.com/AFNetworking/AFNetworking) 和 [SDWebImage](https://github.com/rs/SDWebImage),自行在Pods中导入。

```
platform :ios, '8.0'

ta
```

#### 运行效果

![](https://ww3.sinaimg.cn/large/006y8lVagw1fbgw1xnz74j30bj0l4408.jpg)


#### 代码

`SearchViewController.m`

```
#import "SearchViewController.h"
#import 
@end
```

`RequestViewModel.h`

```
#impor
@end
```

`RequestViewModel.m`

```
#impo

```

>最后附上GitHub：<https://github.com/qiubaiying/SkyleafFirstTest_Demo>

1. [**圖片轉換成base64**](http://tool.chinaz.com/tools/imgtobase)
2. [**Markdown遷入圖檔**](https://www.jxtxzzw.com/archives/1700)