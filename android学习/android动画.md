# 动画分类
android动画分为View动画(tweened animation，又叫补间动画)，帧动画(frame-by-frame animation)，属性动画。逐帧动画的工作原理很简单，其实就是将一个完整的动画拆分成一张张单独的图片，然后再将它们连贯起来进行播放，类似于动画片的工作原理，这样容易造成内存溢出(OOM)。补间动画则是在一定时间内对View进行一系列的动画操作，包括淡入淡出、缩放、平移、旋转四种，而且只能做这四种动画操作。属性动画在android 3.0之后引入，指在一定时间内通过动态地改变对象的属性从而达到动画效果。<br>
## 为何引入属性动画
补间动画只能对View做淡入淡出、缩放、平移、旋转四种四种动画，而且逐帧动画容易造成OOM，这些功能不足以满足所有的场景（比如改变View的背景色），也就是View动画有很大的局限性。而且补间动画有一个缺陷，只改变了View的显示效果，而不会真正的改变View的属性。也就是说现在屏幕的左上角有一个按钮，然后我们通过补间动画将它移动到了屏幕的右下角，现在你可以去尝试点击一下这个按钮，点击事件是绝对不会触发的，因为实际上这个按钮还是停留在屏幕的左上角，只不过补间动画将这个按钮绘制到了屏幕的右下角而已。<br>
新引入的属性动画不再只针对View，不仅现定于实现缩放，移动，旋转，淡入淡出这几种动画了，也不再是一种视觉上的动画效果了。实际上是一种不断的对值进行操作的机制，并将值赋值到指定对象的指定属性上，可以是任意对象的任意属性。所以我们仍然可以讲一个View进行移动和缩放，同事也可以对自定义View中得Point对象进行动画操作了，我们只需要告诉系统动画的运行时长，需要执行哪些类型的动画，以及动画的初始值和结束值，剩下的工作可以全部交给系统去完成了。由于属性动画是对目标对象的属性进行赋值并修改其属性来实现的，因此上面的按钮显示问题也就不复存在了，这样按钮就是真正的移动，而不再是仅仅在另外一个位置绘制了。
# 补间动画
可以在一个视图容器内执行一系列简单转换（位置，大小，旋转，透明度）。通过xml和代码定义，建议使用XML文件定义，XML文件更具有可读性，可重用性。类名和xml标签关系如下:

|java类|标签|描述|
|-----|----|----|
|AlaphaAnimation|<alpha>放置在res/anim目录下|透明度动画|
|RotateAnimation|<rotate>放置在res/anim目录下|旋转动画|
|ScacleAnimation|<scale>放置在res/anim目录下|尺寸缩放动画|
|TranslateAnimation|<translate>放置在res/anim目录下|位置动画|
|AnimationSet|<set>放置在res/anim目录下|一个持有其它动画元素alpha、scale、translate、rotate或者其它set元素的容器|

## 视图动画
Animation类是所有补间动画的基类，具有以下属性：<br>

|xml属性|	java方法|	解释|
|------|----------|-------|
|android:detachWallpaper|	setDetachWallpaper(boolean)|	是否在壁纸上运行|
|android:duration	|setDuration(long)|	动画持续时间，毫秒为单位|
|android:fillAfter	|setFillAfter(boolean)	|控件动画结束时是否保持动画最后的状态|
|android:fillBefore	|setFillBefore(boolean)|	控件动画结束时是否还原到开始动画前的状态|
|android:fillEnabled	|setFillEnabled(boolean)|	与android:fillBefore效果相同|
|android:interpolator|setInterpolator(Interpolator)|	设定插值器（指定的动画效果，譬如回弹等）|
|android:repeatCount	|setRepeatCount(int)|	重复次数|
|android:repeatMode	|setRepeatMode(int)|	重复类型有两个值，reverse表示倒序回放，restart表示从头播放|
|android:startOffset	|setStartOffset(long)|	调用start函数之后等待开始运行的时间，单位为毫秒|
|android:zAdjustment|	setZAdjustment(int)|	表示被设置动画的内容运行时在Z轴上的位置（top/bottom/normal），默认为normal|
无论补间动画属于哪一种都具备这些属性，都可以使用这些属性中的一个或多个。
## alpha动画属性

|xml属性	|java方法	|解释|
|-------|----------|----|
|android:fromAlpha	|AlphaAnimation(float fromAlpha, …)|	动画开始的透明度（0.0到1.0，0.0是全透明，1.0是不透明）|
|android:toAlpha	|AlphaAnimation(…, float toAlpha)|	动画结束的透明度，同上|

