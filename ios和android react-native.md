# ios和android react-native生成原生模块，原生控件对比
## 原生模块
### 模块类
#### ios
实现RCTBridgeModule接口，在.h代码如下所示:
```javascript
#import <Foundation/Foundation.h>
#import <RCTBridgeModule.h>
@interface CalendarManager : NSObject <RCTBridgeModule>

@end
```
.m文件

```javascript
#import "CalendarManager.h"
#import "RCTBridge.h"
#import "RCTEventDispatcher.h"
#import "RCTLog.h"

@implementation CalendarManager
//bridge
@synthesize bridge = _bridge;
//表明是一个模块，参数为模块的名称，不带参数则使用类的名字作为模块名称
RCT_EXPORT_MODULE();
//导出的方法
RCT_EXPORT_METHOD(addEvent : (NSString *)name loaction : (NSString *)location) {
  RCTLogInfo(@"Pretending to create an event %@ at %@", name, location);
}

//导出的方法
RCT_EXPORT_METHOD(findEvents : (RCTResponseSenderBlock)callback) {
  NSArray *datas = @[ @"a", @"b" ];
  //带有回调方法，对象数组
  callback(@[ [NSNull null], datas ]);
}
//导出的常量
- (NSDictionary *)constantsToExport {
  return @{ @"firstDayOfTheWeek" : @"Monday" };
}
//发送消息到js
- (void)calendarEventReminderReceived:(NSNotification *)notification {
  NSString *eventName = notification.userInfo[@"name"];
  [self.bridge.eventDispatcher sendAppEventWithName:@"EventReminder"
                                               body:@{
                                                 @"name" : eventName
                                               }];
}
@end
```
#### android
##### 继承Module类

```javscript

import android.util.Log;
import android.widget.Toast;

import com.facebook.react.bridge.Callback;
import com.facebook.react.bridge.ReactApplicationContext;
import com.facebook.react.bridge.ReactContextBaseJavaModule;
import com.facebook.react.bridge.ReactMethod;
import com.facebook.react.uimanager.IllegalViewOperationException;

import java.util.HashMap;
import java.util.Map;

public class ToastCustomModule extends ReactContextBaseJavaModule {

    private static final String DURATION_SHORT = "SHORT";
    private static final String DURATION_LONG = "LONG";

    public ToastCustomModule(ReactApplicationContext reactContext) {
        super(reactContext);
    }
    //模块名称
    @Override
    public String getName() {
        return "ToastCustomAndroid";
    }
    //返回的常量
    @Override
    public Map<String, Object> getConstants() {
        final Map<String, Object> constants = new HashMap<>();
        constants.put(DURATION_SHORT, Toast.LENGTH_SHORT);
        constants.put(DURATION_LONG, Toast.LENGTH_LONG);
        return constants;
    }

    /**
     * 该方法用于给JavaScript进行调用
     *
     * @param message
     * @param duration
     */
    @ReactMethod
    public void show(String message, int duration) {
        Log.d("ToastCustomModule", "show: " + duration);
        Toast.makeText(getReactApplicationContext(), message, duration).show();
    }

    /**
     * 这边只是演示相关回调方法的使用,所以这边的使用方法是非常简单的
     *
     * @param errorCallback   数据错误回调函数
     * @param successCallback 数据成功回调函数
     */
    @ReactMethod
    public void measureLayout(Callback errorCallback,
                              Callback successCallback) {
        try {
            successCallback.invoke(100, 100, 200, 200);
        } catch (IllegalViewOperationException e) {
            errorCallback.invoke(e.getMessage());
        }
    }
}

```

##### ReactPackage

```javascript

import com.facebook.react.ReactPackage;
import com.facebook.react.bridge.JavaScriptModule;
import com.facebook.react.bridge.NativeModule;
import com.facebook.react.bridge.ReactApplicationContext;
import com.facebook.react.uimanager.ViewManager;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
public class AnToastReactPackage implements ReactPackage {
    //管理模块
    @Override
    public List<NativeModule> createNativeModules(ReactApplicationContext reactContext) {
        List<NativeModule> modules = new ArrayList<>();
        modules.add(new ToastCustomModule(reactContext));
        return modules;
    }

    @Override
    public List<Class<? extends JavaScriptModule>> createJSModules() {
        return Collections.emptyList();
    }

    @Override
    public List<ViewManager> createViewManagers(ReactApplicationContext reactContext) {
        return Collections.emptyList();
    }
}

```

##### 初始化ReactInstanceManager

