# Android内存优化分析
Android手持设备的资源是非常有限的，相对于桌面平台软件开发，在Android平台上开发app应该更有效地管理自己的资源。内存使用是否优化将直接影响到app使用的手感，内存优化得不好会直接造成操作不流畅，卡顿。我之前虽然也在Google Play发布过一些小应用，但是对于内存使用的优化并没有注意，我觉得深入地去探讨这个问题会更有利于我开发水平的提高。

---------
## OutOfMemoryError
Android是一个多任务的平台，所以为每个app设置了一个最大的堆大小，我们写代码的时候，申请的对象内存会被分配在这个堆上，当堆大小达到最大限制，而我们再尝试分配内存，就会得到OutOfMemoryError。最大堆大小针对不同的设备也有不同的值，可以用如下代码获取当前设备的最大堆大小：

    ActivityManager mManager = (ActivityManager) getSystemService(ACTIVITY_SERVICE);
    int mTotalSize = mManager.getMemoryClass();

这里我在Galaxy Nexus上测得这个值为64MB，在Nexus 7上得到的值是192MB，所以当我们的程序需要占用大量内存时，必须时刻注意是否可能超过这个值，并作出相应的优化措施。下面我会根据我的理解和体会阐述一些优化的方案。

--------
## Bitmap造成的OOM
很多OOM出现的地方都是大量使用了Bitmap的地方，图片要显示出来就需要全部加载进内存，而图片本身大小还是相当可观的，所以如果所有图片都按照原样载入内存那肯定会出问题的，在阅读了官方关于内存和Bitmap处理的文档之后，我总结了如下几个方面，我觉得应该可以很大程度上减少Bitmap造成的OOM问题。

---------

### 加载Bitmap造成OOM
先考虑一次加载一张图片的情况，一般使用Gallery控件时，如果没有自己在Adapter中使用缓存，那么都是一次加载一张图片，如果加载的图片非常大，那么当加载到这一张的时候，会直接Force close，抛出OOM。原因是加载这一张图片就达到堆的大小限制了，我确实见过一张就大小几十M的图片，虽然这种情况是少数，但是还是需要处理。加载大图的一个办法就是根据需要适当降低图片质量，下面是一般的降低图片质量的步骤。 

1. 在加载图片之前，首先需要读取出需要加载图片的类型和大小，这样可以为后续的加载和缩放获得必要的参数，当然不是加载进内存来读这些信息，这样就没意义了，官方文档中有一段代码解释了如何完成这个步骤

        BitmapFactory.Options options = new BitmapFactory.Options();
        options.inJustDecodeBounds = true;
        BitmapFactory.decodeResource(getResources(), R.id.myimage, options);
        int imageHeight = options.outHeight;
        int imageWidth = options.outWidth;
        String imageType = options.outMimeType;
以上代码里，inJustDecodeBounds参数被设置为true，这样就使得decodeResource方法并不会进行内存分配，但是options参数却被填充了图片的信息。

2.  根据实际情况确定如何缩放图片，这个需要根据图片使用的位置来确定，举一个最常用的例子，我们加载图片一般会放到ImageView里面，ImageView的大小通常不会很大，要加载的原图肯定远远大于ImageView的大小，如果原分辨率地加载进去肯定不划算，这个时候只要加载一个跟ImageView大小差不多的图片，视觉效果上是一样的。要实现这个功能，就需要设置解码的时候用的BitmapFactory.Options的inSampleSize字段，这个字段如果设置为大于1, 那么解码的时候会分配更少的内存获取一个缩略版的图片。官方文档中给出了一个计算inSampleSize的方法，代码如下：

        public static int calculateInSampleSize(
            BitmapFactory.Options options, int reqWidth, int reqHeight) {
            final int height = options.outHeight;
            final int width = options.outWidth;
            int inSampleSize = 1;
            if (height > reqHeight || width > reqWidth) {
                final int halfHeight = height / 2;
                final int halfWidth = width / 2;
                while ((halfHeight / inSampleSize) > reqHeight
                        && (halfWidth / inSampleSize) > reqWidth) {
                    inSampleSize *= 2;
                }
            }
            return inSampleSize;
        }
这里我们只需要传入在步骤一中获得的options对象和ImageView的宽度和高度，这样这个方法就会返回一个能让缩放后图片略大于ImageView的宽度和高度的最大inSampleSize值，这样我们就在基本不损失视觉效果的情况下减少了加载图片的内存占用。我们现在可以用如下的代码来获取适合一个ImageView的Bitmap：

        public static Bitmap decodeSampledBitmapFromResource(Resources res, int resId,
                int reqWidth, int reqHeight) {
        
            // 这里第一次不分配内存，只是为了获取原图的信息
            final BitmapFactory.Options options = new BitmapFactory.Options();
            options.inJustDecodeBounds = true;
            BitmapFactory.decodeResource(res, resId, options);
        
            // 计算合适的inSampleSize
            options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight);
        
            // 根据合适的inSampleSize产生低内存消耗的图片
            options.inJustDecodeBounds = false;
            return BitmapFactory.decodeResource(res, resId, options);
        }
        