## Rotate属性
|xml属性	|java方法	|解释|
|--------|---------|-----|
|android:fromDegrees	|RotateAnimation(float fromDegrees, …)|	旋转开始角度，正代表顺时针度数，负代表逆时针度数|
|android:toDegrees|	RotateAnimation(…, float toDegrees, …)|	旋转结束角度，正代表顺时针度数，负代表逆时针度数|
|android:pivotX	|RotateAnimation(…, float pivotX, …)|	旋转中心点X坐标（数值、百分数、百分数p，譬如50表示以当前View左上角坐标加50px为中心点、50%表示以当前当前View宽的50%做为中心点、50%p表示以父控件宽的50%做为中心点）|
|android:pivotY	|RotateAnimation(…, float pivotY)|	缩放起点Y坐标，同上规律|
## Translate属性
|xml属性|	java方法|	解释|
|------|-----------|------|
|android:fromXDelta	|TranslateAnimation(float fromXDelta, …)|	起始点X轴坐标（数值、百分数、百分数p，譬如50表示以当前View左上角坐标加50px为初始点、50%表示以当前View宽的50%做为初始点、50%p表示以父控件宽的50%做为初始点）|
|android:fromYDelta	|TranslateAnimation(…, float fromYDelta, …)|	起始点Y轴从标，同上规律|
|android:toXDelta	|TranslateAnimation(…, float toXDelta, …)|	结束点X轴坐标，同上规律|
|android:toYDelta	|TranslateAnimation(…, float toYDelta)|	结束点Y轴坐标，同上规律|

## AnimationSet
AnimationSet继承自Animation，是上面四种的组合容器管理类，没有自己特有的属性，他的属性继承自Animation，所以特别注意，当我们对set标签使用Animation的属性时会对该标签下的所有子控件都产生影响。具体使用方法如下：<br>

```javascript
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android"
    android:interpolator="@[package:]anim/interpolator_resource"
    android:shareInterpolator=["true" | "false"] >
    <alpha
        android:fromAlpha="float"
        android:toAlpha="float" />
    <scale
        android:fromXScale="float"
        android:toXScale="float"
        android:fromYScale="float"
        android:toYScale="float"
        android:pivotX="float"
        android:pivotY="float" />
    <translate
        android:fromXDelta="float"
        android:toXDelta="float"
        android:fromYDelta="float"
        android:toYDelta="float" />
    <rotate
        android:fromDegrees="float"
        android:toDegrees="float"
        android:pivotX="float"
        android:pivotY="float" />
    <set>
        ...
    </set>
</set>
```
## 使用方法

```javascript
ImageView spaceshipImage = (ImageView) findViewById(R.id.spaceshipImage);
Animation hyperspaceJumpAnimation = AnimationUtils.loadAnimation(this, R.anim.hyperspace_jump);
spaceshipImage.startAnimation(hyperspaceJumpAnimation);
```
Animation具有如下的方法:<br>

|Animation类的方法	|解释|
|------------------|----|
|reset()	|重置Animation的初始化|
|cancel()	|取消Animation动画|
|start()	|开始Animation动画|
|setAnimationListener(AnimationListener listener)|	给当前Animation设置动画监听|
|hasStarted()	|判断当前Animation是否开始|
|hasEnded()	|判断当前Animation是否结束|

View具有StartAnimation和clearAnimation两个方法用于启动动画和清除动画。

# 帧动画
帧动画同样可以使用java和XML方式实现，XML更简单。但是xml应当放在res/drawable目录下，具体如下:<br>

<animation-list> 必须是根节点，包含一个或者多个<item>元素，属性有：<br>

* android:oneshot true代表只执行一次，false循环执行。
* <item> 类似一帧的动画资源。

<item> animation-list的子项，包含属性如下：<br>

* android:drawable 一个frame的Drawable资源。
* android:duration 一个frame显示多长时间。

xml如下所示：<br>

```javascript
<!-- 注意：rocket.xml文件位于res/drawable/目录下 -->
<?xml version="1.0" encoding="utf-8"?>
<animation-list xmlns:android="http://schemas.android.com/apk/res/android"
    android:oneshot=["true" | "false"] >
    <item
        android:drawable="@[package:]drawable/drawable_resource_name"
        android:duration="integer" />
</animation-list>
android:onShot表示是否动画只执行一次，还是执行多次。true一次，false多次。
```
调用的代码如下：<br>

```javascript
ImageView rocketImage = (ImageView) findViewById(R.id.rocket_image);
rocketImage.setBackgroundResource(R.drawable.rocket_thrust);

rocketAnimation = (AnimationDrawable) rocketImage.getBackground();
rocketAnimation.start();
```
> 补间动画，和帧动画考虑到android版本升级换代很快，可以考虑不再使用。
> 可以使用

