# Bitmap
![](https://upload-images.jianshu.io/upload_images/2618044-cd996dd172cce293.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

## 配置信息与压缩方式
**Bitmap 中有两个内部枚举类：**
- Config 是用来设置颜色配置信息
- CompressFormat 是用来设置压缩方式

| Config | 单位像素所占字节数 | 解析
| ------- | ------- | ------
| Bitmap.Config.ALPHA_8 | 1 | 颜色信息只由透明度组成，占8位
| Bitmap.Config.ARGB_4444 | 2 |颜色信息由rgba四部分组成，每个部分都占4位，总共占16位
| Bitmap.Config.ARGB_8888 | 4 |颜色信息由rgba四部分组成，每个部分都占8位，总共占32位。是Bitmap默认的颜色配置信息，也是最占空间的一种配置
| Bitmap.Config.RGB_565 | 2 | 颜色信息由rgb三部分组成，R占5位，G占6位，B占5位，总共占16位
| RGBA_F16 | 8 | Android 8.0 新增（更丰富的色彩表现HDR）
| HARDWARE | Special | Android 8.0 新增 （Bitmap直接存储在graphic memory）

通常我们优化 Bitmap 时，当需要做性能优化或者防止 OOM，我们通常会使用 Bitmap.Config.RGB_565 这个配置，因为 Bitmap.Config.ALPHA_8 只有透明度，显示一般图片没有意义，Bitmap.Config.ARGB_4444 显示图片不清楚，Bitmap.Config.ARGB_8888 占用内存最多。

| CompressFormat | 解析
| ------- | -------
| Bitmap.CompressFormat.JPEG | 表示以 JPEG 压缩算法进行图像压缩，压缩后的格式可以是 ``.jpg`` 或者 ``.jpeg``，是一种有损压缩 |
| Bitmap.CompressFormat.PNG | 颜色信息由 rgba 四部分组成，每个部分都占 4 位，总共占 16 位 |
| Bitmap.Config.ARGB_8888 | 颜色信息由 rgba 四部分组成，每个部分都占 8 位，总共占 32 位。是 Bitmap 默认的颜色配置信息，也是最占空间的一种配置
| Bitmap.Config.RGB_565 | 颜色信息由 rgb 三部分组成，R 占 5 位，G 占 6 位，B 占 5 位，总共占 16 位

## 常用操作
### 裁剪、缩放、旋转、移动
```java
Matrix matrix = new Matrix();  
// 缩放
matrix.postScale(0.8f, 0.9f);  
// 左旋，参数为正则向右旋
matrix.postRotate(-45);  
// 平移, 在上一次修改的基础上进行再次修改 set 每次操作都是最新的 会覆盖上次的操作
matrix.postTranslate(100, 80);
// 裁剪并执行以上操作
Bitmap bitmap = Bitmap.createBitmap(source, 0, 0, source.getWidth(), source.getHeight(), matrix, true);
````
虽然Matrix还可以调用postSkew方法进行倾斜操作，但是却不可以在此时创建Bitmap时使用。

### Bitmap与Drawable转换
```java
// Drawable -> Bitmap
public static Bitmap drawableToBitmap(Drawable drawable) {
    Bitmap bitmap = Bitmap.createBitmap(drawable.getIntrinsicWidth(), drawable.getIntrinsicHeight(), drawable.getOpacity() != PixelFormat.OPAQUE ? Bitmap.Config.ARGB_8888 : Bitmap.Config.RGB_565);
    Canvas canvas = new Canvas(bitmap);
    drawable.setBounds(0, 0, drawable.getIntrinsicWidth(), drawable.getIntrinsicHeight();
    drawable.draw(canvas);
    return bitmap;
}

// Bitmap -> Drawable
public static Drawable bitmapToDrawable(Resources resources, Bitmap bm) {
    Drawable drawable = new BitmapDrawable(resources, bm);
    return drawable;
}
```

### 保存与释放
```java
Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.drawable.test);
File file = new File(getFilesDir(),"test.jpg");
if(file.exists()){
    file.delete();
}
try {
    FileOutputStream outputStream=new FileOutputStream(file);
    bitmap.compress(Bitmap.CompressFormat.JPEG,90,outputStream);
    outputStream.flush();
    outputStream.close();
} catch (FileNotFoundException e) {
    e.printStackTrace();
} catch (IOException e) {
    e.printStackTrace();
}
//释放bitmap的资源，这是一个不可逆转的操作
bitmap.recycle();
```

### 图片压缩

```java
public static Bitmap compressImage(Bitmap image) {
    if (image == null) {
        return null;
    }
    ByteArrayOutputStream baos = null;
    try {
        baos = new ByteArrayOutputStream();
        image.compress(Bitmap.CompressFormat.JPEG, 100, baos);
        byte[] bytes = baos.toByteArray();
        ByteArrayInputStream isBm = new ByteArrayInputStream(bytes);
        Bitmap bitmap = BitmapFactory.decodeStream(isBm);
        return bitmap;
    } catch (OutOfMemoryError e) {
        e.printStackTrace();
    } finally {
        try {
            if (baos != null) {
                baos.close();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    return null;
}
```

## BitmapFactory
### Bitmap创建流程
![](https://upload-images.jianshu.io/upload_images/2618044-9c2046ca5054da05.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)


### Option类
| 常用方法 | 说明
|-----|------
| boolean inJustDecodeBounds | 如果设置为true，不获取图片，不分配内存，但会返回图片的高度宽度信息
| int inSampleSize | 图片缩放的倍数
| int outWidth | 获取图片的宽度值
| int outHeight | 获取图片的高度值
| int inDensity | 用于位图的像素压缩比
| int inTargetDensity | 用于目标位图的像素压缩比（要生成的位图）
| byte[] inTempStorage | 创建临时文件，将图片存储
| boolean inScaled | 设置为true时进行图片压缩，从inDensity到inTargetDensity
| boolean inDither | 如果为true,解码器尝试抖动解码
| Bitmap.Config inPreferredConfig | 设置解码器这个值是设置色彩模式，默认值是ARGB_8888，在这个模式下，一个像素点占用4bytes空间，一般对透明度不做要求的话，一般采用RGB_565模式，这个模式下一个像素点占用2bytes
| String outMimeType | 设置解码图像
| boolean inPurgeable | 当存储Pixel的内存空间在系统内存不足时是否可以被回收
| boolean inInputShareable | inPurgeable为true情况下才生效，是否可以共享一个InputStream
| boolean inPreferQualityOverSpeed | 为true则优先保证Bitmap质量其次是解码速度
| boolean inMutable | 配置Bitmap是否可以更改，比如：在Bitmap上隔几个像素加一条线段
| int inScreenDensity | 当前屏幕的像素密度

### 基本使用
```java
try {
    FileInputStream fis = new FileInputStream(filePath);
    BitmapFactory.Options options = new BitmapFactory.Options();
    options.inJustDecodeBounds = true;
    // 设置inJustDecodeBounds为true后，再使用decodeFile()等方法，并不会真正的分配空间，即解码出来的Bitmap为null，但是可计算出原始图片的宽度和高度，即options.outWidth和options.outHeight
    BitmapFactory.decodeFileDescriptor(fis.getFD(), null, options);
    float srcWidth = options.outWidth;
    float srcHeight = options.outHeight;
    int inSampleSize = 1;

    if (srcHeight > height || srcWidth > width) {
        if (srcWidth > srcHeight) {
            inSampleSize = Math.round(srcHeight / height);
        } else {
            inSampleSize = Math.round(srcWidth / width);
        }
    }

    options.inJustDecodeBounds = false;
    options.inSampleSize = inSampleSize;

    return BitmapFactory.decodeFileDescriptor(fis.getFD(), null, options);
} catch (Exception e) {
    e.printStackTrace();
}
```

## 内存回收
```java
if(bitmap != null && !bitmap.isRecycled()){
    // 回收并且置为null
    bitmap.recycle();
    bitmap = null;
}
```

Bitmap 类的构造方法都是私有的，所以开发者不能直接 new 出一个 Bitmap 对象，只能通过 BitmapFactory 类的各种静态方法来实例化一个 Bitmap。仔细查看 BitmapFactory 的源代码可以看到，生成 Bitmap 对象最终都是通过 JNI 调用方式实现的。所以，加载 Bitmap 到内存里以后，是包含两部分内存区域的。简单的说，一部分是Java 部分的，一部分是 C 部分的。这个 Bitmap 对象是由 Java 部分分配的，不用的时候系统就会自动回收了，但是那个对应的 C 可用的内存区域，虚拟机是不能直接回收的，这个只能调用底层的功能释放。所以需要调用 recycle() 方法来释放 C 部分的内存。从 Bitmap 类的源代码也可以看到，recycle() 方法里也的确是调用了 JNI 方法了的。