```javascript

protected List<ReactPackage> getPackages() {
        return Arrays.<ReactPackage>asList(
            new MainReactPackage(),
                new AnToastReactPackage()
        );
    }
 ReactInstanceManager初始化的时候设置其package为getPckage返回数组。
 
  ReactInstanceManager.Builder builder = ReactInstanceManager.builder()
        .setApplication(getApplication())
        .setJSMainModuleName(getJSMainModuleName())
        .setUseDeveloperSupport(getUseDeveloperSupport())
        .setInitialLifecycleState(mLifecycleState);
    
    for (ReactPackage reactPackage : getPackages()) {
      builder.addPackage(reactPackage);
    }
    
```
### js端
#### 封装module
```javascript
'use strict'

var {
	NativeModules
} = require('react-native');
module.exports = NativeModules.CalendarManager;

#### 引用的js封装的module
//引入模块
import CalendarManager from './CalendarManager'

CalendarManager.addEvent('Birthday party', '4 privt drive surrey');
//error表示错误信息，datas表示native传递过来的数据，使用回调
CalendarManager.findEvents((error, datas) => {
	console.log(error);
	console.log(datas);
});

console.log(CalendarManager.firstDayOfTheWeek);
```

## 原生控件
### ios
#### 自定义控件

```javascript
//MyCustomView.h
#import <UIKit/UIKit.h>

#import "RCTComponent.h"

@interface MyCustomView : UIControl
//定义的两个属性
@property(nonatomic) BOOL isRed;
//事件
@property(nonatomic, copy) RCTBubblingEventBlock onTap;
@end

//MyCustomView.m
#import "MyCustomView.h"

@implementation MyCustomView {
  UIColor *squareColor;
}
@synthesize isRed;
- (void)setIsRed:(BOOL)red {
  NSLog(@"isRed3:%d", red);
  isRed = red;
  NSLog(@"isRed4:%d", red);
  squareColor = (isRed) ? [UIColor redColor] : [UIColor greenColor];
  [self setNeedsDisplay];
}

- (void)drawRect:(CGRect)rect {
  [squareColor setFill];
  CGContextFillRect(UIGraphicsGetCurrentContext(), rect);
}

@end
```
#### ViewManager

```javascript
//MyCustomViewManager.h
#import <UIKit/UIKit.h>

#import "RCTViewManager.h"

@interface MyCustomViewManager : RCTViewManager

@end

//MyCustomViewManager.m
#import "MyCustomView.h"
#import "MyCustomViewManager.h"
#import <UIKit/UIKit.h>

@implementation MyCustomViewManager
//导出为module
RCT_EXPORT_MODULE();
//导出属性
RCT_EXPORT_VIEW_PROPERTY(isRed, BOOL)
RCT_EXPORT_VIEW_PROPERTY(onTap, RCTBubblingEventBlock)
//导出的View
- (UIView *)view {
  MyCustomView *theView = [[MyCustomView alloc] init];
  [theView addTarget:self
                action:@selector(onTaped:)
      forControlEvents:UIControlEventTouchUpInside];
  return theView;
}
- (void)onTaped:(MyCustomView *)sender {
  NSLog(@"isRed1:%d", sender.isRed);
  sender.isRed = sender.isRed ? FALSE : TRUE;
  NSLog(@"isRed2:%d", sender.isRed);
  //回调自定义view的onTap方法，会调用js的方法,传递的数据为空的字典
  if (sender.onTap != nil) {
    sender.onTap(@{});
  }
}
@end
```
### android
#### 自定义View
与普通的自定义View方法相同。
#### 定义View的事件
```javascript

import com.facebook.react.uimanager.events.Event;
import com.facebook.react.uimanager.events.RCTEventEmitter;
public class ReactBGABadgeEvent extends Event<ReactBGABadgeEvent> {
    public static final String EVENT_NAME = "topDismiss";

    protected ReactBGABadgeEvent(int viewTag, long timestampMs) {
        super(viewTag, timestampMs);
    }
    //事件名称
    @Override
    public String getEventName() {
        return EVENT_NAME;
    }
    //事件分发
    @Override
    public void dispatch(RCTEventEmitter rctEventEmitter) {
        rctEventEmitter.receiveEvent(getViewTag(), getEventName(), null);
    }

}
```