# 属性动画
## ValueAnimator
ValueAnimator是整个属性动画机制中最核心的一个类，属性动画的运行机制是通过不断的对值进行操作来实现的，而初始值和结束值之间的动画过度就是由ValueAnimator这个类来负责计算的。内部使用一种时间循环的机制来计算与值之间的动画过渡，只需要将初始值和结束值提供给ValueAnimator，并且告诉它动画所需运动的时长，那么ValueAnimator就会自动帮我们完成从初始值平滑过渡到结束值这样的效果。ValueAnimator还负责管理动画的播放次数，播放模式，以及对动画设置监听器等。<br>
使用方法为：<br>

```javascript
ValueAnimator anim = ValueAnimator.ofFloat(0f, 1f);  
anim.setDuration(300);  
anim.start();  
```
上面的代码只是将值从0过渡到1，看不到任何效果。可以通过如下的代码来确定动画已经真正在运行了。<br>

```javascript
ValueAnimator anim = ValueAnimator.ofFloat(0f, 1f);  
anim.setDuration(300);  
anim.addUpdateListener(new ValueAnimator.AnimatorUpdateListener(){
	@override
	public void onAnimationUpdate(ValueAnimator animation){
		float currentValue = (float) animation.getAnimatedValue();
		Log.d("Animation","current value is " + currentValue);	}
});
anim.start();
```
这样就可以通过anim.addUpdateListener添加一个动画的监听器，在动画执行的过程中不断进行回调，知道动画真正在运行。<br>
当然ofFloat可以接受多个值，表示在这些值的范围内进行变化。如果需要的值为整型，可以使用ofInt方法。
可以调用setStartDelay方法设置动画延迟播放，setRepeatCount方法设置动画循环播放的次数，setRepeatMode设置循环播放的模式，循环模式包括RESTART和REVERSE两种，分别表示重新播放和倒序播放。

## ObjectAnimator
相比于ValueAnimator，ObjectAnimator可能是我们最常接触到的类，因为ValueAnimator只不过是对值进行了一个平滑的动画过渡，但我们实际使用到这种功能的场景好像并不多。而ObjectAnimator则就不同了，它是可以直接对任意对象的任意属性进行动画操作的，比如说View的alpha属性。但是ObjectAnimator继承自ValueAnimator，地城的动画机制也是基于ValueAnimator来完成的。比如想将一个TextView在5秒中内从常规变换成全透明，再从全透明变换成常规，就可以这样写：<br>

```javascript
//第一个参数为要操作的Object，第二个参数为Object的属性，后面的参数不固定，想要完成什么样的动画就传入什么值。
ObjectAnimator animator = ObjectAnimator.ofFloat(textView,"alpha", 1f,0f,1f);
animator.setDuration(5000);
animator.start();
```
第二个参数的要求，如果要对对象的xxx属性执行动画，必须满足如下的条件:<br>

* object必须有setXxx方法，如果动画的时候没有传递初始值，那么还要提供getXxx方法，因为系统要获取xxx的初始值（如果不满足，程序会直接crash）
*  object的setXxx属性所做的改变必须通过某种方法反映出来，比如会带来UI的变化之类的（如果不满足，动画无效但不会crash）

如果对象只能满足第一个条件，但是第二个条件无法满足，可以使用以下方法解决。<br>

* 如果有权限，给对象加上get和set方法
* 用一个类来包装原始对象，间接为其提供get和set方法，使set方法实现动画效果。
* 采用ValueAnimator，监听动画的过程，自己实现属性的改变，实现动画效果。

假设要通过动画来改变一个button的宽度，动画代码如下:<br>

```javascript
 ObjectAnimator.ofInt(mButton, "width", 500).setDuration(5000).start();  
```
对于第一种方法一般情况下无法实现。而setWidth不能修改button的宽度。 <br>
第二种方法，使用一个类包装原始对象，代码如下:<br>

```javascript
private static class ViewWrapper {  
    private View mTarget;  
  
    public ViewWrapper(View target) {  
        mTarget = target;  
    }  
  
    public int getWidth() {  
        return mTarget.getLayoutParams().width;  
    }  
  
    public void setWidth(int width) { 
    //修改View的layoutParams进行修改view的宽度 
        mTarget.getLayoutParams().width = width;  
        mTarget.requestLayout();  
    }  
}
ViewWrapper wrapper = new ViewWrapper(mButton);  
ObjectAnimator.ofInt(wrapper, "width", 500).setDuration(5000).start();
```
第三种方法，采用ValueAnimator，监听动画过程，自己实现属性的改变:<br>

