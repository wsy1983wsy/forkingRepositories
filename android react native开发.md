# android react-native开发
## 开发环境配置
[参考1](http://reactnative.cn/docs/0.24/getting-started.html#content)
[参考2](http://www.lcode.org/react-native/)
### 下载后的代码如何配置
在代码的根目录执行npm install，即可配置好需要的react-native
## 如何让ReactRootView作为activity的一部分
自定义activity的布局的xml，将ReactRootView作为其中一部分，重写Activity的onCreate方法，代码如下
```javascript

public class HelloWorldActivity extends Activity implements DefaultHardwareBackBtnHandler {

    private static final String REDBOX_PERMISSION_MESSAGE =
            "Overlay permissions needs to be granted in order for react native apps to run in dev mode";

    private
    @Nullable
    ReactInstanceManager mReactInstanceManager;
    private LifecycleState mLifecycleState = LifecycleState.BEFORE_RESUME;
    private boolean mDoRefresh = false;
    
    //ReactRootView
    private ReactRootView rootView;


    protected
    @Nullable
    String getBundleAssetName() {
        //要加载的bundle名称
        return "index.android.bundle";
    }

    protected
    @Nullable
    String getJSBundleFile() {
        return null;
    }
    //要使用的package
    protected List<ReactPackage> getPackages() {
        return Arrays.<ReactPackage>asList(
                new MainReactPackage()
        );
    }

    protected
    @Nullable
    Bundle getLaunchOptions() {
        return null;
    }

    //要加载的js文件名称
    protected String getJSMainModuleName() {
        return "HelloWorld";
    }
    //要加载的js组件名称
    protected String getMainComponentName() {
        return "HelloWorld";
    }

    protected boolean getUseDeveloperSupport() {
        return BuildConfig.DEBUG;
    }

    protected ReactInstanceManager createReactInstanceManager() {
        //初始化ReactInstanceManager
        mReactInstanceManager = ReactInstanceManager.builder()
                .setApplication(getApplication())
                .setBundleAssetName(getBundleAssetName())
                .setJSMainModuleName(getJSMainModuleName())
                .addPackage(new MainReactPackage())
                .setUseDeveloperSupport(getUseDeveloperSupport())
                .setInitialLifecycleState(mLifecycleState)
                .build();
        return mReactInstanceManager;
    }

    /**
     * A subclass may override this method if it needs to use a custom {@link ReactRootView}.
     */
    protected ReactRootView createRootView() {
        return new ReactRootView(this);
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.helloworld);
        if (getUseDeveloperSupport() && Build.VERSION.SDK_INT >= 23) {
            // Get permission to show redbox in dev builds.
            if (!Settings.canDrawOverlays(this)) {
                Intent serviceIntent = new Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION);
                startActivity(serviceIntent);
                FLog.w(ReactConstants.TAG, REDBOX_PERMISSION_MESSAGE);
                Toast.makeText(this, REDBOX_PERMISSION_MESSAGE, Toast.LENGTH_LONG).show();
            }
        }

        mReactInstanceManager = createReactInstanceManager();
        rootView = (ReactRootView) findViewById(R.id.rootView);
        //显示view
        rootView.startReactApplication(mReactInstanceManager, "HelloWorld", getLaunchOptions());
    }

    @Override
    protected void onPause() {
        super.onPause();
        mLifecycleState = LifecycleState.BEFORE_RESUME;
        if (mReactInstanceManager != null) {
            mReactInstanceManager.onHostPause();
        }
    }

    @Override
    protected void onResume() {
        super.onResume();
        mLifecycleState = LifecycleState.RESUMED;
        if (mReactInstanceManager != null) {
            mReactInstanceManager.onHostResume(this, this);
        }
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        if (mReactInstanceManager != null) {
            mReactInstanceManager.destroy();
        }
    }

    @Override
    public void onActivityResult(int requestCode, int resultCode, Intent data) {
        if (mReactInstanceManager != null) {
            mReactInstanceManager.onActivityResult(requestCode, resultCode, data);
        }
    }
    //初见按键操作
    @Override
    public boolean onKeyUp(int keyCode, KeyEvent event) {
        if (mReactInstanceManager != null &&
                mReactInstanceManager.getDevSupportManager().getDevSupportEnabled()) {
            if (keyCode == KeyEvent.KEYCODE_MENU) {
                mReactInstanceManager.showDevOptionsDialog();
                return true;
            }
            if (keyCode == KeyEvent.KEYCODE_R && !(getCurrentFocus() instanceof EditText)) {
                // Enable double-tap-R-to-reload
                if (mDoRefresh) {
                    mReactInstanceManager.getDevSupportManager().handleReloadJS();
                    mDoRefresh = false;
                } else {
                    mDoRefresh = true;
                    new Handler().postDelayed(
                            new Runnable() {
                                @Override
                                public void run() {
                                    mDoRefresh = false;
                                }
                            },
                            200);
                }
            }
        }
        return super.onKeyUp(keyCode, event);
    }

    @Override
    public void onBackPressed() {
        if (mReactInstanceManager != null) {
            mReactInstanceManager.onBackPressed();
        } else {
            super.onBackPressed();
        }
    }

    @Override
    public void invokeDefaultOnBackPressed() {
        super.onBackPressed();
    }

    public void sendMessage(View view) {
        Hello hell0 = new Hello();
        hell0.age = 10;
        hell0.name = "1111";
        com.facebook.react.bridge.WritableNativeMap map = new WritableNativeMap();
        map.putInt("age", hell0.age);
        map.putString("name", hell0.name);
        mReactInstanceManager.getCurrentReactContext().getJSModule(DeviceEventManagerModule.RCTDeviceEventEmitter.class)
                .emit("Hello", map);
    }

    public static class Hello {
        public int age;
        public String name;
    }
}

```

## 如何让js调用android编写的程序
### 自定义模块
自定义模块可以让js调用android提供的数据，网络请求等服务。
#### 自定义module
继承自ReactContextBaseJavaModule，示例代码如下
```javascript
package com.facebook.react.modules.toast;

import android.widget.Toast;

import com.facebook.react.bridge.NativeModule;
import com.facebook.react.bridge.ReactApplicationContext;
import com.facebook.react.bridge.ReactContext;
import com.facebook.react.bridge.ReactContextBaseJavaModule;
import com.facebook.react.bridge.ReactMethod;

import java.util.Map;

public class ToastModule extends ReactContextBaseJavaModule {

  private static final String DURATION_SHORT_KEY = "SHORT";
  private static final String DURATION_LONG_KEY = "LONG";

  public ToastModule(ReactApplicationContext reactContext) {
    super(reactContext);
  }
}
//js调用的模块名称,通过React.NativeModules.ToastAndroid访问
@Override
  public String getName() {
    return "ToastAndroid";
  }
  //getContants返回了需要导出给JavaScript使用的常量。它并不一定需要实现，但在定义一些可以被JavaScript同步访问到的预定义的值时非常有用。
   @Override
  public Map<String, Object> getConstants() {
    final Map<String, Object> constants = new HashMap<>();
    constants.put(DURATION_SHORT_KEY, Toast.LENGTH_SHORT);
    constants.put(DURATION_LONG_KEY, Toast.LENGTH_LONG);
    return constants;
  }
  //要导出一个方法给JavaScript使用，Java方法需要使用注解@ReactMethod。方法的返回类型必须为void
  @ReactMethod
  public void show(String message, int duration) {
    Toast.makeText(getReactApplicationContext(), message, duration).show();
  }
```
### 注册模块
```javascript
class AnExampleReactPackage implements ReactPackage {

  @Override
  public List<Class<? extends JavaScriptModule>> createJSModules() {
    return Collections.emptyList();
  }

  @Override
  public List<ViewManager> createViewManagers(ReactApplicationContext reactContext) {
    return Collections.emptyList();
  }

  @Override
  public List<NativeModule> createNativeModules(
                              ReactApplicationContext reactContext) {
    List<NativeModule> modules = new ArrayList<>();

    modules.add(new ToastModule(reactContext));

    return modules;
  }

```
### 将package保存起来
需要在初始化ReactInstanceManager的时候设置
```javascript
protected List<ReactPackage> getPackages() {
    return Arrays.<ReactPackage>asList(
            new MainReactPackage(),
            new AnExampleReactPackage()); // <-- 添加这一行，类名替换成你的Package类的名字.
}
```
### js如何调用
新建一个js文件为ToastAndroid.js，内容为：
```javascript
var { NativeModules } = require('react-native');
//ToastAndroid在ToastModule中通过getName获取
module.exports = NativeModules.ToastAndroid;
```
在其他js文件中调用
```javascript
var ToastAndroid = require('./ToastAndroid')
ToastAndroid.show('Awesome', ToastAndroid.SHORT);
```
## 回调函数
原生模块还支持一种特殊的参数——回调函数。它提供了一个函数来把返回值传回给JavaScrip
```javascript
public class UIManagerModule extends ReactContextBaseJavaModule {

...
    //Callback为回调函数方法接口，其有一个invoke方法，参数为可变参数。
  @ReactMethod
  public void measureLayout(
      int tag,
      int ancestorTag,
      Callback errorCallback,
      Callback successCallback) {
    try {
      measureLayout(tag, ancestorTag, mMeasureBuffer);
      float relativeX = PixelUtil.toDIPFromPixel(mMeasureBuffer[0]);
      float relativeY = PixelUtil.toDIPFromPixel(mMeasureBuffer[1]);
      float width = PixelUtil.toDIPFromPixel(mMeasureBuffer[2]);
      float height = PixelUtil.toDIPFromPixel(mMeasureBuffer[3]);
      successCallback.invoke(relativeX, relativeY, width, height);
    } catch (IllegalViewOperationException e) {
      errorCallback.invoke(e.getMessage());
    }
  }

...
```
js调用
```javascript
UIManager.measureLayout(
  100,
  100,
  (msg) => {
    console.log(msg);
  },
  (x, y, width, height) => {
    console.log(x + ':' + y + ':' + width + ':' + height);
  }
);
```
## 如何向js发送消息
### android端发送消息
```javascript
public void sendMessage(View view) {
        Hello hell0 = new Hello();
        hell0.age = 10;
        hell0.name = "1111";
        com.facebook.react.bridge.WritableNativeMap map = new WritableNativeMap();
        map.putInt("age", hell0.age);
        map.putString("name", hell0.name);
        //通过使用RCTDeviceEventEmitter的emit方法发送消息，消息的名称为Hello，参数为map
        mReactInstanceManager.getCurrentReactContext().getJSModule(DeviceEventManagerModule.RCTDeviceEventEmitter.class)
                .emit("Hello", map);
    }
```
### js响应消息
js端代码如下
```javascript

import { DeviceEventEmitter } from 'react-native';

class HelloWorld extends Component {
	componentWillMount() {
        //msg为从android传递的参数
  	DeviceEventEmitter.addListener('Hello', (msg)=>console.log(msg));
	}
    
	render() {
		return (
			<View style={styles.Hello}>
         <Text>HelloWorld</Text>
			 </View>
		);
	}
}

```

## 自定义组件
自定义组件可以提供以前创建的控件或者自定义js不容易实现的空间等。
### 步骤
1. 创建一个ViewManager的子类。
2. 实现createViewInstance方法。
3. 导出视图的属性设置器：使用@ReactProp（或@ReactPropGroup）注解。
4. 把这个视图管理类注册到应用程序包的createViewManagers里。
5. 实现JavaScript模块。
1,2,3三个步骤都是在ViewManager子类中处理

### 创建ViewManager的子类
创建一个视图管理类ReactImageManager，它继承自SimpleViewManager<ReactImageView>。ReactImageView是这个视图管理类所管理的对象类型，这应当是一个自定义的原生视图。getName方法返回的名字会用于在JavaScript端引用这个原生视图类型。
```javascript
public class ReactImageManager extends SimpleViewManager<ReactImageView> {

  public static final String REACT_CLASS = "RCTImageView";

  @Override
  public String getName() {
    return REACT_CLASS;
  }
}
```
### ViewManager子类实现createViewInstance方法
视图在createViewInstance中创建，且应当把自己初始化为默认的状态。所有属性的设置都通过后续的updateView来进行。
```javascript
@Override
  public ReactImageView createViewInstance(ThemedReactContext context) {
      //初始化视图
    return new ReactImageView(context, Fresco.newDraweeControllerBuilder(), mCallerContext);
  }

```
###  导出视图的属性设置器：使用@ReactProp（或@ReactPropGroup）注解
要导出给JavaScript使用的属性，需要申明带有@ReactProp（或@ReactPropGroup）注解的设置方法。方法的第一个参数是要修改属性的视图实例，第二个参数是要设置的属性值。方法的返回值类型必须为void，而且访问控制必须被声明为public。JavaScript所得知的属性类型会由该方法第二个参数的类型来自动决定。支持的类型有：boolean, int, float, double, String, Boolean, Integer, ReadableArray, ReadableMap。<br>
@ReactProp注解必须包含一个字符串类型的参数name。这个参数指定了对应属性在JavaScript端的名字。<br>

除了name，@ReactProp注解还接受这些可选的参数：defaultBoolean, defaultInt, defaultFloat。这些参数必须是对应的基础类型的值（也就是boolean, int, float），这些值会被传递给setter方法，以免JavaScript端某些情况下在组件中移除了对应的属性。注意这个"默认"值只对基本类型生效，对于其他的类型而言，当对应的属性删除时，null会作为默认值提供给方法。<br>

使用@ReactPropGroup来注解的设置方法和@ReactProp不同。请参见@ReactPropGroup注解类源代码中的文档来获取更多详情。<br>

**重要！**在ReactJS里，修改一个属性会引发一次对设置方法的调用。有一种修改情况是，移除掉之前设置的属性。在这种情况下设置方法也一样会被调用，并且“默认”值会被作为参数提供（对于基础类型来说可以通过defaultBoolean、defaultFloat等@ReactProp的属性提供，而对于复杂类型来说参数则会设置为null）<br>
```javascript
@ReactProp(name = "src")
  public void setSrc(ReactImageView view, @Nullable String src) {
    view.setSource(src);
  }

  @ReactProp(name = "borderRadius", defaultFloat = 0f)
  public void setBorderRadius(ReactImageView view, float borderRadius) {
    view.setBorderRadius(borderRadius);
  }

  @ReactProp(name = ViewProps.RESIZE_MODE)
  public void setResizeMode(ReactImageView view, @Nullable String resizeMode) {
    view.setScaleType(ImageResizeMode.toScaleType(resizeMode));
  }

```
### 注册ViewManager
在Java中的最后一步就是把视图控制器注册到应用中。这和原生模块的注册方法类似，唯一的区别是我们把它放到createViewManagers方法的返回值里。
```javascript
@Override
  public List<ViewManager> createViewManagers(
                            ReactApplicationContext reactContext) {
    return Arrays.<ViewManager>asList(
      new ReactImageManager()
    );
  }
```
该方法在ReactPackage子类中处理，也就是说需要增加一个ReactPackage的子类。

### 实现对应的Javascript模块
整个过程的最后一步就是创建JavaScript模块并且定义Java和JavaScript之间的接口层。大部分过程都由React底层的Java和JavaScript代码来完成，你所需要做的就是通过propTypes来描述属性的类型。
```javascript
// ImageView.js

import { PropTypes } from 'react';
import { requireNativeComponent, View } from 'react-native';

var iface = {
  name: 'ImageView',
  propTypes: {
    src: PropTypes.string,
    borderRadius: PropTypes.number,
    resizeMode: PropTypes.oneOf(['cover', 'contain', 'stretch']),
    ...View.propTypes // 包含默认的View的属性
  },
};

module.exports = requireNativeComponent('RCTImageView', iface);
```
requireNativeComponent通常接受两个参数，第一个参数是原生视图的名字，而第二个参数是一个描述组件接口的对象。组件接口应当声明一个友好的name，用来在调试信息中显示；组件接口还必须声明propTypes字段，用来对应到原生视图上。这个propTypes还可以用来检查用户使用View的方式是否正确。<br>

注意，如果你还需要一个JavaScript组件来做一些除了指定name和propTypes以外的事情，譬如事件处理，你可以把原生组件用一个普通React组件封装。在这种情况下，reactNativeComponent的第二个参数变为用于封装的组件。这个在后文的MyCustomView例子里面用到。<br>

### 组件事件
#### 实现满足功能的自定义控件（java类）
####  实现需要的事件
继承自com.facebook.react.uimanager.events.Event。代码如下所示:
```javascript
public class ReactBGABadgeEvent extends Event<ReactBGABadgeEvent> {
    //事件名称，以top开头
    public static final String EVENT_NAME = "topDismiss";

    protected ReactBGABadgeEvent(int viewTag, long timestampMs) {
        super(viewTag, timestampMs);
    }

    @Override
    public String getEventName() {
        return EVENT_NAME;
    }
    //使用RCTEventEmitter发送事件到js
    @Override
    public void dispatch(RCTEventEmitter rctEventEmitter) {
        rctEventEmitter.receiveEvent(getViewTag(), getEventName(), null);
    }

}

```

#### 实现ViewManager
示例代码如下:
```javascript
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

   //增加事件的发射器，需要重写此方法
    @Override
    protected void addEventEmitters(final ThemedReactContext reactContext, final BGABadgeView view) {
        view.setDragDismissDelegage(new BGADragDismissDelegate() {
            @Override
            public void onDismiss(BGABadgeable badgeable) {
                // 一定要用view.getId()，不能用badgeable.getId()，这个坑踩了半天
                //发送ReactBGABadgeEvent事件
                reactContext.getNativeModule(UIManagerModule.class).getEventDispatcher()
                        .dispatchEvent(new ReactBGABadgeEvent(view.getId(), SystemClock.uptimeMillis()));
            }
        });
    }
    //设置事件对应的映射名称，需要将top给取消掉
    @Override
    public Map<String, Object> getExportedCustomDirectEventTypeConstants() {
        return MapBuilder.<String, Object>builder()
                .put(ReactBGABadgeEvent.EVENT_NAME, MapBuilder.of("registrationName", "onDismiss"))
                .build();
    }

    public static int getMipmap(Context context, String name) {
        return context.getResources().getIdentifier(name, "mipmap", context.getPackageName());
    }
}

```
#### js端处理
增加模块js文件
```javascript
'use strict';
//引用
import React, {
  Component,
  View,
  requireNativeComponent,
  PropTypes
} from 'react-native';
//自定义对应的js组件，使用原生的组件
class BGABadgeViewAndroid extends Component {
  constructor(props) {
    super(props);
  }
  //自定义事件，调用属在其他js文件中使用该控件时设置的onDismiss属性值。
  _onDismiss() {
    if (this.props.onDismiss) {
      this.props.onDismiss();
    }
  }
  //渲染组件，直接返回一个native控件，onDismiss与定义的_onDismiss方法绑定,此处增加了一个onDismiss属性，将AndroidBGABadgeView的onDismiss与_onDismiss绑定。
  render() {
    return <AndroidBGABadgeView {...this.props} onDismiss={this._onDismiss.bind(this)} />;
  }
}
//设置属性类型
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
//需要nativie组件，第一个参数是原生视图的名字，而第二个参数是一个描述组件接口的对象。AndroidBGABadgeView与ViewManager的geName返回一致
var AndroidBGABadgeView = requireNativeComponent(`AndroidBGABadgeView`, BGABadgeViewAndroid);
//导出组件
export { BGABadgeViewAndroid as default };

```

组件的使用：
```javascript
'use strict';
import React, {
  AppRegistry,
  Component,
  StyleSheet,
  Text,
  View,
  Image,
  ToastAndroid
} from 'react-native';
//上文定义的js组件
import BGABadgeViewAndroid from './app/Components/BGABadge/BGABadgeViewAndroid';

lass BGABadgeViewRN extends Component {

  onDismiss(tip) {
    ToastAndroid.show('消失 ' + tip, ToastAndroid.SHORT)
  }
  
  render() {
    return (
      <View style={styles.container}>
        <Text style={{paddingBottom: 30, fontSize: 22, color: '#50a3fc'}}>React Native - BGABadgeView</Text>
        //使用BGABadgeViewAndroid，onDismiss返回当前组件的onDismiss方法
        <BGABadgeViewAndroid badgeBgColor="#00ff00" badgeTextColor="#ff0000" badgePaddingDp={10} badgeTextSizeSp={14} textBadge="9" style={styles.badge} onDismiss={() => this.onDismiss("第2个")} />
      </View>
    );
  }
}

//注册组件
AppRegistry.registerComponent('BGABadgeViewRN', () => BGABadgeViewRN);
```


>参考文章

[React-Native系列Android——自定义View组件开发](http://blog.csdn.net/megatronkings/article/details/50959518)

[React Native进阶之原生模块封装基础篇1](http://www.lcode.org/react-native%E8%BF%9B%E9%98%B6%E4%B9%8B%E5%8E%9F%E7%94%9F%E6%A8%A1%E5%9D%97%E7%89%B9%E6%80%A7%E7%AF%87%E8%AF%A6%E8%A7%A3-%E9%80%82%E9%85%8Dandroid/)

[React Native进阶之原生模块封装基础篇1](http://www.lcode.org/react-native%E8%BF%9B%E9%98%B6%E4%B9%8B%E5%8E%9F%E7%94%9F%E6%A8%A1%E5%9D%97%E7%BB%84%E4%BB%B6%E5%B0%81%E8%A3%85%E5%9F%BA%E7%A1%80%E7%AF%871-%E9%80%82/)
[一个自定义的badge控件](https://github.com/bingoogolapple/react-native-bga-badge-view)



