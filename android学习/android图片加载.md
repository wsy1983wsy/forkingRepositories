# android图片加载
图片加载分为第三方库加载图片，显示大图，选择照片，图片加载原理几个部分。
#第三方库加载

## Universal Image Loader（UIL）
### url
https://github.com/nostra13/Android-Universal-Image-Loader
### 使用方法
#### 统一配置
一般情况下在一个应用程序中只有一个ImageLoader的实例变量，此实例变量设置了缓存目录，优先级，内存大小，硬盘缓存大小，缓存文件名命名规则等。代码如下所示：

```javascript
public static PersistentImageLoader getImageLoader() {
        if (imageLoader == null) {
            try {
                String cachePath = Tools.getCachedFolder();
                if (cachePath == null) {
                    cachePath = getApplicationContext().getCacheDir()
                            .getAbsolutePath() + File.separator + "images";
                }
                File cacheFolder = new File(cachePath);
                if (!cacheFolder.exists()) {
                    cacheFolder.mkdirs();
                }
                imageLoaderConfig = new ImageLoaderConfiguration.Builder(
                        context)
                        .threadPriority(Thread.NORM_PRIORITY - 1) //下载优先级
                        .tasksProcessingOrder(QueueProcessingType.LIFO)//下载队列，先进后出
                        .memoryCache(new LruMemoryCache(100 * 1024 * 1024))//内存缓存方式
                        .memoryCacheSize(100 * 1024 * 1024)//内存缓存大小
                        .denyCacheImageMultipleSizesInMemory()//不在内存中缓存同一张图片的多种尺寸
                        .diskCache(new UnlimitedDiskCache(cacheFolder))//硬盘缓存方式
                        .diskCacheSize(1000000 * 1024 * 1024)//硬盘缓存尺寸
                        .diskCacheFileCount(10000000)//硬盘缓存文件个数
                        .diskCacheFileNameGenerator(new Md5FileNameGenerator())//硬盘文件命名方式
                        .build();
                ImageLoader.getInstance().init(imageLoaderConfig);
                imageLoader = PersistentImageLoader.getInstance();
            } catch (Exception e) {
                // TODO: handle exception
                e.printStackTrace();
            }
        }
        return imageLoader;
    }
```
整个系统中只使用此方法返回的ImageLoader即可。
#### 显示选项
显示选项设置了默认图片，加载失败图片，加载中图片，是否保存到硬盘，图片配置等信息,配置代码如下:

```javascript
 public static DisplayImageOptions getCorpLogoOptions() {
        if (corpLogoOptions == null) {
            corpLogoOptions = new DisplayImageOptions.Builder()
                    .showImageOnLoading(R.drawable.ic_corp)//加载中显示的图片
                    .showImageForEmptyUri(R.drawable.ic_corp)//URL为空显示的图片
                    .showImageOnFail(R.drawable.ic_corp)//加载失败显示的图片
                    .cacheInMemory(true)//是否在内存中缓存
                    .cacheOnDisk(true)//是否在硬盘中缓存
                    .considerExifParams(true)//是否考虑Exif参数
                    .displayer(new SimpleBitmapDisplayer())//显示方式
                    .imageScaleType(ImageScaleType.EXACTLY_STRETCHED)//图片拉伸模式
                    .bitmapConfig(Bitmap.Config.RGB_565)//图片配置，使用RGB_565
                    .build();
        }
        return corpLogoOptions;
    }
```
一般情况一种业务逻辑设置一个显示选项。比如头像，帖子图片各使用一种显示选项。
#### 加载图片
##### 显示网络图片