```javascript
ValueAnimator valueAnimator = ValueAnimator.ofInt(1, 100);  
  
    valueAnimator.addUpdateListener(new AnimatorUpdateListener() {  
  
        //持有一个IntEvaluator对象，方便下面估值的时候使用  
        private IntEvaluator mEvaluator = new IntEvaluator();  
  
        @Override  
        public void onAnimationUpdate(ValueAnimator animator) {  
            //获得当前动画的进度值，整型，1-100之间  
            int currentValue = (Integer)animator.getAnimatedValue();  
            Log.d(TAG, "current value: " + currentValue);  
            //计算当前进度占整个动画过程的比例，浮点型，0-1之间
            float fraction = currentValue / 100f;   
            target.getLayoutParams().width = mEvaluator.evaluate(fraction, start, end); 
            //修改targe在parent中的布局 
            target.requestLayout();  
        }  
    });  
  
    valueAnimator.setDuration(5000).start();  
```

## 组合动画
独立的动画能够实现的视觉效果毕竟相当有限，因此需要多个动画组合在一起。实现组合动画主要借助AnimatorSet类，该类提供了play方法。如果我们想这个方法传入一个Animator对象（ValueAnimator或ObjectAnimator）将会返回一个AnimatorSet.Builder对象，AnimatorSet.Builder包含以下四个方法:<br>

* after(Animator anim) 将现有动画插入到传入的动画之后执行
* after(long delay) 将现有动画延迟指定毫秒后执行
* before(Animator anim) 将现有动画插入到传入的动画之前执行
* with(Animator anim) 将现有动画和传入的动画同时执行

下面的代码：<br>

```javascript
 	 ObjectAnimator moveIn = ObjectAnimator.ofFloat(tv,"translationX", -500, 0f);
     ObjectAnimator rotate = ObjectAnimator.ofFloat(tv, "rotation", 0, 360);
     ObjectAnimator fadeInOut = ObjectAnimator.ofFloat(tv, "alpha", 1f, 0f,1f);
     AnimatorSet animatorSet = new AnimatorSet();
     animatorSet.play(rotate).with(fadeInOut).after(moveIn);
     animatorSet.setDuration(3000);
     animatorSet.start();
```
这个动画让TextView先从屏幕外移动进屏幕，然后开始旋转360度，旋转的同时进行淡入淡出操作。

## 监听器
很多时候，我们希望可以监听到动画的各种事件，比如动画什么时候开始，结束，然后再开始和结束的时候去执行一些处理逻辑。Animator类当中提供了一个addListener()方法，这个方法接收一个AnimatorListener，我们只需要去实现这个AnimatorListener就可以监听动画的各种事件了。ObjectAnimator是继承自ValueAnimator的，而ValueAnimator又是继承自Animator的。因此ObjectAnimator和ValueAnimator都可有addListener方法。下面的代码表示了如何使用监听器：<br>

```javascript
ObjectAnimator moveIn = ObjectAnimator.ofFloat(tv,"translationX", -500, 0f);
        moveIn.addListener(new AnimatorListener() {
            @Override
            public void onAnimationStart(Animator animation) {

            }

            @Override
            public void onAnimationEnd(Animator animation) {

            }

            @Override
            public void onAnimationCancel(Animator animation) {

            }

            @Override
            public void onAnimationRepeat(Animator animation) {

            }
        });
```
把上面的方法都实现一遍很麻烦，而且有时候又不需要都监听，可以使用适配器类进行监听AnimatorListenerAdapter，该类实现了AnimatorListener和AnimatorPauseListener接口，只需要实现任何自己想要的方法即可。

## ValueAnimator 进阶使用方法
有一个自定义的View，在这个View当中有一个Point对象用于管理坐标，然后在onDraw()方法当中就是根据这个Point对象的坐标值来进行绘制的。也就是说，如果我们可以对Point对象进行动画操作，那么整个自定义View的动画效果就有了。首先需要了解TypeEvaluator，TypeEvaluator就是告诉动画系统如何从初始值过渡到结束值。ValueAnimator的ofFloat方法就是实现了初始值到结束值之间的平滑过渡，如实实现的呢？就是系统内置了一个FloatEvaluator，它通过计算告知动画系统如何从初始值过渡到结束值。其实现代码为：<br>

```javascript
public class FloatEvaluator implements TypeEvaluator<Number> {
    public Float evaluate(float fraction, Number startValue, Number endValue) {
        float startFloat = startValue.floatValue();
        return startFloat + fraction * (endValue.floatValue() - startFloat);
    }
}
```
FloatEvaluator实现了TypeEvaluator接口，然后重写evaluate()方法。evaluate()方法当中传入了三个参数，第一个参数fraction非常重要，这个参数用于表示动画的完成度的，我们应该根据它来计算当前动画的值应该是多少，第二第三个参数分别表示动画的初始值和结束值。那么上述代码的逻辑就比较清晰了，用结束值减去初始值，算出它们之间的差值，然后乘以fraction这个系数，再加上初始值，那么就得到当前动画的值了。ValueAnimator还有一个ofObject方法，是用于对任意对象进行动画操作，但是相比于浮点型或整型数据，对象的动画操作明显更复杂一些，因为系统完全无法知道如何从初始对象过渡到结束对象，因此需要实现一个自己的TypeValuator来告知系统如何进行过渡。定义一个Point类，如下所示：<br>