### 缓存Bitmap造成OOM
Bitmap加载是比较费时的操作，无论是从本地存储还是从网络加载，所以如果每次加载都需要从外部加载，就会造成界面的卡顿，一个比较好的办法是缓存Bitmap，也就是预先把Bitmap加载到内存中，这就涉及到速度和内存占用的权衡问题。尽可能多地缓存图片自然会让app有更好地流畅行，但是可能造成OOM。
通常缓存图片的途径就是保存一个对Bitmap的强引用，这样就可以让它驻留在内存中，比如我们可以用一个HashMap来做这件事情，但是这种做法就需要我们自己手动管理内存，因为HashMap中存的是强引用，如果不手动从HashMap中移除引用，这块内存永远不会被GC回收，这种做法就没有利用好Java的GC。
我本来想了一个办法，使用WeakReference来保存已经加载的Bitmap的引用，这样即达到了缓存的效果，又能让GC在内存不够的时候自动回收内存。但是查阅了一下官方文档，找到了如下的说明：
>Note: In the past, a popular memory cache implementation was a SoftReference or WeakReference bitmap cache, however this is not recommended. Starting from Android 2.3 (API Level 9) the garbage collector is more aggressive with collecting soft/weak references which makes them fairly ineffective. In addition, prior to Android 3.0 (API Level 11), the backing data of a bitmap was stored in native memory which is not released in a predictable manner, potentially causing an application to briefly exceed its memory limits and crash.

看来Android平台上的GC表现的和预期还是有很大差异的，WeakReference对于缓存几乎没有作用。我想大概是因为Android本身运行在资源紧张的嵌入式设备上，所以对于WeakReference指向的对象，GC也会尽可能地去回收。
操作系统课程中学到过LRU替换算法，思想就是用最近的过去来近似代替最近的未来，安卓里面缓存图片的一个好办法也是用LruCache，要求是API 12以上的版本，也就是Android 3.1以上，现在大部分设备都是4.x的版本，这种缓存方法应该比较具有通用性。我这里写了一个非常简单的类，代码如下：

    public class CacheManager {
        /**
         * LRU cache，用于缓存图片
         */
        private LruCache<String, Bitmap> mLruCache;
    
        /**
         * 构造器
         *
         * @param size 缓存大小，单位是K
         */
        public CacheManager(int size) {
            mLruCache = new LruCache<String, Bitmap>(size);
        }
    
        /**
         * 将Bitmap添加到LRU缓存
         * @param key 用于保持引用的key
         * @param bitmap 实际需要引用的对象
         */
        public void add(String key, Bitmap bitmap) {
            if(get(key) == null) {
                mLruCache.put(key, bitmap);
            }
        }
    
        /**
         * 从LRU缓存获取Bitmap
         * @param key
         * @return 如果Bitmap已经从缓存中去除，这个方法将会返回null
         */
        public Bitmap get(String key) {
            return mLruCache.get(key);
        }
    }
这样管理内存的工作就交给了LruCache这个类，对于我们来说，缓存的管理是透明的，但是却在效率和内存占用上达到了一个平衡。LRU缓存的大小设置得当，基本可以避免OOM问题。
这里再考虑一种比较常见的情况，在使用Gallery时缓存图片，在这种地方使用LRU缓存效果不会很好，根据自己的使用经验，在看相册的时候，用户滑动通常只局限在3张的范围内，即当前这张，前一张和后一张，所以缓存只需要缓存这三张就行。

---------
## 异步任务造成的OOM
AsyncTask是为了简化Android后台线程与前台的交互逻辑，我们很多时候启动了一个AsyncTask就不去管了，因为AsyncTask往往执行的也不是很耗时的操作，请求个API什么的基本都很快会返回。但是如果有这样一种情况，在执行AsyncTask的时候，发起请求的Activity已经结束，那么AsyncTask会结束吗？不会的，AsyncTask除非正常返回，执行了onPostExecute方法或者是调用了cancel方法才会结束，如果存在大量我们以为已经结束的AsyncTask，那么就会造成OOM。下面举个例子，如果我们按照这样写：

    class BitmapWorkerTask extends AsyncTask<Integer, Void, Bitmap> {
        private final ImageView imageView;
        private int data = 0;
    
        public BitmapWorkerTask(ImageView imageView) {
            this.imageView = imageView;
        }
    
        @Override
        protected Bitmap doInBackground(Integer... params) {
            data = params[0];
            return decodeSampledBitmapFromResource(getResources(), data, 100, 100));
        }
    
        @Override
        protected void onPostExecute(Bitmap bitmap) {
            if (bitmap != null) {
                if (imageView != null) {
                    imageView.setImageBitmap(bitmap);
                }
            }
        }
    }
    
这样就会造成AsyncTask一直保存一个对ImageView的引用，导致潜在的内存泄漏。在这个地方应该用弱引用来防止AsyncTask可能造成的无法回收内存。如果按照下面这样写就可以避免AsyncTask出现OOM：

    class BitmapWorkerTask extends AsyncTask<Integer, Void, Bitmap> {
        private final WeakReference<ImageView> imageViewReference;
        private int data = 0;
    
        public BitmapWorkerTask(ImageView imageView) {
            // 弱引用，Android 3.0之后的GC已经几乎无视这种引用，不影响内存回收
            imageViewReference = new WeakReference<ImageView>(imageView);
        }
    
        @Override
        protected Bitmap doInBackground(Integer... params) {
            data = params[0];
            return decodeSampledBitmapFromResource(getResources(), data, 100, 100));
        }
    
        @Override
        protected void onPostExecute(Bitmap bitmap) {
            if (imageViewReference != null && bitmap != null) {
                final ImageView imageView = imageViewReference.get();
                // 因为这个地方不能保证ImageView还在内存中，所以需要检测
                if (imageView != null) {
                    imageView.setImageBitmap(bitmap);
                }
            }
        }
    }
同时需要注意的是，如果把AsyncTask作为Activity的内部类，AsyncTask会隐式地保存一个对Activity的引用。
