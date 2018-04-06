# PushDemo
iOS8对推送消息的快速回复demo


全文导图

对Push的理解

    苹果对Push的所有优化最终目的都是为了提升用户体验。这一点上不得不佩服Apple！

在讲Push之前先谈谈为什么需要推送。对用户而言大部分就是为了获得最新的信息，感兴趣的资讯；对app开发者而言大部分是为了通过推送让用户打开app增加日活（别喷我），其次是为了向用户提供更好的资讯；对苹果而言因为iOS系统给app在后台最多存活的时间最多三分钟（后来新增后台模式后增加到十分钟），为了保证这个iOS平台能够给用户良好的体验（对推送可控），app可以主动和用户沟通。

对APNS（Remote Push）的理解

可以通俗的把APNS理解为iOS系统为每个app提供的长连接通道。只是这个通道需要通过苹果中转，为什么苹果要设计这么一套服务呢。上面也提到了过，下面在针对用户体验详细一点介绍。

    提升用户体验：苹果限制了每个app在后台存活的时间，最重要的目的是为了省电，其次优化内存这些。如果彻彻底底的将app杀死了，服务端永远不能主动和客户端建立联系。所以需要一种机制来保证在必要的时候让用户知道服务端所做的改变。技术上只要只有长连接可以做到。

    便于苹果、用户控制：如果直接让app和服务端建立长连接（比如iOS8之前的voip，就是app在后台保持长连接），苹果是不能控制的。所以通过在app和服务端中间加一个APNS可以有效的进行拦截处理。比如可以由用户开启是否接受远程推送。退一万步讲，如果哪天苹果对你上架的app进行了下架处理。即使有用户安装了你的app，切掉你的APNS，用户也无法收到推送，除非用户自己点开app否则你的app永远不会存活。

APNS缺点也很明显

    可靠性、稳定性。一般情况下，Apple会保证这个通道的Qaulity of Service，也就是推送的消息能及时稳定到达设备。不过一旦用户的设备处于offline状态，Apple只会存储发送给用户的最新一条push，之前发送的push会被直接丢掉。而且这最后一条离线push也是有过期时间的。一些用户应该有过这种经历，在使用某些的时候，明明对方发送了多条消息，却只收到了一条push。

    消息大小限制：由于苹果APNS服务于以万计的app，所以对消息内容大小有严格限制。只能传递一些文本信息。Apple在文档里清楚的说明，push只应该用来通知用户有新的内容，而不应该用来承载内容本身。理论上payload size越小，push到达设备的概率就越高。苹果一直在改善，在iOS8之前max payload size是256字节，到iOS8发布这个最大值被调整到了2048字节，再到的iOS9发布，引入了HTTP2.0，payload size又被设为4KB（4 * 1024字节）了。

对Local Push的理解

有了APNS（Remote Push）为什么苹果还搞了一套Local Push呢。从笔者的角度能做出如下猜测:

    苹果开发者中，有很大一部分是个人开发者。对于个人开发者而言，做一款小型的app，很多时候用APNS需要搭建服务，成本太高。

    APNS前提条件是必须在联网的状态下，这样一些不需要联网的app，比如日程提醒类，就和Push彻底告别了。但是对于这种小型app，Local Push非常适合。

    APNS稳定性及成功率并不是那么高。大型app可以采用两种渠道提高通知用户的成功率。比如后面会讲到的voip Push的使用一般就是结合local Push使用。

    Local Push比APNS更加灵活。参数更加多样。

    ......

如何选择

因为app不总是会运行的，Local Push提供了另一种提醒用户信息的方式。比如应用在后台没有被杀死的时候，从服务端拉取数据更新，本地就可以根据服务端返回的信息，删选出是否有用户感兴趣的部分，然后组织好消息，通过本地推送告诉用户。当然也可以用远程push。但是这个时候远程push并没有本地push可靠。

Remote Push一般在应用杀死的情况下才使用。用户主动去打开app，之后请求数据。

    简单总结一下，如果能够直接拿到数据最好用本地push，如果拿不到，比如说应用被杀死采用远程push。