```javascript
public class Point {
    private float x;
    private float y;

	  public Point(float x, float y){
			this.x = x;
			this.y = y;
	 }
	 
    public float getX() {
        return x;
    }

    public void setX(float x) {
        this.x = x;
    }

    public float getY() {
        return y;
    }

    public void setY(float y) {
        this.y = y;
    }
}
```
定义Point对应的TypeEvaluator类，PointEvaluator:<br>

```javascript
public class PointEvaluator implements TypeEvaluator{
	@override
	public Ojbect evaluate(float fraction, Ojbect startValue, Object endValue){
		Point startPoint = (Point)startValue;
		point endPoint = (Point)endValue;
		float x = startPoint.getX() + fraction*(endPoint.getX() - startPoint.getX());
		float y = startPoint.getY() + fraction*(endPoint.getY() - startPoint.getY());
		return new Point(x,y);
	}
}
```
可以看到，PointEvaluator同样实现了TypeEvaluator接口并重写了evaluate()方法。其实evaluate()方法中的逻辑还是非常简单的，先是将startValue和endValue强转成Point对象，然后同样根据fraction来计算当前动画的x和y的值，最后组装到一个新的Point对象当中并返回。PointEvaluator的使用方法：<br>

```javascript
Point point1 = new Point(0,0);
Point point2 = new Point(300,300);
valueAnimator anim = ValueAnimator.ofObject(new PointEvaluator(), point1, point2);
anim.setDuration(5000);
anim.start();
```
自定义View根据Point变化位置：<br>

```javascript
public class MyAnimView extends View {  
  	//半径
    public static final float RADIUS = 50f;  
  //当前点位置
    private Point currentPoint;  
  
    private Paint mPaint;  
  
    public MyAnimView(Context context, AttributeSet attrs) {  
        super(context, attrs);  
        mPaint = new Paint(Paint.ANTI_ALIAS_FLAG);  
        mPaint.setColor(Color.BLUE);  
    }  
  
    @Override  
    protected void onDraw(Canvas canvas) { 
    //如果当前点为null，则设置当前点在半径处，并开始动画。 
        if (currentPoint == null) {  
            currentPoint = new Point(RADIUS, RADIUS);  
            drawCircle(canvas);  
            startAnimation();  
        } else {  
            drawCircle(canvas);  
        }  
    }  
  
    private void drawCircle(Canvas canvas) {  
        float x = currentPoint.getX();  
        float y = currentPoint.getY();  
        canvas.drawCircle(x, y, RADIUS, mPaint);  
    }  
  
    private void startAnimation() {  
        Point startPoint = new Point(RADIUS, RADIUS); 
        //终点为右下角 ，getWidth获取到的是当前View的宽度，getHeight获取到的时当前View的高度。
        Point endPoint = new Point(getWidth() - RADIUS, getHeight() - RADIUS);  
        ValueAnimator anim = ValueAnimator.ofObject(new PointEvaluator(), startPoint, endPoint);  
        anim.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {  
            @Override  
            public void onAnimationUpdate(ValueAnimator animation) {  
            // 更新右下角并重绘
                currentPoint = (Point) animation.getAnimatedValue();  
                invalidate();  
            }  
        });  
        anim.setDuration(5000);  
        anim.start();  
    }  
}
```

MyAnimView的使用方法：<br>

```javascript
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"  
    android:layout_width="match_parent"  
    android:layout_height="match_parent"  
    >  
	<!--设置该View铺满整个屏幕，而圆只是View的一部分-->
    <com.example.tony.myapplication.MyAnimView  
        android:layout_width="match_parent"  
        android:layout_height="match_parent" />  
  
</RelativeLayout> 
```

## ObjectAnimator的进阶用法
ObjectAnimator内部的工作机制是通过寻找特定属性的get和set方法，然后通过方法不断地对值进行改变，从而实现动画效果的。因此我们就需要在MyAnimView中定义一个color属性，并提供它的get和set方法。这里我们可以将color属性设置为字符串类型，使用#RRGGBB这种格式来表示颜色值，代码如下所示：<br>

```javascript
public class MyAnimView extends View {  
  
    ...  
  
    private String color;  
  
    public String getColor() {  
        return color;  
    }  
  		//设置Color时使View重绘。这样就相当于执行了动画
    public void setColor(String color) {  
        this.color = color;  
        mPaint.setColor(Color.parseColor(color));  
        invalidate();  
    }  
    ...  
} 
```

