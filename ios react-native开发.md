# ios react-native开发
## 模块开发
模块用于native实现需要的功能，js端调用原生实现的功能，比如网络请求，数据库存取。
## 声明模块 
在React Native中，一个“原生模块”就是一个实现了“RCTBridgeModule”协议的Objective-C类，其中RCT是ReaCT的缩写。<br>
```javascript
// CalendarManager.h
#import "RCTBridgeModule.h"

@interface CalendarManager : NSObject <RCTBridgeModule>
@end
```
你必须明确的声明要给Javascript导出的方法，否则React Native不会导出任何方法。声明通过RCT_EXPORT_METHOD()宏来实现：
```javascript
RCT_EXPORT_METHOD(addEvent:(NSString *)name location:(NSString *)location)
{
  RCTLogInfo(@"Pretending to create an event %@ at %@", name, location);
}
```

js端可以这样调用native导出的方法:
```javascript
var CalendarManager = require('react-native').NativeModules.CalendarManager;
CalendarManager.addEvent('Birthday Party', '4 Privet Drive, Surrey');
```
### 回调函数
原生模块还支持一种特殊的参数——回调函数。它提供了一个函数来把返回值传回给JavaScript。
```javascript
RCT_EXPORT_METHOD(findEvents:(RCTResponseSenderBlock)callback)
{
  NSArray *events = ...
  callback(@[[NSNull null], events]);
}
```
RCTResponseSenderBlock只接受一个参数——传递给JavaScript回调函数的参数数组。在上面这个例子里我们用Node.js的常用习惯：第一个参数是一个错误对象（没有发生错误的时候为null），而剩下的部分是函数的返回值。
```javascript
CalendarManager.findEvents((error, events) => {
  if (error) {
    console.error(error);
  } else {
    this.setState({events: events});
  }
})
```
原生模块通常只应调用回调函数一次。但是，它可以保存callback并在将来调用。这在封装那些通过“委托函数”来获得返回值的iOS API时最为常见。RCTAlertManager中就属于这种情况。

### 导出常量
原生模块可以导出一些常量，这些常量在JavaScript端随时都可以访问。用这种方法来传递一些静态数据，可以避免通过bridge进行一次来回交互。
```javacript
- (NSDictionary *)constantsToExport
{
  return @{ @"firstDayOfTheWeek": @"Monday" };
}

```
Javascript端可以随时同步地访问这个数据：
```javacript
console.log(CalendarManager.firstDayOfTheWeek);
```
但是注意这个常量仅仅在初始化的时候导出了一次，所以即使你在运行期间改变constantToExport返回的值，也不会影响到JavaScript环境下所得到的结果。


## native发送消息给js端
### 模块中发送消息
即使没有被JavaScript调用，本地模块也可以给JavaScript发送事件通知。由于在AppDelegate中已经有RCTRootView变量，可以直接使用RCTRootView的bridge。eventDispatcher执行sendAppEventWithName方法。最直接的方式是使用eventDispatcher:
```javascript
#import "RCTBridge.h"
#import "RCTEventDispatcher.h"

@implementation CalendarManager

@synthesize bridge = _bridge;

- (void)calendarEventReminderReceived:(NSNotification *)notification
{
  NSString *eventName = notification.userInfo[@"name"];
  //使用EventDispatcher的sendAppEventWithName发送消息
  //EventReminder是消息的名字
  //body为传递的数据字典.
  [self.bridge.eventDispatcher sendAppEventWithName:@"EventReminder"
                                               body:@{@"name": eventName}];
}

@end

```
js端调用方法
```javascript
var CustomSwiftComponent = React.createClass({
    //增加监听器
    componentWillMount: function() {
        //第一个参数为事件的名称，第二个参数为事件参数的处理方法
        this.subscription = NativeAppEventEmitter.addListener(
            'EventReminder',
            (reminder) => console.log(reminder.name)
        );
    },
    //移除监听器
    componentWillUnMount: function() {
        this.subscription.remove();
    }
}
```
### 界面中发送消息
如果界面中有rootView则可以调用rootView的bridge发送消息
```javascript
//swift版本
self.rootView!.bridge.eventDispatcher.sendAppEventWithName("EventReminder", body: ["name": "HelloEvent"])
//objectivie-c
 [self.rootView.bridge.eventDispatcher sendAppEventWithName:@"EventReminder"
                                                        body:@{
                                                          @"name" : @"Hello"
                                                        }];
```

