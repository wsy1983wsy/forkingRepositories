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
### 使用方法

## fresco
### url
https://github.com/facebook/fresco
### 使用方法

## picasso
### url
https://github.com/square/picasso
### 使用方法

