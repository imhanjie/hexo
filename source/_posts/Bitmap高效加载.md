title: Bitmap高效加载
date: 2016-05-29 17:42:32
tags:tags: [Android,Note]
---

如何加载一个Bitmap对象？如何高效加载一个Bitmap对象？

<!--more-->

### 如何加载一个Bitmap对象？

BitmapFactory提供了四类方法：decodeFile、decodeResource、decodeStream和decodeByteArray，分别用于支持从文件系统、资源、输入流以及字节数组中加载一个Bitmap对象，其中decodeFile和decodeResource又间接调用了decodeStream方法，这四类方法最终是在Android的底层实现的，对应着BitmapFactory类的几个native方法。


### 如何高效加载一个Bitmap对象？
* 核心是通过BitmapFactory.Options来加载所需尺寸的图片，通过BitmapFactory.Options就可以很方便地对一个图片进行采样缩放

``` java
    public Bitmap decodeSampledBitmapFromResource(Resources res, int resId, int reqWidth, int reqHeight) {
        // 首先将inJustDecodeBounds置为true来检查尺寸(而不真正加载图片)
        BitmapFactory.Options options = new BitmapFactory.Options();
        options.inJustDecodeBounds = true;
        BitmapFactory.decodeResource(res, resId, options);
        options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight);
        // 将inJustDecodeBounds置为false去加载图片
        options.inJustDecodeBounds = false;
        return BitmapFactory.decodeResource(res, resId, options);
    }
```

``` java
    public Bitmap decodeSampledBitmapFromFileDescriptor(FileDescriptor fd, int reqWidth, int reqHeight) {
        BitmapFactory.Options options = new BitmapFactory.Options();
        options.inJustDecodeBounds = true;
        BitmapFactory.decodeFileDescriptor(fd, null, options);
        options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight);
        // 将inJustDecodeBounds置为false去加载图片
        options.inJustDecodeBounds = false;
        return BitmapFactory.decodeFileDescriptor(fd, null, options);
    }
```

``` java
    private int calculateInSampleSize(BitmapFactory.Options options, int reqWidth, int reqHeight) {
        if (reqWidth == 0 || reqHeight == 0) {
            return 1;
        }
        // 获取图片的原始宽高信息
        int rawWidth = options.outWidth;
        int rawHeight = options.outHeight;
        int inSampleSize = 1;
        if (rawWidth > reqWidth || rawHeight > reqHeight) {
            int halfWidth = rawWidth / 2;
            int halfHeight = rawHeight / 2;
            while ((halfWidth / inSampleSize) >= reqWidth && (halfHeight / inSampleSize) >= reqHeight) {
                // inSampleSize应为2的指数，例如inSampleSize为2，代表宽高均为原来的1/2
                inSampleSize *= 2;
            }
        }
        return inSampleSize;
    }
```

使用：
``` java
// 这里只列举了decodeSampledBitmapFromResource和decodeSampledBitmapFromFileDescriptor，其他类似
mImageView.setBitmapBitmap(decodeSampledBitmapFromResource(getResource(),R.id.img,100,100));
```



### 参考：
> 《Android开发艺术探索》