```javascript
//displayImage，第一个参数为图片的url，第二个图片为要加载图片的ImageView，可以是ImageView或其子类，第三个参数为显示选项，第四个参数为图片加载监听器
imageLoader.displayImage(thumbImageUrl, logoRoundedView, imageOptions, new ImageLoadingListener() {
     @Override
     public void onLoadingStarted(String imageUri, View view) {

      }

      @Override
      public void onLoadingFailed(String imageUri, View view, FailReason failReason) {
                    }

      @Override
      public void onLoadingComplete(String imageUri, View view, Bitmap loadedImage) {
                    
      }

      @Override
      public void onLoadingCancelled(String imageUri, View view) {

      }
    });
}
```
##### 显示本地图片
需要有图片的绝对路径。然后再路径的前面加上"file:///";显示方式与显示网络图片相同。
#### 显示圆形或其他形状图片
1. 自定义ImageView，让ImageView在onDraw方法中显示要求的形状的Image。可以使用[RoundedImageView](https://github.com/vinc3m1/RoundedImageView) , [CircleImageView](https://github.com/hdodenhof/CircleImageView) , [AvatarImageView](https://github.com/Carbs0126/AvatarImageView) , [CircleTextImageView](https://github.com/CoolThink/CircleTextImageView)
2. 使用UIL自身带的CircleBitmapDisplayer或者RoundedBitmapDisplayer，但是在使用displayImage方法时需要加载图片的必须是ImageAware，可以通过ImageAware的构造函数从ImageView转为ImageAware。在设置显示选项的时候设置显示方式为这两种Displayer中任意一个即可。

推荐使用第一种方式。因为第二种方式加载速度稍慢。

### 获取缓存的图片
有时我们需要获取已经显示过的url对应的图片，因为UIL具有缓存作用，因此不需要再从网络下载图片了，只需要从UIL缓存的文件夹中获取对应的图片即可。具体方法为通过imageLoader获取其diskCache，再根据Url获取缓存文件。<br>

```javascript
 File imageLoaderCachedFile = imageLoader.getDiskCache().get(imageUri);
//根据imageLoaderCachedFile是否为null，以及是否存在可以判断是否对应的文件是否可有，如果可用则是我们需要的文件，如果不可用再从网络下载图片。
``` 
### 优化加载，ListView滑动时暂停加载
可以设置ListView的ScrollListener为PauseOnScrollListener，具体方位为:<br>

```javscript
//第一个参数为ImageLoader，第二个参数表示是否在滚动时停止加载，第三个参数表示否在fling时停止加载，第四各参数表示额外的滚动监听器。
PauseOnScrollListener  listener = new PauseOnScrollListener(imageLoader, true, true, null)
listView.setOnScrollListener(listener);
```
### 下载图片
使用ImageLoader的loadImageSync方法下载图片，该方法返回的是一个bitmap，由于该方法是一个同步方法，需要另起线程执行。一般使用方法为:<br>

```javascript
new Thread(new Runnable() {
            @Override
            public void run() {
            //options表示显示选项，需要设置缓存在sd卡上才能在保存对应的文件，loadImageSync只返回一个Bitmap，如果需要可以手动保存。
                imageLoader.loadImageSync("url", options);
	             File imageFile = imageLoader.getDiskCache().get("url");
	             //判断文件是否缓存。
                if (imageFile != null && imageFile.exists()) {
                    Log.d("MeActivity", "imageExists");
                }
            }
        }).start();
```
> 考虑到UIL作者已经不再更新，所以在新的代码中不建议使用此库。

## glide
### url
https://github.com/bumptech/glide
### 基本的加载方法
#### 加载URL
```javascript
 ImageView targetImageView = (ImageView) findViewById(R.id.imageView);
 String imageUrl = Urls.ImageURls.get(0);
 //with方法中的是activity，fragment，Context
 Glide.with(this).load(imageUrl).into(targetImageView);
```
#### 加载资源
```javascript
 ImageView targetImageView = (ImageView) findViewById(R.id.imageView);
 Glide.with(this).load(R.mipmap.stone).into(targetImageView);
```
#### 加载文件
```javascript
File externalStorage = Environment.getExternalStorageDirectory();
//sd卡下Pictures目录下有一个图片Picture_11_Taste.jpg
String imagePath = externalStorage.getAbsolutePath() + FOREWARD_SLASH + "Pictures" + FOREWARD_SLASH + "Picture_11_Taste.jpg";
File imageFile = new File(imagePath);
ImageView targetImageView = (ImageView) findViewById(R.id.imageView);
Glide.with(this).load(imageFile).into(targetImageView);
```
#### 加载Uri
```javascript
public void onUriImageClicked(View view) {
  ImageView targetImageView = (ImageView) findViewById(R.id.imageView);
  Uri uri = resourceIdToUri(this, R.mipmap.stone);
  Glide.with(this).load(uri).into(targetImageView);
}

public static final String ANDROID_RESOURCE = "android.resource://";
public static final String FOREWARD_SLASH = "/";

private static Uri resourceIdToUri(Context context, int resourceId) {
  return Uri.parse(ANDROID_RESOURCE + context.getPackageName() + FOREWARD_SLASH + resourceId);
}
```
### 转换
#### 基础的转换
* centerCrop 会按原始比例缩小图像，使宽或者高的一边等于给定的值，另外一边会等于或者大于给定值。CenterCrop会裁剪掉多余部分。 CenterCrop和Android中的ScaleType.CENTER_CROP效果相同。
* fitCenter 会按原始比例缩小图像，使图像可以在放在给定的区域内。FitCenter会尽可能少地缩小图片，使宽或者高的一边等于给定的值。另外一边会等于或者小于给定值。 FitCenter和Android中的ScaleType.FIT_CENTER效果相同。等比缩放。

#### 自定义转换
如果需要自定义转换，需要继承自BitmapTransformation，一般的转换代码如下：<br>

```javascript
private static class MyTransformation extends BitmapTransformation {

    public MyTransformation(Context context) {
       super(context);
    }

    @Override
    protected Bitmap transform(BitmapPool pool, Bitmap toTransform,
            int outWidth, int outHeight) {
       //转换代码，进行bitmap处理
       Bitmap myTransformedBitmap = ... // apply some transformation here.
       return myTransformedBitmap;
    }

    @Override
    public String getId() {
        // Return some id that uniquely identifies your transformation.
        return "com.example.myapp.MyTransformation";
    }
}
```
这样在加载的时候就可以使用transfrom方法替换掉fitCenter，centerCrop方法了。
#### 第三方类库
使用 [glide-transformations](https://github.com/wasabeef/glide-transformations) 类库。
### 缓存
#### 内存缓存
skipMemeoryCache方法表示是否不在内存中缓存，默认是在内容中缓存。
#### 磁盘缓存
diskCacheStartegy,表示图片的磁盘缓存策略。<br>
磁盘缓存策略有以下几种:<br>

* DiskCacheStrategy.NONE 什么都不缓存，就像刚讨论的那样
* DiskCacheStrategy.SOURCE 仅仅只缓存原来的全分辨率的图像。在我们上面的例子中，将会只有一个  1000x1000 像素的图片
* DiskCacheStrategy.RESULT 仅仅缓存最终的图像，即，降低分辨率后的（或者是转换后的）
* DiskCacheStrategy.ALL 缓存所有版本的图像

#### 缓存失效
##### 缓存的key
DiskCacheStrategy.RESULT磁盘缓存策略（注：我们配置的一种磁盘缓存策略）使用的key由以下四个主要部分组成：<br>

* DataFetcher的方法getId()返回的字符。典型地，DataFetcher仅仅返回由数据Model的toString()方法得到的值。所以，如果Model是一个URL，那么会返回URL的字符串，如果Model是是一个文件，那么会返回文件的路径。
* 宽和高。如果你调用过override(width,height)方法，那么就是是它传入的值。没有调用过，默认是通过Target的getSize()方法获得这个值。
* 各种编码器、解码器的getId()方法返回的字符串。这些编码器、解码器用于加载和缓存你的图片。仅有哪些堆bytes数据有影响的编码器、解码器才会有这些id值。比如，你只有一个将bytes数据写入磁盘的编码器，那么它就没有id值，因为不管怎样它都不会修改数据。
* 可选地，你可以为图片加载提供签名(Signature)，请看下面的缓存失效部分。

所有的这些key，以特定的顺序计算出hash值，并将这个值作为保存图片到磁盘上的唯一且安全的文件名。

##### 失效原因
由于文件名是hash值，没有简便的方式删除磁盘上某个特定url或者文件路径的所有缓存文件。如果仅仅缓存原文件，问题可能还比较简单，但是Glide会缓存缩略图，各种变换后的图片，所有这些缓存都会产生新文件。跟踪和删除所有文件是十分困难的。因此可能无法准确的从缓存取取到一个已经显示过的图片的文件。

### 请求优先级
priority可以设置图片下载的优先级，优先级分为：<br>

* Priority.LOW
* Priority.NORMAL
* Priority.HIGH
* Priority.IMMEDIATE

优先级从低到高。高优先级先执行。

### 缩略图
通过指定thumbnail，参数为float，该缩略图使用的要显示图的缩略图。同样的也可以不使用要显示的图像，方法为:<br>

```javascript
 DrawableRequestBuilder<String> thumbnailRequest = Glide
        .with( context )
        .load( eatFoodyImages[2] );

    // pass the request as a a parameter to the thumbnail request
 Glide
        .with( context )
        .load( UsageExampleGifAndVideos.gifUrl )
        .thumbnail( thumbnailRequest )
        .into( imageView3 );
```
### Target
Glide提供了一个用Target获取Bitmap资源的方法。Target只是用来回调，它会在所有的加载和处理完毕时返回想要的结果。<br>
lide使用Target作为资源的接收器，以及生命周期事件的回调接口。<br>
#### SimpleTarget
使用方法为:<br>

```javascript
private SimpleTarget target = new SimpleTarget<Bitmap>() {
        @Override
        public void onResourceReady(Bitmap bitmap, GlideAnimation glideAnimation) {
            imageView.setImageBitmap(bitmap);
        }
};
```
在onResourceReady方法中返回了Bitmap，在该方法中可以讲bitmap设置到imageview中。另外可以设置图片的大小，示例代码为：<br>

```javascript
//200,200表示图片的大小
private SimpleTarget target = new SimpleTarget<Bitmap>(200, 200) {
        @Override
        public void onResourceReady(Bitmap bitmap, GlideAnimation glideAnimation) {
            imageView.setImageBitmap(bitmap);
        }

    };
```
#### ViewTarget
某些情况下，我们不能使用ImageView接收图片，可能需要使用自定义的View来接收图片，使用方法为:<br>

```javascript
ViewTarget viewTarget = new ViewTarget<CustomViewClass, GlideDrawable>( customView ) {
        @Override
        public void onResourceReady(GlideDrawable resource, GlideAnimation<? super GlideDrawable> glideAnimation) {
			Bitmap bitamp =  resource.getCurrent();
         //自定义View对bitmap的处理
        }
    };

    Glide
        .with( context.getApplicationContext() ) // safer!
        .load( eatFoodyImages[2] )
        .into( viewTarget );
```
这样自定View就可以接收Glide下载下来的图片了。
### GlideModule
#### 简介
Glide modules是一个全局改变Glide行为的抽象的方式。你需要创建Glide的实例，来访问GlideBuilder。可以通过创建一个公共的类，实现GlideModule的接口来定制Glide：<br>

```javascript
public class SimpleGlideModule implements GlideModule {  
    @Override public void applyOptions(Context context, GlideBuilder builder) {
        // todo
    }

    @Override public void registerComponents(Context context, Glide glide) {
        // todo
    }
}
```
applyOptions方法将GlideBuilder的对象当作参数，并且是void返回类型，所以你在这个方法里能调用GlideBuilder可以用的方法。<br>
可以在这个方法中调用GlideBuilder的方法对Glide进行各种配置。<br>
在AndroidManifest.xml中声明这个类，这样Glide知道它应该加载并使用它。Glide会扫描AndroidManifest.xml的Glide modules的meta定义。这样，你必须在AndroidManifest.xml里的<application>标签下声明刚创建的Glide module。<br>

```javascript
<manifest
    ...
    <application>
        <meta-data
            android:name="io.futurestud.tutorials.glide.glidemodule.SimpleGlideModule"
            android:value="GlideModule" />
        ...
    </application>
</manifest>
```
name需要填写完整的包名和类名。如果你想要禁止Glide Module，只要从AndroidManifest.xml里移除它。你可以一次同时声明多个Glide Module。Glide会（没有特殊的顺序）都遍历所有声明的module。由于你当前未定义顺序，确保你的定制不会造成冲突！
#### 自定义缓存
Glide内存使用MemorySizeCalculator类去决定内存缓存和bitmap池的大小。修改内存缓存的方法如下所示:

```javascript
public class CustomCachingGlideModule implements GlideModule {  
    @Override public void applyOptions(Context context, GlideBuilder builder) {
        MemorySizeCalculator calculator = new MemorySizeCalculator(context);
        int defaultMemoryCacheSize = calculator.getMemoryCacheSize();
        int defaultBitmapPoolSize = calculator.getBitmapPoolSize();

        int customMemoryCacheSize = (int) (1.2 * defaultMemoryCacheSize);
        int customBitmapPoolSize = (int) (1.2 * defaultBitmapPoolSize);

        builder.setMemoryCache( new LruResourceCache( customMemoryCacheSize );
        builder.setBitmapPool( new LruBitmapPool( customBitmapPoolSize );
    }

    @Override public void registerComponents(Context context, Glide glide) {
        // nothing to do here
    }
}
```
磁盘缓存可以存在app的私有目录下或者sd卡上。有两个类InternalCacheDiskCacheFactory，ExternalCacheDiskCacheFactory可以进行私有目录下和sd卡目录下得存储。<br>
一般的修改磁盘缓存的方法如下：<br>

```javascript
public class CustomCachingGlideModule implements GlideModule {  
    @Override
    public void applyOptions(Context context, GlideBuilder builder) {
        // set size & external vs. internal
        int cacheSize100MegaBytes = 104857600;

        builder.setDiskCache(
            new InternalCacheDiskCacheFactory(context, cacheSize100MegaBytes)
        );

        //builder.setDiskCache(
        //new ExternalCacheDiskCacheFactory(context, cacheSize100MegaBytes));
    }

    @Override
    public void registerComponents(Context context, Glide glide) {
        // nothing to do here
    }
}
```
上面的方法只是设置了缓冲区的大小，并不能改变缓冲区的位置。如果想改变缓冲区的位置需要使用DiskLruCacheFactory。示例代码为：<br>

```javascript
// or any other path
String downloadDirectoryPath = Environment.getDownloadCacheDirectory().getPath(); 

builder.setDiskCache(  
        new DiskLruCacheFactory( downloadDirectoryPath, cacheSize100MegaBytes )
);

// In case you want to specify a cache sub folder (i.e. "glidecache"):
//builder.setDiskCache(
//    new DiskLruCacheFactory( downloadDirectoryPath, "glidecache", cacheSize100MegaBytes ) 
//);
```
### 获取已经加载过的图片
由于glide会保存原图片和显示的图片，以及缓存失效问题，因此无法通过url获取图片的缓存文件，可以通过这种方法获取图片信息。

```javascript
private SimpleTarget bytesTarget = new SimpleTarget<byte[]>() {
        @Override
        public void onResourceReady(byte[] bytes, GlideAnimation glideAnimation) {
            Log.d("GlideCache", "" + bytes.length);
            //可以对bytes执行各种操作
        }
    };
Glide.with(this) 
                .load(Urls.eatImages.get(2))
                .asBitmap()
                .diskkCacheStrategy(DiskCacheStrategy.SOURCE )
                .toBytes()
                .into(bytesTarget);
```

### 问题以及解决方法
#### 有的图片第一次加载的时候只显示占位图，第二次才显示正常的图片呢？
1.如果你刚好使用了这个圆形Imageview库或者其他的一些自定义的圆形Imageview，而你又刚好设置了占位的话，那么，你就会遇到第一个问题。如何解决呢？<br>
方案一: 不设置占位；<br>
方案二：使用Glide的Transformation API自定义圆形Bitmap的转换。可以使用如下的代码:

```javascript
Glide.with(this).load(URL).transform(new CircleTransform(context)).into(imageView);

public static class CircleTransform extends BitmapTransformation {
    public CircleTransform(Context context) {
        super(context);
    }

    @Override protected Bitmap transform(BitmapPool pool, Bitmap toTransform, int outWidth, int outHeight) {
        return circleCrop(pool, toTransform);
    }

    private static Bitmap circleCrop(BitmapPool pool, Bitmap source) {
        if (source == null) return null;

        int size = Math.min(source.getWidth(), source.getHeight());
        int x = (source.getWidth() - size) / 2;
        int y = (source.getHeight() - size) / 2;

        // TODO this could be acquired from the pool too
        Bitmap squared = Bitmap.createBitmap(source, x, y, size, size);

        Bitmap result = pool.get(size, size, Bitmap.Config.ARGB_8888);
        if (result == null) {
            result = Bitmap.createBitmap(size, size, Bitmap.Config.ARGB_8888);
        }

        Canvas canvas = new Canvas(result);
        Paint paint = new Paint();
        paint.setShader(new BitmapShader(squared, BitmapShader.TileMode.CLAMP, BitmapShader.TileMode.CLAMP));
        paint.setAntiAlias(true);
        float r = size / 2f;
        canvas.drawCircle(r, r, r, paint);
        return result;
    }

    @Override public String getId() {
        return getClass().getName();
    }
} 
```
方案三：使用下面的代码加载图片：<br>

```javascript
Glide.with(mContext)
    .load(url) 
    .placeholder(R.drawable.loading_spinner)
    .into(new SimpleTarget<Bitmap>(width, height) {
        @Override 
        public void onResourceReady(Bitmap bitmap, GlideAnimation anim) {
            // setImageBitmap(bitmap) on CircleImageView 
        } 
    });
```
该方法在listview上复用有问题的bug,如果在listview中加载CircleImageView，请不要使用该方法。<br>
方案四：不使用Glide的默认动画

```javascript
Glide.with(mContext)
    .load(url) 
    .dontAnimate()
    .placeholder(R.drawable.loading_spinner)
    .into(circleImageview);
```
#### 我总会得到类似You cannot start a load for a destroyed activity这样的异常呢？
不要在非主线程里面使用Glide加载图片，如果真的使用了，请把context参数换成getApplicationContext。
#### 我不能给加载的图片setTag()呢？
方案一：使用setTag(int,object)方法设置tag,具体用法如下：<br>

```javascript
Glide.with(context).load(urls.get(i).getUrl()).fitCenter().into(imageViewHolder.image);
        imageViewHolder.image.setTag(R.id.image_tag, i);
        imageViewHolder.image.setOnClickListener(new View.OnClickListener() {
            @Override
                int position = (int) v.getTag(R.id.image_tag);
                Toast.makeText(context, urls.get(position).getWho(), Toast.LENGTH_SHORT).show();
            }
        });
```
同时在values文件夹下新建ids.xml，添加：<br>

```javascript
<item name="image_tag" type="id"/>
```
方案二：从Glide的3.6.0之后，新添加了全局设置的方法。具体方法如下：
先实现GlideMoudle接口，全局设置ViewTaget的tagId:<br>

```javascrip
public class MyGlideMoudle implements GlideModule{
    @Override
    public void applyOptions(Context context, GlideBuilder builder) {
        ViewTarget.setTagId(R.id.glide_tag_id);
    }

    @Override
    public void registerComponents(Context context, Glide glide) {

    }
}
```
同样，也需要在ids.xml下添加id:<br>

```javascript
<item name="R.id.glide_tag_id" type="id"/>
```
最后在AndroidManifest.xml文件里面添加

```javascript
<meta-data
    android:name="com.yourpackagename.MyGlideMoudle"
    android:value="GlideModule" />
```
方案三：写一个继承自ImageViewTaget的类，复写它的get/setRequest方法。<br>

```javascript
Glide.with(context).load(urls.get(i).getUrl()).fitCenter().into(new ImageViewTarget<GlideDrawable>(imageViewHolder.image) {
            @Override
            protected void setResource(GlideDrawable resource) {
                imageViewHolder.image.setImageDrawable(resource);
            }

            @Override
            public void setRequest(Request request) {
                imageViewHolder.image.setTag(i);
                imageViewHolder.image.setTag(R.id.glide_tag_id,request);
            }

            @Override
            public Request getRequest() {
                return (Request) imageViewHolder.image.getTag(R.id.glide_tag_id);
            }
        });

        imageViewHolder.image.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                int position = (int) v.getTag();
                Toast.makeText(context, urls.get(position).getWho(), Toast.LENGTH_SHORT).show();
            }
        });
```
### 技巧
1. 当列表在滑动的时候，调用Glide.with(context).pauseRequests()取消请求，滑动停止时，调用Glide.with(context).resumeRequests()恢复请求。
2. 当你想清除掉所有的图片加载请求时调用Glide.clear()。
3. 让列表预加载使用ListPreloader类。

### ListPreLoader
ListPreLoader表示列表预先加载的数据。使用方法如下：<br>

```javasccript
private ListPreloader listPreloader = new ListPreloader<String>(new ListPreloader.PreloadModelProvider<String>() {
        @Override
        public List<String> getPreloadItems(int position) {
            List<String> preloads = new ArrayList<>(1);
            preloads.add(Urls.ImageURls.get(position));
            return preloads;
        }

        @Override
        public GenericRequestBuilder getPreloadRequestBuilder(String item) {
            return Glide.with(GlideListView.this).load(item);
        }
    }, new ListPreloader.PreloadSizeProvider<String>() {
        @Override
        public int[] getPreloadSize(String item, int adapterPosition, int perItemPosition) {
            int[] size = {200, 200};
            return size;
        }
    }, 3);


listview = (ListView) findViewById(R.id.listview);
listview.setOnScrollListener(listPreloader);
```


## picasso
### url
https://github.com/square/picasso
### 使用方法
使用方法与glide完全相同，不过picasso没有动画显示。有resize方法。
### 获取缓存图片方法

```javascript
Picasso.with(this).load(imageUrl).into(new Target() {
            @Override
            public void onBitmapLoaded(Bitmap bitmap, LoadedFrom from) {
                Log.d("Picasso", "" + bitmap.getWidth());
                //在此处可以对bitmap进行操作。
            }

            @Override
            public void onBitmapFailed(Drawable errorDrawable) {

            }

            @Override
            public void onPrepareLoad(Drawable placeHolderDrawable) {

            }
        });
```
### 图片处理
Picasso同样有图片的转换类Transformation，该类对要显示的图片进行了处理。使用方法同glide。或者使用[picasso-transformations](https://github.com/wasabeef/picasso-transformations)

### 显示优化
#### 图片裁剪
在列表页尽量使用裁剪后的图片，在查看大图模式下才加载完整的图片。<br>

```javascript
	Picasso.with( imageView.getContext() )
	.load(url)
	.resize(dp2px(250),dp2px(250))
	.centerCrop()
	.into(imageView);
```
picasso默认情况下会使用全局的ApplicationContext，即开发者传进去Activity，picasso也会通过activity获取ApplicationContext。

#### 查看大图放弃memory cache
Picasso默认会使用设备的15%的内存作为内存图片缓存，且现有的api无法清空内存缓存。我们可以在查看大图时放弃使用内存缓存，图片从网络下载完成后会缓存到磁盘中，加载会从磁盘中加载，这样可以加速内存的回收。<br>

```javascript
	Picasso.with(getApplication())
	.load(mURL)
	.memoryPolicy(NO_CACHE, NO_STORE)
	.into(imageView);
```
其中memoryPolicy的NO_CACHE是指图片加载时放弃在内存缓存中查找，NO_STORE是指图片加载完不缓存在内存中。
#### RecyclableImageView
重写ImageView的onDetachedFromWindow方法，在它从屏幕中消失时回调，去掉drawable引用，能加快内存的回收。<br>

```javascript
public class RecyclerImageView extends ImageView
{ 
    ...

    @Override    
    protected void onDetachedFromWindow() {
        super.onDetachedFromWindow();
        setImageDrawable(null);   
    }
}
```
#### 新进程中查看大图
列表页的内存已经非常稳定，但是查看大图时，大图往往占用了20+m内存，加上现有进程中的内存，非常容易oom，在新进程中打开Activity成为比较取巧的避免oom的方式。<br>

```javascript
<activity android:name=".DetailActivity" android:process=":picture"/>
```
只要在AndroidManifest.xml中定义Activity时加入process属性，即可在新进程中打开此Activity。由此，picasso也将在新进程中创建基于新ApplicationContext的单例。

#### 列表页滑动优化
picasso可以对多个加载请求设置相同的tag，即

```
	Object tag = new Object();
	Picasso.with( imageView.getContext() )
	.load(url)
	.resize(dp2px(250),dp2px(250))
	.centerCrop()
	.tag(tag)
	.into(imageView);
```
例如在RecyclerView滑动时监听，处理不同的表现：<br>

```javascript
mRecyclerView.addOnScrollListener(new RecyclerView.OnScrollListener(){
    @Override
    public void onScrollStateChanged(RecyclerView recyclerView, int newState)
    {
        if (newState == RecyclerView.SCROLL_STATE_IDLE)
        {
            Picasso.with(context).resumeTag(tag);
        }
        else
        {
            Picasso.with(context).pauseTag(tag);
        }
    }
});
```
 
#### RGB_565
对于不透明的图片可以使用RGB_565来优化内存。<br>

```javascript
	Picasso.with( imageView.getContext() )
	.load(url)
	.config(Bitmap.Config.RGB_565)
	.into(imageView);
```
默认情况下，Android使用ARGB_8888

Android中有四种，分别是：

* ALPHA_8：每个像素占用1byte内存
* ARGB_4444:每个像素占用2byte内存
* ARGB_8888:每个像素占用4byte内存
* RGB_565:每个像素占用2byte内存

## fresco
### url
https://github.com/facebook/fresco
### 使用方法