## swift导出模块
Swift不支持宏，所以从Swift向React Native导出类和函数需要多做一些设置，但是大致与Objective-C是相同的。<br>
假设我们已经有了一个一样的CalendarManager，不过是用Swift实现的类:
```javascript
// CalendarManager.swift

@objc(CalendarManager)
class CalendarManager: NSObject {

  @objc func addEvent(name: String, location: String, date: NSNumber) -> Void {
    // Date is ready to use!
  }

}
```
> **注意**: 你必须使用@objc标记来确保类和函数对Objective-C公开。
接着，创建一个私有的实现文件，并将必要的信息注册到React Native中。
```javascript
// CalendarManagerBridge.m
#import "RCTBridgeModule.h"

@interface RCT_EXTERN_MODULE(CalendarManager, NSObject)

RCT_EXTERN_METHOD(addEvent:(NSString *)name location:(NSString *)location date:(nonnull NSNumber *)date)

@end

```
请注意，一旦你在IOS中混用2种语言, 你还需要一个额外的桥接头文件，称作“bridging header”，用来导出Objective-C文件给Swift。如果你是通过Xcode菜单中的File>New File来创建的Swift文件，Xcode会自动为你创建这个头文件。在这个头文件中，你需要引入RCTBridgeModule.h。
```javascript
// CalendarManager-Bridging-Header.h
#import "RCTBridgeModule.h"

```

## 原生的UI控件
原生视图都需要被一个RCTViewManager的子类来创建和管理。这些管理器在功能上有些类似“视图控制器”，但它们本质上都是单例 - React Native只会为每个管理器创建一个实例。它们创建原生的视图并提供给RCTUIManager，RCTUIManager则会反过来委托它们在需要的时候去设置和更新视图的属性。RCTViewManager还会代理视图的所有委托，并给JavaScript发回对应的事件。<br>
提供原生视图很简单：
1. 首先创建一个子类
2. 添加RCT_EXPORT_MODULE()标记宏
3. 实现-(UIView *)view方法
### 自定义控件
```javascript
//MyCustomView.h
#import <UIKit/UIKit.h>

#import "RCTComponent.h"

@interface MyCustomView : UIControl
//定义属性，isRed
@property(nonatomic) BOOL isRed;
//onTap事件
@property(nonatomic, copy) RCTBubblingEventBlock onTap;

@end


#import "MyCustomView.h"

@implementation MyCustomView {
  UIColor *squareColor;
}
@synthesize isRed;
//set方法
- (void)setIsRed:(BOOL)red {
  NSLog(@"isRed3:%d", red);
  isRed = red;
  NSLog(@"isRed4:%d", red);
  squareColor = (isRed) ? [UIColor redColor] : [UIColor greenColor];
  [self setNeedsDisplay];
}
//绘制
- (void)drawRect:(CGRect)rect {
  [squareColor setFill];
  CGContextFillRect(UIGraphicsGetCurrentContext(), rect);
}

@end

```
### 自定义RCTViewManager
```javascript
#import <UIKit/UIKit.h>

#import "RCTViewManager.h"

@interface MyCustomViewManager : RCTViewManager

@end


#import "MyCustomView.h"
#import "MyCustomViewManager.h"
#import <UIKit/UIKit.h>

@implementation MyCustomViewManager
//表明是一个module
RCT_EXPORT_MODULE();
//导出属性
RCT_EXPORT_VIEW_PROPERTY(isRed, BOOL)
RCT_EXPORT_VIEW_PROPERTY(onTap, RCTBubblingEventBlock)
//设置管理的View，并初始化View
- (UIView *)view {
  MyCustomView *theView = [[MyCustomView alloc] init];
  [theView addTarget:self
                action:@selector(onTaped:)
      forControlEvents:UIControlEventTouchUpInside];
  return theView;
}
//点击事件响应操作，native处理
- (void)onTaped:(MyCustomView *)sender {
  NSLog(@"isRed1:%d", sender.isRed);
  sender.isRed = sender.isRed ? FALSE : TRUE;
  NSLog(@"isRed2:%d", sender.isRed);
  //使用自定义View的onTap方法，传递字典到js
  if (sender.onTap != nil) {
    sender.onTap(@{});
  }
}
@end

```
### 定义js端控件

```javscript
'use strict'
import React, {
  requireNativeComponent,
  View
} from 'react-native'
class CustomView extends  React.Component{
  constructor(props) {
    super(props);
    //绑定事件
      this._onChange = this._onChange.bind(this);
  }
  _onChange(event:Event) {
      console.log("hello")
      //  this._rctSwitch.setNativeProps({isRed: !this._rctSwitch.isRed});
  }
  _rctSwitch: {}
  render() {
    return <MyCustomView {...this.props} style={{width: 100, height: 100}} isRed={false} onTap={this._onChange}
        ref={(ref) => { this._rctSwitch = ref; }}/>
  }
}
//属性
CustomView.propTypes = {
   ...View.propTypes,
  /**
   * isRed
   */
  isRed: React.PropTypes.bool,
  /**
  * tap
  */
  onTap:React.PropTypes.func,
}

var MyCustomView = requireNativeComponent("MyCustomView",CustomView)
export { CustomView as default };
```
### 使用
```javscript
//引用
var CustomView = require('./CustomView').default;
//显示
<CustomView style={{width: 100, height: 100}}/>
```