从用户角度，这两种Push没有任何区别，具体来讲我们可以控制Push的如下形式：

    显示提示框还是横幅

    app icon上显示的数字

    显示横幅、数字、提示框所用的提示音

    特别需要注意的几点：

    App在前台运行的时候，通知不会展示出来（在iOS10之后可以在userNotificationCenter:willPresentNotification:

    withCompletionHandler:进行处理，满足可以在前台显示通知）

    点击通知，默认会自动打开推送通知的App

    不管App是否打开，通知都可以发出

    可以取消本地push

    可以设定本地push的自定义处理action（控制点击Push是否打开App）

上面这几点注意事项可以在在UILocalNotification Class Reference找到（最好的资料还是官方文档）

本地推送（Local Push）

Local Push是不需要走APNS，让客户端更加灵活的控制。常见的比如一款闹钟App。app本地处理好推送逻辑，然后把推送逻辑交给系统。即使当app不在前台的时候也可以由系统发出通知告知用户。

App在启动的时候就需要对本地和远程推送进行设置，也就是不能晚于application:didFinishLaunchingWithOptions:调用。官方文档上说其实也可以，只要在处理只通知之前配置过就行。

配置推送

使用Push第一步需要向用户申请权限

iOS10之后直接用下面这代码
1
2
3
4
5
	
UNUserNotificationCenter* center = [UNUserNotificationCenter currentNotificationCenter];
[center requestAuthorizationWithOptions:(UNAuthorizationOptionAlert + UNAuthorizationOptionSound)
   completionHandler:^(BOOL granted, NSError * _Nullable error) {
      // Enable or disable features based on authorization.
}];

    注意：因为系统会保存用户的授权状态，所以下次启动的时候虽然调用了这代码，依然不会再次提示用户。

通知类别和通知Action

通知类别：类别定义了app是如何展示通知的。将类别和自定义的action关联起来，并且设置具体的选项（option）。让推送更加灵活

Action（UNNotificationAction）具体来讲长什么样：一般推送就一个横幅，下面没有按钮。Action其实就是横幅下面的按钮。截图如下

Actionable notifications的界面提供了用户可以点击的按钮，Actionable notifications让用户可以快速的执行相关的操作，不用强迫用户打开你的app(通过option设置是否打开app)。当用户点击的时候，会把相关的事件立即转发到app中处理。

在启动的时候。app可以注册一个或多个的通知类别，这些类别决定了app会发出哪中类型的通知。 通过设置UNNotificationCategoryOptions决定。

总的来讲这里有如下几个重要的类：

    UNUserNotificationCenter ：iOS10之后新增加的通知框架中。可以简单理解为这就是一个单例的Service。基本所有的通知设置都是通过去设置的。设置通知的入口 Manages the notification-related activities for your app or app extension.大致来讲有如下几个作用：

    请求通知权限

    声明app支持的通知类型及自定义action。

    安排发出通知

    管理在通知中心显示的具体通知。

    获取通知相关的设置

UNNotificationCategory :设置类别的名称和各个可选项。类别的名称就是这个通知的唯一标识，系统会用来查找和显示相关通知。Defines the types of notifications your app supports and the custom actions displayed for each type.

UNNotificationAction：定义响应通知的具体任务。比如设置当点击action的时候是否打开app，处理通知是否在非锁屏状态。Defines a task to perform in response to a delivered notification.

创建及注册类别

前面提到过每个通知类别可以包含最多四个自定义action。如果类别包含了自定义的action，系统会在通知界面上添加按钮，每个都会以设置的action的title为按钮的title。如果用户点击了其中任何一个action。系统将会发送相关的action标识给app，app可以使用这个标识来标识后面执行的任务。
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
	
// Create the custom actions for expired timer notifications.
UNNotificationAction* snoozeAction = [UNNotificationAction
      actionWithIdentifier:@"SNOOZE_ACTION"
      title:@"Snooze"
      options:UNNotificationActionOptionNone];
  
UNNotificationAction* stopAction = [UNNotificationAction
      actionWithIdentifier:@"STOP_ACTION"
      title:@"Stop"
      options:UNNotificationActionOptionForeground];
  
// Create the category with the custom actions.
UNNotificationCategory* expiredCategory = [UNNotificationCategory
      categoryWithIdentifier:@"TIMER_EXPIRED"
      actions:@[snoozeAction, stopAction]
      intentIdentifiers:@[]
      options:UNNotificationCategoryOptionNone];
  