注意在setColor()方法当中，我们编写了一个非常简单的逻辑，就是将画笔的颜色设置成方法参数传入的颜色，然后调用了invalidate()方法。这段代码虽然只有三行，但是却执行了一个非常核心的功能，就是在改变了画笔颜色之后立即刷新视图，然后onDraw()方法就会调用。在onDraw()方法当中会根据当前画笔的颜色来进行绘制，这样颜色也就会动态进行改变了。<br>
那么接下来的问题就是怎样让setColor()方法得到调用了，毫无疑问，当然是要借助ObjectAnimator类，但是在使用ObjectAnimator之前我们还要完成一个非常重要的工作，就是编写一个用于告知系统如何进行颜色过度的TypeEvaluator。创建ColorEvaluator并实现TypeEvaluator接口，代码如下所示：<br>

```javascript
public class ColorEvaluator implements TypeEvaluator {  
  
    private int mCurrentRed = -1;  
  
    private int mCurrentGreen = -1;  
  
    private int mCurrentBlue = -1;  
  
    @Override  
    public Object evaluate(float fraction, Object startValue, Object endValue) {  
        String startColor = (String) startValue;  
        String endColor = (String) endValue;  
        int startRed = Integer.parseInt(startColor.substring(1, 3), 16);  
        int startGreen = Integer.parseInt(startColor.substring(3, 5), 16);  
        int startBlue = Integer.parseInt(startColor.substring(5, 7), 16);  
        int endRed = Integer.parseInt(endColor.substring(1, 3), 16);  
        int endGreen = Integer.parseInt(endColor.substring(3, 5), 16);  
        int endBlue = Integer.parseInt(endColor.substring(5, 7), 16);  
        // 初始化颜色的值  
        if (mCurrentRed == -1) {  
            mCurrentRed = startRed;  
        }  
        if (mCurrentGreen == -1) {  
            mCurrentGreen = startGreen;  
        }  
        if (mCurrentBlue == -1) {  
            mCurrentBlue = startBlue;  
        }  
        // 计算初始颜色和结束颜色之间的差值  
        int redDiff = Math.abs(startRed - endRed);  
        int greenDiff = Math.abs(startGreen - endGreen);  
        int blueDiff = Math.abs(startBlue - endBlue);  
        int colorDiff = redDiff + greenDiff + blueDiff;  
        if (mCurrentRed != endRed) {  
            mCurrentRed = getCurrentColor(startRed, endRed, colorDiff, 0,  
                    fraction);  
        } else if (mCurrentGreen != endGreen) {  
            mCurrentGreen = getCurrentColor(startGreen, endGreen, colorDiff,  
                    redDiff, fraction);  
        } else if (mCurrentBlue != endBlue) {  
            mCurrentBlue = getCurrentColor(startBlue, endBlue, colorDiff,  
                    redDiff + greenDiff, fraction);  
        }  
        // 将计算出的当前颜色的值组装返回  
        String currentColor = "#" + getHexString(mCurrentRed)  
                + getHexString(mCurrentGreen) + getHexString(mCurrentBlue);  
        return currentColor;  
    }  
  
    /** 
     * 根据fraction值来计算当前的颜色。 
     */  
    private int getCurrentColor(int startColor, int endColor, int colorDiff,  
            int offset, float fraction) {  
        int currentColor;  
        if (startColor > endColor) {  
            currentColor = (int) (startColor - (fraction * colorDiff - offset));  
            if (currentColor < endColor) {  
                currentColor = endColor;  
            }  
        } else {  
            currentColor = (int) (startColor + (fraction * colorDiff - offset));  
            if (currentColor > endColor) {  
                currentColor = endColor;  
            }  
        }  
        return currentColor;  
    }  
      
    /** 
     * 将10进制颜色值转换成16进制。 
     */  
    private String getHexString(int value) {  
        String hexString = Integer.toHexString(value);  
        if (hexString.length() == 1) {  
            hexString = "0" + hexString;  
        }  
        return hexString;  
    }  
}  
```
这样setColor在动画时就可以被执行到了。修改MyAnimView执行颜色变化:<br>

```javascript
public class MyAnimView extends View {  
  
    ...  
  
    private void startAnimation() {  
        Point startPoint = new Point(RADIUS, RADIUS);  
        Point endPoint = new Point(getWidth() - RADIUS, getHeight() - RADIUS);  
        ValueAnimator anim = ValueAnimator.ofObject(new PointEvaluator(), startPoint, endPoint);  
        anim.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {  
            @Override  
            public void onAnimationUpdate(ValueAnimator animation) {  
                currentPoint = (Point) animation.getAnimatedValue();  
                invalidate();  
            }  
        });  
        ObjectAnimator anim2 = ObjectAnimator.ofObject(this, "color", new ColorEvaluator(),   
                "#0000FF", "#FF0000");  
        AnimatorSet animSet = new AnimatorSet();  
        animSet.play(anim).with(anim2);  
        animSet.setDuration(5000);  
        animSet.start();  
    }  
}
```

