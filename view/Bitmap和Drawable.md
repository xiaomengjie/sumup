#### Bitmap

位图信息，保存每个像素的颜色

#### Drawable

使用canvas绘制的上层工具，专注于绘制。可以说保存的是绘制规则

在调用draw绘制时，需要先调用setBounds设置绘制范围

#### Bitmap2Drawable

直接创建一个BitmapDrawable

```kotlin
//直接new BitmapDrawable，传入bitmap
public inline fun Bitmap.toDrawable(
    resources: Resources
): BitmapDrawable = BitmapDrawable(resources, this)
```

#### Drawable2Bitmap

如果是BitmapDrawable，直接通过getBitmap获得bitmap返回。

如果不是，创建Bitmap和canvas，使用drawable通过canvas把内容绘制到bitmap上

```kotlin
public fun Drawable.toBitmap(
    @Px width: Int = intrinsicWidth,
    @Px height: Int = intrinsicHeight,
    config: Config? = null
): Bitmap {
  	//是bitmapDrawable
    if (this is BitmapDrawable) {
        if (bitmap == null) {
            // This is slightly better than returning an empty, zero-size bitmap.
            throw IllegalArgumentException("bitmap is null")
        }
        if (config == null || bitmap.config == config) {
            //要求的宽高没变，直接返回
            if (width == bitmap.width && height == bitmap.height) {
                return bitmap
            }
          	//宽高有变化，创建新的bitmap按照比例缩放后返回
            return Bitmap.createScaledBitmap(bitmap, width, height, true)
        }
    }
		
  	//保存drawable的返回
    val (oldLeft, oldTop, oldRight, oldBottom) = bounds

  	//创建bitmap
    val bitmap = Bitmap.createBitmap(width, height, config ?: Config.ARGB_8888)
  	//设置绘制范围
    setBounds(0, 0, width, height)
  	//内容绘制到bitmap上
    draw(Canvas(bitmap))

  	//drawable范围恢复
    setBounds(oldLeft, oldTop, oldRight, oldBottom)
    return bitmap
}
```