// Register the notification categories.
UNUserNotificationCenter* center = [UNUserNotificationCenter currentNotificationCenter];
[center setNotificationCategories:[NSSet setWithObjects:generalCategory, expiredCategory,
      nil]];

    有时候虽然为类别设置了四个action，但是系统在某些环境下只有显示前两个。比如系统会在以横幅显示通知的时候只会显示前两个action。所以初始化action的时候，最好把优先级高的设置在前面。如果使用UNTextInputNotificationAction , 系统会提供用户一个输入框最为这个通知的响应。文本输入框这种通知响应方式对IM相关的app非常有用。截图如下：

配置推送的声音

本地推送和远程推送可以自定义提示音。因为是通过系统声音渠道播放的，所以自定义声音必须遵循如下几种数据格式：

    Linear PCM

    MA4 (IMA/ADPCM)

    μLaw

    aLaw

具体来讲：声音文件可以放到app bundle里面，或者如果是从网上下载的话就放在你pp沙盒目录下的Library/Sounds。自定义声音必须少于30秒。如果大于30秒将会用默认的系统的提示音取代。

需要注意的几点

    通知类别UNNotificationCategoryOptions参数含义。

1
2
3
4
5
6
7
8
9
10
11
12
	
typedef NS_OPTIONS(NSUInteger, UNNotificationCategoryOptions) {
     
    // 用户点击系统取消Action是否响应到通知代理
    UNNotificationCategoryOptionCustomDismissAction = (1 << 0),
     
    // 通知在CarPlay(CarPlay 是美国苹果公司发布的车载系统)
    UNNotificationCategoryOptionAllowInCarPlay = (1 << 1),
     
    // 通知预览关闭情况下是否显示通知的title
    UNNotificationCategoryOptionHiddenPreviewsShowTitle __IOS_AVAILABLE(11.0) __WATCHOS_PROHIBITED = (1 << 2),
    // 通知预览关闭情况下是否显示通知的子title    UNNotificationCategoryOptionHiddenPreviewsShowSubtitle __IOS_AVAILABLE(11.0) __WATCHOS_PROHIBITED  = (1 << 3),
}

这里简单说一下什么是通知预览。在设置里面

关闭之后设置UNNotificationCategoryOptions为UNNotificationCategoryOptionHiddenPreviewsShowTitle | UNNotificationCategoryOptionHiddenPreviewsShowSubtitle。效果如下：

显示的内容就用数字代替了。

    通知Action参数UNNotificationActionOptions含义

1
2
3
4
5
6
7
8
9
	
typedef NS_OPTIONS(NSUInteger, UNNotificationActionOptions) {
    //执行代理是否需要在非锁屏状态下
    UNNotificationActionOptionAuthenticationRequired = (1 << 0),
     
    //决定按钮显示是否为红色。（红色代表消极？）
    UNNotificationActionOptionDestructive = (1 << 1),
    //决定点击action是否打开app
    UNNotificationActionOptionForeground = (1 << 2),
}

特别注意当设置为UNNotificationActionOptionAuthenticationRequired的时候，即使点击了action代理方法userNotificationCenter:didReceiveNotificationResponse:withCompletionHandler也不会走。如果设置为UNNotificationActionOptionNone。则即使在锁屏情况下也会走代理方法

创建本地推送

在app运行的时候（无论是前台还是后台），设置好本地推送，然后系统会在合适的时间发出推送。

    如果app没有在运行或者在后台，系统将直接向用户展示通知，如果app提供了app通知扩展，系统使用用户自定义的界面提示用户。

    如果app在前台，系统会给app在内部处理通知的机会。