## Interpolator的用法
Interpolator叫做差值器，主要作用是可以控制动画的变化速率，比如实现一种非线性的动画效果。不过Interpolator并不是属性动画中新增的技术，实际上从Android 1.0版本开始就一直存在Interpolator接口了，而之前的补间动画当然也是支持这个功能的。只不过在属性动画中新增了一个TimeInterpolator接口，这个接口是用于兼容之前的Interpolator的，这使得所有过去的Interpolator实现类都可以直接拿过来放到属性动画当中使用。使用属性动画时，系统默认的Interpolator其实就是一个先加速后减速的Interpolator，对应的实现类就是AccelerateDecelerateInterpolator。通过Animator的setInterpolator()方法可以替换差值器。<br>
自定义Interpolator，通过实现TimeInterpolator接口实现:<br>

```javascript
public interface TimeInterpolator {    
    float getInterpolation(float input);  
}
```
getInterpolation()方法中接收一个input参数，这个参数的值会随着动画的运行而不断变化，不过它的变化是非常有规律的，就是根据设定的动画时长匀速增加，变化范围是0到1。也就是说当动画一开始的时候input的值是0，到动画结束的时候input的值是1，而中间的值则是随着动画运行的时长在0到1之间变化的。<br>
getInterpolation的input的值决定了TypeEvluator的fraction的值。input的值是由系统经过计算后传入到getInterpolation()方法中的，然后我们可以自己实现getInterpolation()方法中的算法，根据input的值来计算出一个返回值，而这个返回值就是fraction了。编写自定义Interpolator最主要的难度都是在于数学计算，通过input计算出fraction。

## ViewPropertyAnimator
ViewPropertyAnimator是android 3.1系统附带的新功能。我们知道属性动画机制已经不是再针对View进行的了，而是一种不断的对值进行操作的机制，它可以讲值赋值到指定对象的执行属性上。但是大多说情况下，大家还是对View进行动画操作的。ViewPoropertyAnimator提供了更更加易懂，更加面向对象的API，如下所示：<br>

```javascript
textView.animate().alpha(0f);
```
animate()方法就是在Android 3.1系统上新增的一个方法，这个方法的返回值是一个ViewPropertyAnimator对象，也就是说拿到这个对象之后我们就可以调用它的各种方法来实现动画效果了，这里我们调用了alpha()方法并转入0，表示将当前的textview变成透明状态。比起使用ObjectAnimator，ViewPropertyAnimator的用法明显更加简单易懂吧。除此之外，ViewPropertyAnimator还可以很轻松地将多个动画组合到一起，比如我们想要让textview运动到500,500这个坐标点上，就可以这样写：<br>

```javascript
textview.animate().x(500).y(500); 
```

ViewPropertyAnimator是支持连缀用法的，我们想让textview移动到横坐标500这个位置上时调用了x(500)这个方法，然后让textview移动到纵坐标500这个位置上时调用了y(500)这个方法，将所有想要组合的动画通过这种连缀的方式拼接起来，这样全部动画就都会一起被执行。
如何设置动画时长：<br>

```javascript
textview.animate().x(500).y(500).setDuration(5000);  
```
设置Interpolator：<br>

```javascript
textview.animate().x(500).y(500).setDuration(5000)  
        .setInterpolator(new BounceInterpolator());  
```
关于ViewPropertyAnimator有几个细节需要注意：<br>

* 整个ViewPropertyAnimator的功能都是建立在View类新增的animate()方法之上的，这个方法会创建并返回一个ViewPropertyAnimator的实例，之后的调用的所有方法，设置的所有属性都是通过这个实例完成的。
* 在使用ViewPropertyAnimator时，我们自始至终没有调用过start()方法，这是因为新的接口中使用了隐式启动动画的功能，只要我们将动画定义完成之后，动画就会自动启动。并且这个机制对于组合动画也同样有效，只要我们不断地连缀新的方法，那么动画就不会立刻执行，等到所有在ViewPropertyAnimator上设置的方法都执行完毕后，动画就会自动启动。当然如果不想使用这一默认机制的话，我们也可以显式地调用start()方法来启动动画。
* ViewPropertyAnimator的所有接口都是使用连缀的语法来设计的，每个方法的返回值都是它自身的实例，因此调用完一个方法之后可以直接连缀调用它的另一个方法，这样把所有的功能都串接起来，我们甚至可以仅通过一行代码就完成任意复杂度的动画功能。