#### ViewManager
```javascript

import android.content.Context;
import android.graphics.Bitmap;
import android.graphics.BitmapFactory;
import android.graphics.Color;
import android.os.SystemClock;
import android.text.TextUtils;

import com.facebook.react.common.MapBuilder;
import com.facebook.react.uimanager.SimpleViewManager;
import com.facebook.react.uimanager.ThemedReactContext;
import com.facebook.react.uimanager.UIManagerModule;
import com.facebook.react.uimanager.annotations.ReactProp;

import java.util.Map;

import cn.bingoogolapple.badgeview.BGABadgeView;
import cn.bingoogolapple.badgeview.BGABadgeable;
import cn.bingoogolapple.badgeview.BGADragDismissDelegate;
public class ReactBGABadgeViewManager extends SimpleViewManager<BGABadgeView> {
    private static final String REACT_CLASS = "AndroidBGABadgeView";

    @Override
    public String getName() {
        return REACT_CLASS;
    }

    @Override
    protected BGABadgeView createViewInstance(ThemedReactContext reactContext) {
        BGABadgeView badgeView = new BGABadgeView(reactContext);
        badgeView.getBadgeViewHelper().setDragable(true);
        return badgeView;
    }

    @ReactProp(name = "badgeBgColor")
    public void setBadgeBgColor(BGABadgeView view, String color) {
        if (!TextUtils.isEmpty(color)) {
            view.getBadgeViewHelper().setBadgeBgColorInt(Color.parseColor(color));
        }
    }
    //分发事件需要重写的方法
    @Override
    protected void addEventEmitters(final ThemedReactContext reactContext, final BGABadgeView view) {
        view.setDragDismissDelegage(new BGADragDismissDelegate() {
            @Override
            public void onDismiss(BGABadgeable badgeable) {
                // 一定要用view.getId()，不能用badgeable.getId()，这个坑踩了半天
                reactContext.getNativeModule(UIManagerModule.class).getEventDispatcher()
                        .dispatchEvent(new ReactBGABadgeEvent(view.getId(), SystemClock.uptimeMillis()));
            }
        });
    }
    //事件对应于js模块的属性字典，也就是topDismiss对应于onDismiss属性
    @Override
    public Map<String, Object> getExportedCustomDirectEventTypeConstants() {
        return MapBuilder.<String, Object>builder()
                .put(ReactBGABadgeEvent.EVENT_NAME, MapBuilder.of("registrationName", "onDismiss"))
                .build();
    }

}
```

##### ReactPackage
```javascript
import com.facebook.react.ReactPackage;
import com.facebook.react.bridge.JavaScriptModule;
import com.facebook.react.bridge.NativeModule;
import com.facebook.react.bridge.ReactApplicationContext;
import com.facebook.react.uimanager.ViewManager;

import java.util.Arrays;
import java.util.List;

public class BGABadgeViewReactPackage implements ReactPackage {

    @Override
    public List<NativeModule> createNativeModules(ReactApplicationContext reactContext) {
        return Arrays.asList();
    }

    @Override
    public List<Class<? extends JavaScriptModule>> createJSModules() {
        return Arrays.asList();
    }

    @Override
    public List<ViewManager> createViewManagers(ReactApplicationContext reactContext) {
        return Arrays.<ViewManager>asList(new ReactBGABadgeViewManager());
    }
}
```

### js端代码
#### js模块
```javascript
'use strict';

import React, {
  Component,
  View,
  requireNativeComponent,
  PropTypes
} from 'react-native';

class BGABadgeViewAndroid extends Component {
  constructor(props) {
    super(props);
  }

  _onDismiss() {
    if (this.props.onDismiss) {
      this.props.onDismiss();
    }
  }
  render() {
      //使用native的控件,设置属性，onDismiss事件
    return <AndroidBGABadgeView {...this.props} onDismiss={this._onDismiss.bind(this)} />;
  }
}

BGABadgeViewAndroid.propTypes = {
  ...View.propTypes,
  textBadge: PropTypes.string,
  badgeBgColor: PropTypes.string,
  badgeTextColor: PropTypes.string,
  drawableBadge: PropTypes.string,
  circlePointBadge: PropTypes.bool,
  dragable: PropTypes.bool,
  onDismiss: PropTypes.func,
  badgeTextSizeSp: PropTypes.number,
  badgePaddingDp: PropTypes.number,
  badgeBorderWidthDp: PropTypes.number,
  badgeBorderColor: PropTypes.string,
};
//引用控件，requireNativeComponent第一个参数为native的类名称，第二个参数为本js定义的类名称，返回值为native的类名称
var AndroidBGABadgeView = requireNativeComponent(`AndroidBGABadgeView`, BGABadgeViewAndroid);
//导出本js定义的类名称
export { BGABadgeViewAndroid as default };
```
#### 其他js文件引用
```javascript
import BGABadgeViewAndroid from './app/Components/BGABadge/BGABadgeViewAndroid';

<BGABadgeViewAndroid dragable={false} textBadge={this.state.badgeText} style={styles.badge} onDismiss={() => this.onDismiss("第1个")} />

```