前面讲过配置本地推送，总结一下有如下几个步骤：

    设置内容：创建并设置好UNMutableNotificationContent。

    设置触发器：创建通知触发器UNCalendarNotificationTrigger, UNTimeIntervalNotificationTrigger, UNLocationNotificationTrigger其中一种。设置好触发通知的条件。

    连接内容和触发器：创建UNNotificationRequest ，设置content和trigger。

    添加通知：调用`addNotificationRequest:withCompletionHandler:计划通知。

1
2
3
4
5
6
7
8
9
10
11
12
	
 UNMutableNotificationContent* content = [[UNMutableNotificationContent alloc] init];
    content.title = [NSString localizedUserNotificationStringForKey:@"Wake up!" arguments:nil];
    content.body = [NSString localizedUserNotificationStringForKey:@"Rise and shine! It's morning time!"
                                                         arguments:nil];
NSDateComponents* date = [[NSDateComponents alloc] init];
    date.second = 10;
    UNCalendarNotificationTrigger* trigger = [UNCalendarNotificationTrigger
                                              triggerWithDateMatchingComponents:date repeats:NO];
     
    // Create the request object.
    UNNotificationRequest* request = [UNNotificationRequest
                                      requestWithIdentifier:@"MorningAlarm" content:content trigger:trigger];

这里提供的唯一标识是为了后面查找或者取消通知。

前面提到的通知类别，这个时候就可以排上用场了。在UNMutableNotificationContent设置好之前已经注册了的通知类别categoryIdentifier。注意必须在安排全这个通知请求之前就设置好这个值。
1
2
3
4
5
	
UNNotificationContent *content = [[UNNotificationContent alloc] init];
// Configure the content. . .
  
// Assign the category (and the associated actions).
content.categoryIdentifier = @"TIMER_EXPIRED";

前面提到过可以自定义通知的声音。这里同样是在UNMutableNotificationContent对象上设置。使用UNNotificationSound对象，这个对象决定是使用自定义的声音还是系统默认声音。注意自定义的音频文件必须在设备上存在。存储在app的main bundle里面，或者下载并存储到app沙盒路径下得Library/Sounds子目录。
1
	
content.sound = [UNNotificationSound soundNamed:@"MySound.aiff"];

发出或者安排通知：系统是异步的安排本地通知。当安排完成或者出错之后通过回调block告知。
1
2
3
4
5
6
7
8
9
10
	
// Create the request object.
UNNotificationRequest* request = [UNNotificationRequest
       requestWithIdentifier:@"MorningAlarm" content:content trigger:trigger];
  
UNUserNotificationCenter* center = [UNUserNotificationCenter currentNotificationCenter];
[center addNotificationRequest:request withCompletionHandler:^(NSError * _Nullable error) {
   if (error != nil) {
       NSLog(@"%@", error.localizedDescription);
   }
}];

安排过的通知会一直存活到系统取消或者app处理过。a当通知发出过了系统会自己取消掉通知，除非这个通知设定为重复通知。取消通知可以通过removePendingNotificationRequestsWithIdentifiers:实现。上面提到过，取消是通过之前设置的标识取消的。

为了响应用户对通知的处理。需要实现那UNUserNotificationCenter的代理UNUserNotificationCenterDelegate。当自定义了action的时候代理必须实现。
1
2
3
4
	
// The method will be called on the delegate only if the application is in the foreground. If the method is not implemented or the handler is not called in a timely manner then the notification will not be presented. The application can choose to have the notification presented as a sound, badge, alert and/or in the notification list. This decision should be based on whether the information in the notification is otherwise visible to the user.
- (void)userNotificationCenter:(UNUserNotificationCenter *)center willPresentNotification:(UNNotification *)notification withCompletionHandler:(void (^)(UNNotificationPresentationOptions options))completionHandler __IOS_AVAILABLE(10.0) __TVOS_AVAILABLE(10.0) __WATCHOS_AVAILABLE(3.0);
// The method will be called on the delegate when the user responded to the notification by opening the application, dismissing the notification or choosing a UNNotificationAction. The delegate must be set before the application returns from application:didFinishLaunchingWithOptions:.
- (void)userNotificationCenter:(UNUserNotificationCenter *)center didReceiveNotificationResponse:(UNNotificationResponse *)response withCompletionHandler:(void(^)(void))completionHandler __IOS_AVAILABLE(10.0) __WATCHOS_AVAILABLE(3.0) __TVOS_PROHIBITED;

以下几点值得注意下：

    当app在前台的时候，app可以静音掉通知（也就是不处理）或者告诉系统自己展示其他页面。再次提醒，系统会默认静音掉所有通知当app在前台的时候。系统或直接把通知给app，让app自己使用通知传递的数据去做app自定义的任务。如果该需要系统继续显示通知界面，需要为UNUserNotificationCenter提供一个代理对象，实现userNotificationCenter:willPresentNotification:withCompletionHandler: 方法，在这个方法里面应该处理通知数据。完成之后，执行app想让系统去使用的通知发出选项。如果不做任何配置，系统将会静音掉这个通知。

    当app在后台或者已经停止运行。系统不会调用userNotificationCenter:willPresentNotification:withCompletionHandler:，在这种情况下，系统会自己提示用户根据通知的设置。app可以使用getDeliveredNotificationsWithCompletionHandler: 来查看已经发送过得通知。

    当用户通过界面选择了自定义的action，系统将会通知app，用户当前所做的选择。同样是通过代理告诉app。在代理userNotificationCenter:didReceiveNotificationResponse:withCompletionHandler: 必须实现那能够处理所有自定义的action。app如果没有运行，而用户点击了action。系统将会以后台模式打开app用于处理用户的选择。千万不要在这段时间处理一些无关的任务。

除了自定义的，还有系统的action。用户可能没有选择自定义的action而是取消掉通知页面或者打开app。和自定义action一样，系统的action也有对应的唯一标识：

    UNNotificationDismissActionIdentifier ：用户没有选择action，直接取消了通知界面

    UNNotificationDefaultActionIdentifier ：用户没有选择action，直接打开了app

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
	
// The user dismissed the notification without taking action.
    }
    else if ([response.actionIdentifier isEqualToString:UNNotificationDefaultActionIdentifier]) {
        // The user launched the app.
    }
     
    if ([response.notification.request.content.categoryIdentifier isEqualToString:@"TIMER_EXPIRED"]) {
        // Handle the actions for the expired timer.
        if ([response.actionIdentifier isEqualToString:@"SNOOZE_ACTION"])
        {
            // Invalidate the old timer and create a new one. . .
        }
        else if ([response.actionIdentifier isEqualToString:@"STOP_ACTION"])
        {
            // Invalidate the timer. . .
        }
         
    }

注意事项

    （针对本地推送）应用在前台并不是不能弹出系统通知框。上面提到如果该需要系统继续显示通知界面，需要为UNUserNotificationCenter提供一个代理对象，实现userNotificationCenter:willPresentNotification:withCompletionHandler: 方法，在这个方法里面应该处理通知数据。完成之后，执行app想让系统去使用的通知发出选项。如果不做任何配置，系统将会静音掉这个通知。

1
2
3
4
5
6
7
8
	
- (void)userNotificationCenter:(UNUserNotificationCenter *)center
     willPresentNotification:(UNNotification *)notification
       withCompletionHandler:(void (^)(UNNotificationPresentationOptions))completionHandler{
   //1. 处理通知
    
   //2. 处理完成后条用 completionHandler ，用于指示在前台显示通知的形式
   completionHandler(UNNotificationPresentationOptionAlert);
}

    UNNotificationTrigger：系统提供了基于时间和基于地理位置的Trigger。如果想设置为时间Trigger为Repeat，至少需要60s，如果用UNCalendarNotificationTrigger，虽然可以设置小于60s的时间间隔但是并不是按照设置的时间执行

    重复的通知只会在通知栏显示一个

远程推送（Remote Push）

远程推送稍微大点的app都有有这功能。为了接受远程推送需要有如下几步：

    开启远程推送

    注册APNS服务，接收苹果返回的device token

    把收到的device token发送给服务端

    处理接收到的远程推送

配置推送

开启远程推送直接可以在xcode中操作。之前需要申请推送证书。如果没有必须得entitlements，那么app在被app store审核的时候会被拒。

每一次app启动，都会在APNS注册。大致流程如下：

    app请求APNS注册

    注册成功之后，APNS会把对应的token返回给app

    系统通过代理告诉app接收到的token

    app把这个token发送给服务端。

    token其实就是当前这个app在APNS中的唯一标识。服务端拿着这个token，向APNS服务发送推送请求

绝对不要在app里面缓存token，应该直接从系统获取。在某些情况下APNS为了保证token的唯一性会更新app的token。比如用户从备份中恢复系统、用户在全新的设备上安装了app，用户重新安装了操作系统。当token在APNS没有改变的时候，去获取，APNS会快速的返回（也就是APNS或者系统其实做了缓存的，只是不要在app内部去做缓存）。

调用registerForRemoteNotifications注册APNS。在app启动的时候都会调用这个方法。系统会异步的回调回来。

    Token长度是不确定的，所以不要写死token的长度。在注册成功之后，只要当token改变之后才APP才和APNS再次交互。否则调用registerForRemoteNotifications的时候，application:didRegisterForRemoteNotificationsWithDeviceToken: 会立即返回已经存在的token。

如果token在app运行的时候 改变。app会直接调用application:didRegisterForRemoteNotificationsWithDeviceToken:。
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
	
- (void)applicationDidFinishLaunching:(UIApplication *)app {
    // Configure the user interactions first.
    [self configureUserInteractions];
  
   // Register for remote notifications.
    [[UIApplication sharedApplication] registerForRemoteNotifications];
}
  
// Handle remote notification registration.
- (void)application:(UIApplication *)app
        didRegisterForRemoteNotificationsWithDeviceToken:(NSData *)devToken {
    // Forward the token to your provider, using a custom method.
    [self enableRemoteNotificationFeatures];
    [self forwardTokenToServer:devTokenBytes];
}
  
- (void)application:(UIApplication *)app
        didFailToRegisterForRemoteNotificationsWithError:(NSError *)err {
    // The token is not currently available.
    NSLog(@"Remote notification support is unavailable due to error: %@", err);
    [self disableRemoteNotificationFeatures];
}

修改系统推送的展示方式

iOS10之后可以通过 notification service app extension 修改通知的展现方式。（原理就是在之前推送的基础上增加一层notification service app extension ，用于过滤服务端推送过来的数据，根据数据进行自定义）

分如下几个步骤：

    解密从服务端传过来的加密数据

    下载多媒体资源，并且作为通知附件

    改变通知的title和文本

    增加线程标识用于修改通知的userinfo参数

如何实现那notification service app extension网上很多，这里不累赘了。可以看看iOS10推送必看UNNotificationServiceExtension

一张图解释

注意事项

    同宿主target一样， app extension的target也需要在 notification service app extension的target中打开推送的capability才能正常收到推送。

该特性是在iOS7添加的，一句话来讲就是：应用在后台收到通知后能够运行一段代码，可用于从服务器获取内容更新。

静默推送和一般推送的区别：

iOS7之前

iOS7之后

    如果只携带content-available: 1 不携带任何badge，sound 和消息内容等参数，则可以不打扰用户的情况下进行内容更新等操作即为“Silent Remote Notifications”

具体设置如下：

当启用Backgroud Modes -> Remote notifications 后，notification 处理函数一律切换到下面函数，后台推送代码也在此函数中调用
1
2
3
4
	
/*! This delegate method offers an opportunity for applications with the "remote-notification" background mode to fetch appropriate new data in response to an incoming remote notification. You should call the fetchCompletionHandler as soon as you're finished performing that operation, so the system can accurately estimate its power and data cost.
  
 This method will be invoked even if the application was launched or resumed because of the remote notification. The respective delegate methods will be invoked first. Note that this behavior is in contrast to application:didReceiveRemoteNotification:, which is not called in those cases, and which will not be invoked if this method is implemented. !*/
- (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo fetchCompletionHandler:(void (^)(UIBackgroundFetchResult result))completionHandler NS_AVAILABLE_IOS(7_0);

这里提到了application:didReceiveRemoteNotification，看了一下还有这些知识点。
1
	
- (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo NS_DEPRECATED_IOS(3_0, 10_0, "Use UserNotifications Framework's -[UNUserNotificationCenterDelegate willPresentNotification:withCompletionHandler:] or -[UNUserNotificationCenterDelegate didReceiveNotificationResponse:withCompletionHandler:] for user visible notifications and -[UIApplicationDelegate application:didReceiveRemoteNotification:fetchCompletionHandler:] for silent remote notifications");

一定要注意这几个方法是属于不同类的

    application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo fetchCompletionHandler:(void (^)(UIBackgroundFetchResult result))completionHandler ：是属于AppDelegate，用于静默推送，用户不可见。

    其余两个是是属于UNUserNotificationCenter，用于用户可见的推送，比如有提示横幅。

注意事项

    Silent Remote Notifications是在 Apple 的限制下有一定的频率控制，但具体频率不详。所以并不是所有的 “Silent Remote Notifications” 都能按照预期到达客户端触发函数。

    Background下提供给应用的运行时间窗是有限制的，如果需要下载较大的文件请参考 Apple 的 NSURLSession 的介绍。

    Background Remote Notification 的前提是要求客户端处于Background 或 Suspended 状态，如果用户通过 App Switcher 将应用从后台 Kill 掉应用将不会唤醒应用处理 background 代码。（这一点通过voip Push可以实现）

    静默推送格式：

    增加content-available字段，并设成1

    alert字段必须为空，否则收到的就不是静默推送

    sound字段设不设不影响静默推送的接收。

    badge字段设不设不影响静默推送的接收。

    再次强调：静默推送只能在应用在前台和应用在后台挂起时收到，也就是说，如果应用未启动或进程被杀掉，静默推送是唤醒不了设备的。

    推送回调方法

    willPresentNotification:withCompletionHandler 用于前台运行

    didReceiveNotificationResponse:withCompletionHandler 用于后台及程序退出

    didReceiveRemoteNotification:fetchCompletionHandler用于静默推送

    (void)application:(UIApplication *)application didReceiveRemoteNotification（会处理所有推送，iOS10之前）

Voip推送（Voip Push）

Voip Push相对于Silent Push又更近一步。解决了Silent Push使用场景必须是app必须存活的问题。Voip Push可以在应用被杀死的情况下，也可以唤醒app。

基本介绍

苹果在iOS8引入PushKit framework，app可以通过远程推送就可以唤醒，不过这个特性只有在voip类应用起作用。在之前voip类app需要在后台保持长连接，才能实时收到voip，这样非常耗电，于是在iOS8之后引入了Push Kit解决这个问题。

之前有人分析过what's up 在iOS8之后的变化，原文入下：

    每次用户有新的离线消息，普通文本或者是voip call，app都会先被后台唤醒，再从server拉取离线消息，最后生成local push。等用户点击local push启动app的时候，没有启动页面，没有connecting和loading，所有的数据已经准备就绪，就好像WhatsApp一直在后台运行一样。

voip 推送和传统的推送的区别如下：

通过上图可以知道voip push有着自己完整的一套逻辑，与此同时同样需要申请相关证书。可以把voip push 、 远程push 与本地 push三者视为同一级别。

下图是voip push 对应的证书

代码来讲实现下面两个代理就行了
1
2
3
4
5
6
7
8
9
10
11
12
	
- (void)pushRegistry:(PKPushRegistry *)registry didUpdatePushCredentials:(PKPushCredentials *)pushCredentials forType:(PKPushType)type {
    NSData *voipToken = pushCredentials.token;
    NSLog(@"voip token:%@", voipToken);
     
}
- (void)pushRegistry:(PKPushRegistry *)registry didReceiveIncomingPushWithPayload:(PKPushPayload *)payload forType:(NSString *)type {
    //用户处理
    // 呼出系统接听界面
     
     
    // 或者生成本地推送
}

注意事项

    voip Push和remote Push 在token获取、服务端交互是大致相同的。

    voip Push和silent Push最大的不同在于，silent Push必须是app没有被杀死才能在后台唤醒应用，而voip Push是可以在app杀死之后，唤醒应用。

    voip Push一般使用方式是

    服务端发出推送，app收到推送

    app解析数据，根据数据先处理好业务逻辑。比如更新数据

    通过本地推送告诉用户

    用户打开app，显示之前已经处理好的内容

苹果 Push的优化记录

去了解一项技术的进化史，最好的方式就是去看官方的文档更新记录。苹果 Push的优化记录可以看到苹果本地推送和远程推送文档更新记录。

写在最后

花了将近两天的时间总结Push这块知识。过程中甚至感受到了苹果为提升用户体验而做的努力。从最开始的只要应用不在前台就无法唤醒的普通Push，到iOS7的新增的只能在后台唤醒app静默Push，再到后面iOS8新增的即使app没有运行也可以唤醒app的voip Push。Push技术的演变也是苹果对用户体验提升的见证。

回到作为技术人的角度，最近大半年都过得太过于浮躁。之前的炒得风风火火的数字货币，ICO项目等让自己太过于浮躁。直到前不久认识了一个技术大牛，才让自己认识到做技术还是需要沉淀，还是要形成知识体系，静下心来好好打磨。

内心激荡着那句对我说的话：“多看看计算机原理、底层相关的书。遇到问题多想几个为什么”。虽然这句话有装逼嫌疑，但是不得不说是走向技术大牛的最佳途径。

最近打算出一系列关于计算机底层、原理的文章。敬请期待！！！！