## XML编写动画
通过XML来编写动画可能会比通过代码来编写动画要慢一些，但是在重用方面将会变得非常轻松。<br>
如果想使用XML编写动画，首先要在res目录下面建一个animator文件夹，所有的属性动画的XML文件都应该存放在这个文件夹当中，然后XML文件中我们一共可以使用如下三种标签:<br>

* <animator>对应ValueAnimator
* <objectAnimator>对应ObjectAnimator
* <set>对应AnimatorSet

实现从0到100的平滑过渡动画，xml如下：<br>

```javascript
<animator xmlns:android="http://schemas.android.com/apk/res/android"  
    android:valueFrom="0"  
    android:valueTo="100"  
    android:valueType="intType"/>  
```
将视图的alpha属性从1到0，xml如下:<br>

```javascript
<objectAnimator xmlns:android="http://schemas.android.com/apk/res/android"  
    android:valueFrom="1"  
    android:valueTo="0"  
    android:valueType="floatType"  
    android:propertyName="alpha"/>  
```

set使用方法，先做一个transactionX，然后做一个同时执行的操作，旋转和alpha同时变化，alpha先从1到0，然后从0到1。xml如下:<br>

```javascript
<set xmlns:android="http://schemas.android.com/apk/res/android"  
    android:ordering="sequentially" > 
    <objectAnimator  
        android:duration="2000"  
        android:propertyName="translationX"  
        android:valueFrom="-500"  
        android:valueTo="0"  
        android:valueType="floatType" >  
    </objectAnimator>  
  
    <set android:ordering="together" >  
        <objectAnimator  
            android:duration="3000"  
            android:propertyName="rotation"  
            android:valueFrom="0"  
            android:valueTo="360"  
            android:valueType="floatType" >  
        </objectAnimator>  
  
        <set android:ordering="sequentially" >  
            <objectAnimator  
                android:duration="1500"  
                android:propertyName="alpha"  
                android:valueFrom="1"  
                android:valueTo="0"  
                android:valueType="floatType" >  
            </objectAnimator>  
            <objectAnimator  
                android:duration="1500"  
                android:propertyName="alpha"  
                android:valueFrom="0"  
                android:valueTo="1"  
                android:valueType="floatType" >  
            </objectAnimator>  
        </set>  
    </set>
</set>  
```

调用xml的方法为：<br>

```javascript
Animator animator = AnimatorInflater.loadAnimator(context, R.animator.anim_file);  
animator.setTarget(view);  
animator.start(); 
```
调用AnimatorInflater的loadAnimator来将XML动画文件加载进来，然后再调用setTarget()方法将这个动画设置到某一个对象上面，最后再调用start()方法启动动画就可以了。

# LayoutAnimation
LayoutAnimation主要作用于ViewGroup，表示其子View的入场方式，多用于ListView。以ListView为例，设置item的动画。<br>
定义item入场的动画:<br>

```javascript
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android"
     android:duration="300"
     android:interpolator="@android:anim/accelerate_interpolator"
     android:shareInterpolator="true">

    <alpha
        android:fromAlpha="0.0"
        android:toAlpha="1.0"/>
    <translate
        android:fromXDelta="500"
        android:toXDelta="0" />
</set>
```
定义LayoutAnimation:<br>

```javascript
<?xml version="1.0" encoding="utf-8"?>
<layoutAnimation
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:animation="@anim/anim_item"
    android:animationOrder="normal"
    android:delay="0.5">
</layoutAnimation>
```
其中animation表示需要的入场动画。animationOrder表示动画的顺序，norma，依次显示，reverse逆序显示，random随机显示。delay开始动画的延迟时间。<br>
给ListView指定LayoutAnimation：<br>

```javascript
    <ListView
        android:id="@+id/listview"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layoutAnimation="@anim/layout_anim"/>
```
也可以实用java代码指定：<br>

```javascript
  // listview item 动画
        Animation animation = AnimationUtils.loadAnimation(this, R.anim.anim_item);
        LayoutAnimationController controller = new LayoutAnimationController(animation);
        listView.setLayoutAnimation(controller);
```

# Activiyt切换
Activity自定义切换动画，主要用到了overridePendingTransition(enterAnimation,exitAnimation_这个方法，该方法必须在startActivity和finish之后调用才能生效。

```javascript
// activity 跳转动画
Intent intent = new Intent(MainActivity.this, SecondActivity.class);
startActivity(intent);
// 必须在startActivity之后调用
overridePendingTransition(R.anim.enter_anim, R.anim.exit_anim);

// 退出
public void finish() {
    super.finish();
    overridePendingTransition(R.anim.enter_anim, R.anim.exit_anim);
}
```

































