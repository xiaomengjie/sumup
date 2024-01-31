```java
import android.annotation.SuppressLint
import android.content.Context
import android.util.AttributeSet
import android.widget.TextView

/**
 * 生效优先级：
 *
 * 1、xml中直接设置（android:textColor）
 * 2、style中引入设置（style="@style/style"）
 * 3、设置defStyleAttr（R.attr.custom_defStyleAttr，在theme中用style初始化）
 * 4、设置defStyleRes（R.style.custom_defStyleRes）
 * 5、theme主体中设置（）
 *
 *     <style name="Base.Theme.BasicFourConstruction" parent="Theme.Material3.DayNight.NoActionBar">
 *         <item name="custom_defStyleAttr">@style/custom_defStyleAttr</item>
 *         <item name="android:textColor">#ffff00</item>
 *     </style>
 */

@SuppressLint("AppCompatCustomView")
open class BaseTextView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0,
    defStyleRes: Int = 0
) : TextView(context, attrs, defStyleAttr, defStyleRes)

class DefStyleAttrTextView(context: Context, attrs: AttributeSet?) :
    BaseTextView(context, attrs, R.attr.custom_defStyleAttr, R.style.custom_defStyleRes)

class DefStyleResTextView(context: Context, attrs: AttributeSet?) :
    BaseTextView(context, attrs, 0, R.style.custom_defStyleRes)

class ThemeTextView(context: Context, attrs: AttributeSet?) :
    BaseTextView(context, attrs, 0, 0)
```

