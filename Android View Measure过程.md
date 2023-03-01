# Android 自定义View

View展示需要经过Measure(测量)、Layout(摆放)、Draw(绘制)三个过程，其中：

> 1、Measure:测量并确定View的宽、高
>  2、Layout:结合Measure确定View的摆放位置
>  3、Draw:将内容绘制到Layout确定的区域



可以看出，Measure、Layout、Draw 三者是有内在联系的，通过这三步即可将View展示出来。

```java
public class MyView extends View {
    public MyView(Context context) {
        super(context);
    }

    public MyView(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
    }

    public MyView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

    @Override
    protected void onDraw(Canvas canvas) {
        //View 绘制红色
        canvas.drawColor(Color.RED);
    }
}
```

自定义View名为MyView，仅仅简单重写了构造方法与onDraw(xx)。

引用自定义view

```xml
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:background="#000000"
    android:layout_gravity="center_vertical"
    android:clickable="true"
    android:id="@+id/myviewgroup"
    android:layout_width="match_parent"
    android:layout_height="100px"
    tools:context=".MainActivity">

    <com.fish.myapplication.MyView
        android:id="@+id/mYView"
        android:background="@color/colorGreen"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content">
    </com.fish.myapplication.MyView>
</FrameLayout>
```

将MyView放置在其父布局FrameLayout里，其父布局背景为黑色，父布局宽为充满屏幕、高为100px
 我们知道layout_with、layout_height目的是告诉父布局，自己想要多大的空间，其取值可以有三种：



> 1、设置具体的值，如layout_width=50px--自己就想要50px
>  2、设置layout_width=wrap_content--不确定自己想要多大，根据自己内容的大小来确定
>  3、设置layout_width=match_parent--不确定自己想要多大，父布局多大自己就多大

这三种写法实际效果有什么区别呢，来看看实际运行效果：
 **1、设置具体的值：**
 layout_width=50px
 layout_height=50px

![image-20230228094635796](C:\Users\chenjianyu6\AppData\Roaming\Typora\typora-user-images\image-20230228094635796.png)

**2、设置包裹内容**
layout_width=wrap_content
layout_height=wrap_content

![image-20230228094707044](C:\Users\chenjianyu6\AppData\Roaming\Typora\typora-user-images\image-20230228094707044.png)

**3、设置充满父布局**
layout_width=match_parent
layout_height=match_parent

![image-20230228094737541](C:\Users\chenjianyu6\AppData\Roaming\Typora\typora-user-images\image-20230228094737541.png)

可以看出，效果与“设置包裹内容”一致。
 问题来了，在xml里分别设置这三种方式，系统是如何解析的？为什么"wrap_content"与"match_parent"效果一致？这些问题我们将在View Measure里找到答案。



## View Measure过程

子布局向父布局声明了自己想要的尺寸，父布局经过一系列考虑后将子布局能够使用的尺寸结果封装在 int 类型的参数里，最终通过measure(xx)->onMeasure(xx)传递给子布局。 子布局收到这个结果后，决定是否接受这个结果，子布局的决定反过来会影响父布局的决定。
 你可能已经发现了，MyView里没有看到接收父布局传递的结果，实际上View已经实现了默认的onMeasure(xx)方法。
 接下来通过该方法来分析：父布局如何将结果封装以及子布局拿到结果后如何处理的过程。

## View onMeasure(xx)

我们知道在View.onMeasure(xx)里可以获取父布局给的测量结果。

```csharp
#View.java
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        //父布局传入宽、高约束
        //通过比较最小的尺寸与父布局传入的尺寸，找出合适的尺寸
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }

    public static int getDefaultSize(int size, int measureSpec) {
        //size 为默认大小
        int result = size;
        //获取父布局传入的测量模式
        int specMode = MeasureSpec.getMode(measureSpec);
        //获取父布局传入的测量尺寸
        int specSize = MeasureSpec.getSize(measureSpec);

        //根据测量模式选择不同的测量尺寸
        switch (specMode) {
            case MeasureSpec.UNSPECIFIED:
                //父布局不对子布局施加任何约束，使用默认尺寸
                result = size;
                break;
            case MeasureSpec.AT_MOST:
            case MeasureSpec.EXACTLY:
                //使用父布局给的尺寸
                result = specSize;
                break;
        }
        //返回子布局确定后的尺寸
        return result;
    }

    protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
        ...
        setMeasuredDimensionRaw(measuredWidth, measuredHeight);
    }

    private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
        //记录测量后的宽、高
        mMeasuredWidth = measuredWidth;
        mMeasuredHeight = measuredHeight;
        //标记该View已经测量过
        mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
    }
```

## MeasureSpec

上边引入了一个新的类型：MeasureSpec。
 MeasureSpec 里定义了几个常量和方法，其作用其实是个工具类。之前说过父布局将测量结果封装在 int 类型的参数里传递给子布局，这个封装过程就是通过MeasureSpec完成的，子布局收到结果后，需要将封装后的参数解封，这个过程也是通过MeasureSpec完成。接下来看看其具体使用。

```java
    public static class MeasureSpec {
        //掩码
        private static final int MODE_SHIFT = 30;
        //左移30位
        private static final int MODE_MASK  = 0x3 << MODE_SHIFT;
        
        //定义注解
        @IntDef({UNSPECIFIED, EXACTLY, AT_MOST})
        @Retention(RetentionPolicy.SOURCE)
        public @interface MeasureSpecMode {}
        
        //不约束子布局尺寸，子布局想要多大就多大
        public static final int UNSPECIFIED = 0 << MODE_SHIFT;
        //左移30位，高两位=1 其余位=0
        //给子布局指定确切的尺寸
        public static final int EXACTLY     = 1 << MODE_SHIFT;
        //左移30位，高两位=2 其余位=0
        //子布局最大尺寸是父布局给的尺寸，只要不超过该值，随意
        public static final int AT_MOST     = 2 << MODE_SHIFT;
        
        //封装参数到int里
        public static int makeMeasureSpec(int size, int mode) {
            if (sUseBrokenMakeMeasureSpec) {
                return size + mode;
            } else {
                //将mode 存储在高2位
                //将size 存储剩余的30位
                //最终构成 int 数值存储
                return (size & ~MODE_MASK) | (mode & MODE_MASK);
            }
        }
        
        //从int里解封mode
        @MeasureSpecMode
        public static int getMode(int measureSpec) {
            //noinspection ResourceType
            return (measureSpec & MODE_MASK);
        }

        //从int里解封size
        public static int getSize(int measureSpec) {
            return (measureSpec & ~MODE_MASK);
        }
    }
```

> - 可以看出MeasureSpec 提供了封装测量结果与解封测量结果的方法，测量结果保存在 int 类型值里，高2位存储模式，剩余的低30位存储具体尺寸值。
> - 子布局在onMeasure(xx)对传入的参数进行解封，将宽、高尺寸值提取出来分别存放在成员变量mMeasuredWidth、mMeasuredHeight中。

![img](https:////upload-images.jianshu.io/upload_images/19073098-71ce46efb987d084.png?imageMogr2/auto-orient/strip|imageView2/2/w/834/format/webp)



关于测量模式：

> - UNSPECIFIED-->该模式不约束子布局尺寸，这个很少用
> - EXACTLY-->该模式给子布局指定确切的尺寸值，这个确切的值通过MeasureSpec.getSize(int measureSpec) 获取
> - AT_MOST-->给子布局指定尺寸的上限值，这个上限值通过MeasureSpec.getSize(int measureSpec) 获取

再回顾一下View. getDefaultSize(int size, int measureSpec)方法：
 当父布局给的测量模式是AT_MOST或者EXACTLY时，取的尺寸值是一样的，都是从MeasureSpec.getSize(int measureSpec)获取的。

在上面的Demo里，当我们分别设置layout_width=wrap_content、layout_width=match_parent时，父布局传递过来的模式分别为:AT_MOST、EXACTLY，传递过来的尺寸值都是父布局的宽度。因此，上面运行的后的效果MyView的宽度是一致的，这也就回答了我们之前的问题：
 **为什么"wrap_content"与"match_parent"效果一致？**
 当然这种效果并不符合我们的预期，知道了问题的原因，修改起来就比较容易了。

## 重写View onMeasure(xx)

```java
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        //100为宽度默认值
        int measureWidth = getMyViewSize(100, widthMeasureSpec);
        //200为高度默认值
        int measureHeight = getMyViewSize(200, heightMeasureSpec);
        //记录子子布局确认后的尺寸
        setMeasuredDimension(measureWidth, measureHeight);
    }
    
    private int getMyViewSize(int defaultSize, int measureSpec) {
        int result = defaultSize;
        //解封模式和尺寸值
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        switch (specMode) {
            case MeasureSpec.UNSPECIFIED:
                result = defaultSize;
                break;
            case MeasureSpec.AT_MOST:
                //指定一个值
                result = defaultSize;
                break;
            case MeasureSpec.EXACTLY:
                result = specSize;
                break;
        }
        return result;
    }
```

只是修改了当测量模式为AT_MOST时给其一个默认的值，在实际运用过程中，这个尺寸值时可能动态改变的。如TextView，当收到父布局的测量模式是AT_MOST时，会计算TextView里的文字内容实际占用尺寸，将这个尺寸作为TextView的测量值。又比如ImageView，当收到父布局的测量模式是AT_MOST时，会计算ImageView 关联的Drawable尺寸，将这个尺寸作为ImageView的测量值。当然TextView、ImageView实际测量过程中会考虑Padding等因素的影响，具体可参考其源码。



## 小结

以上分析了View 测量过程，老规矩，用图表示：



![img](https://upload-images.jianshu.io/upload_images/19073098-8f3d87b8f6f0a608.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

# ViewGroup Measure过程

**父布局是如何确定子布局的测量模式和测量尺寸的？**
再来看看View.onMeasure(xx)

```csharp
#View.java
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }
```

可以看出该方法是可以被子类重写的，然而ViewGroup并没有重写该方法，再来看看ViewGroup其中一个子类：FrameLayout。它重写了onMeasure(xx)方法，以此为例，分析ViewGroup Measure过程。

## FrameLayout Measure过程

```csharp
#FrameLayout.java
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        //子布局个数
        int count = getChildCount();
        ...
        //记录最大宽、高
        int maxHeight = 0;
        int maxWidth = 0;
        //记录子布局状态
        int childState = 0;

        for (int i = 0; i < count; i++) {
            //遍历直接子布局
            final View child = getChildAt(i);
            if (mMeasureAllChildren || child.getVisibility() != GONE) {
                //GONE 状态下不测量
                //该方法获取子布局的测量结果--->(1)
                measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
                //此时，子布局已经完成测量
                final FrameLayout.LayoutParams lp = (FrameLayout.LayoutParams) child.getLayoutParams();
                //将子布局的测量结果+其margin，就是父布局需要为其预留的尺寸
                maxWidth = Math.max(maxWidth,
                        child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
                maxHeight = Math.max(maxHeight,
                        child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
                //计算state---->(2)
                childState = combineMeasuredStates(childState, child.getMeasuredState());
                ...
            }
        }

        //最大值需要考虑前景预留的padding
        maxWidth += getPaddingLeftWithForeground() + getPaddingRightWithForeground();
        maxHeight += getPaddingTopWithForeground() + getPaddingBottomWithForeground();

        //预留的尺寸是否小于最小尺寸
        maxHeight = Math.max(maxHeight, getSuggestedMinimumHeight());
        maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());

        //预留的尺寸是否小于前景最小尺寸
        final Drawable drawable = getForeground();
        if (drawable != null) {
            maxHeight = Math.max(maxHeight, drawable.getMinimumHeight());
            maxWidth = Math.max(maxWidth, drawable.getMinimumWidth());
        }

        //将测量结果记录---->(3)
        setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
                resolveSizeAndState(maxHeight, heightMeasureSpec,
                        childState << MEASURED_HEIGHT_STATE_SHIFT));

        //此处有段特殊处理，忽略
    }
```

可以看出，FrameLayout 遍历其子布局，计算出尺寸最大值，将该值作为其测量后的尺寸。上面注释出列出了比较重要的三点，接下来逐一分析：

**(1) measureChildWithMargins(xx)**

```csharp
#ViewGroup.java
    protected void measureChildWithMargins(View child,
                                           int parentWidthMeasureSpec, int widthUsed,
                                           int parentHeightMeasureSpec, int heightUsed) {
        //传入待测量的子布局 child
        //传入来自父布局的测量结果
        //传入宽、高方向上已使用过的空间尺寸
        //提取子布局的LayoutParams参数，重点是子布局声明的layout_with/layout_height参数
        final ViewGroup.MarginLayoutParams lp = (ViewGroup.MarginLayoutParams) child.getLayoutParams();
        //获取子布局的测量结果
        //将自己的padding和子布局的margin考虑进去
        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                        + widthUsed, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                        + heightUsed, lp.height);

        //调用子布局的测量方法，将父布局测量结果传递给子布局
        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```

测量单个子布局，父布局将自己父布局传递给自己的测量结果作为入参，配合LayoutParams等计算出要传给子布局的测量结果，最后将测量结果传递给子布局，让其发起测量过程。如果子布局是个ViewGroup，那么继续测量ViewGroup的子布局，如果子布局是View，那么测量View后就完成了测量。如此递归下去，最终将子布局测量完毕。

继续来看看父布局为子布局测量结果的具体实现：

```csharp
#ViewGroup.java
    public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        //childDimension 为layout_width/layout_height的值，用整数表示
        int specMode = MeasureSpec.getMode(spec);
        int specSize = MeasureSpec.getSize(spec);

        //padding为其他已使用的空间，也就是说该布局里已使用了padding，这部分已经不能分配了，只能分配余下的空间了。
        int size = Math.max(0, specSize - padding);
        int resultSize = 0;
        int resultMode = 0;

        switch (specMode) {
            case MeasureSpec.EXACTLY:
                //父布局的测量模式是确切的
                if (childDimension >= 0) {
                    //子布局声明了自己想要确切的尺寸：childDimension，那么父布局就答应它，并且子布局的测量模式为确切的
                    resultSize = childDimension;
                    resultMode = MeasureSpec.EXACTLY;
                } else if (childDimension == ViewGroup.LayoutParams.MATCH_PARENT) {
                    //子布局声明了自己想要的尺寸是与父布局一样大，那么父布局就答应它，将剩下的尺寸给它
                    //因为父布局是确切的尺寸，因此子布局的测量模式为确切的
                    resultSize = size;
                    resultMode = MeasureSpec.EXACTLY;
                } else if (childDimension == ViewGroup.LayoutParams.WRAP_CONTENT) {
                    //子布局声明了自己想要的尺寸是包括内容，它自己当前也不知道具体要多大，那么父布局就答应它，将剩下的尺寸给它
                    //因为不知道自己具体多大，因此子布局测量模式为AT_MOST(最多）
                    resultSize = size;
                    resultMode = MeasureSpec.AT_MOST;
                }
                break;

            case MeasureSpec.AT_MOST:
                if (childDimension >= 0) {
                    //与上面一致
                    resultSize = childDimension;
                    resultMode = MeasureSpec.EXACTLY;
                } else if (childDimension == ViewGroup.LayoutParams.MATCH_PARENT) {
                    //虽然子布局声明了与父布局一样大，但是父布局是AT_MOST，父布局也不知道自己有多大，因此给子布局的测量模式是AT_MOST(最多）
                    resultSize = size;
                    resultMode = MeasureSpec.AT_MOST;
                } else if (childDimension == ViewGroup.LayoutParams.WRAP_CONTENT) {
                    //与上面一致
                    resultSize = size;
                    resultMode = MeasureSpec.AT_MOST;
                }
                break;

            // Parent asked to see how big we want to be
            case MeasureSpec.UNSPECIFIED:
                ...
                break;
        }

        //通过MeasureSpec 封装测量模式和尺寸
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }
```

如此，父布局已经为其子布局测量出结果了，子布局拿到结果后继续发起测量。
 child.measure(childWidthMeasureSpec, childHeightMeasureSpec)里会调用child.onMeasure(childWidthMeasureSpec, childHeightMeasureSpec)。

**(2) state计算**
父布局将子布局的state整合到一个 int 值里，子布局的state如何获取呢？

```csharp
#View.java
    public final int getMeasuredState() {
        //测量值为int 类型
       //对于宽的测量值，将其最高字节提取出来，假设提取出的值为0x11
       //对于高的测量值，将其最高字节提取出来，假设提取出来的值为0x22
      //将上述提取出来的值，组合到一个int值里，其中宽的提取值存放在新值的最高一个字节里，而高的提取值存放在新值的第三个字节里
      //最后的state=0x11002200
        return (mMeasuredWidth&MEASURED_STATE_MASK)
                | ((mMeasuredHeight>>MEASURED_HEIGHT_STATE_SHIFT)
                        & (MEASURED_STATE_MASK>>MEASURED_HEIGHT_STATE_SHIFT));
    }
```

**(3) 记录测量结果**
 记得我们在View测量过程有提过：getDefaultSize(xx)从父布局的测量结果里取出对应的尺寸值，并调用setMeasuredDimension(xx)记录尺寸值。而此处同样是调用了setMeasuredDimension(xx)记录尺寸值，却没有使用getDefaultSize(xx)方法，而是使用了resolveSizeAndState(xx)。

```csharp
#View.java
    public static int resolveSizeAndState(int size, int measureSpec, int childMeasuredState) {
        //size:自己想要的尺寸
        final int specMode = MeasureSpec.getMode(measureSpec);
        //父布局给的尺寸
        final int specSize = MeasureSpec.getSize(measureSpec);
        final int result;
        switch (specMode) {
            case MeasureSpec.AT_MOST:
                if (specSize < size) {
                    //若是自己想要的尺寸超出了父布局能给的尺寸
                    //那么只能给到父布局测量的尺寸，并将"父布局给得太小"标记记录在测量尺寸的最高字节的第4位上
                    //该标记在ViewRootImpl.java里会判断，尝试将Window尺寸放大以期满足子布局想要的达到的尺寸
                    //一些其他的有需求的ViewGroup也会处理该标记位
                    result = specSize | MEASURED_STATE_TOO_SMALL;
                } else {
                    //否则，使用自己需要的尺寸
                    result = size;
                }
                break;
            case MeasureSpec.EXACTLY:
                //如果是确切模式，那么直接使用父布局测量的尺寸
                result = specSize;
                break;
            case MeasureSpec.UNSPECIFIED:
            default:
                result = size;
        }
        //最后，尺寸值拿到了，将之与state组合成一个int值，其中最高字节存储state，低三字节存储特定的尺寸值
        return result | (childMeasuredState & MEASURED_STATE_MASK);
    }
```

我们之前因为getDefaultSize(xx)获取的尺寸不合理而重写了onMeasure(xx)，并新增了getMyViewSize(xx)代替getDefaultSize(xx)方法。从上面可以看出，我们其实不用新增getMyViewSize(xx)，直接使用View. resolveSizeAndState(xx)从测量结果里获取相应的尺寸值。
 以上分析了**父布局是如何确定子布局的测量模式和测量尺寸**。
 父布局、子布局真正保存测量值的地方是在onMeasure(xx)里，那么问题来了：**是谁将ViewGroup和View 的onMeasure(xx)方法联系起来了？换句话说：是谁在调用onMeasure(xx)**
 答案是：**measure(xx) 方法**

```csharp
#View.java
    public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
        ...
        //key-value 缓存测量结果
        long key = (long) widthMeasureSpec << 32 | (long) heightMeasureSpec & 0xffffffffL;
        if (mMeasureCache == null) mMeasureCache = new LongSparseLongArray(2);
        
        //PFLAG_FORCE_LAYOUT 在View.requestLayout()里赋值
        //通常来说，forceLayout=true
        final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;
        
        //如果上次父布局给的测量结果与此次不同，那么表示尺寸变了
        final boolean specChanged = widthMeasureSpec != mOldWidthMeasureSpec
                || heightMeasureSpec != mOldHeightMeasureSpec;
        //是否是Exactly模式
        final boolean isSpecExactly = MeasureSpec.getMode(widthMeasureSpec) == MeasureSpec.EXACTLY
                && MeasureSpec.getMode(heightMeasureSpec) == MeasureSpec.EXACTLY;
        //当前测量的尺寸与父布局给的尺寸是否一致
        final boolean matchesSpecSize = getMeasuredWidth() == MeasureSpec.getSize(widthMeasureSpec)
                && getMeasuredHeight() == MeasureSpec.getSize(heightMeasureSpec);
        //在6.0及其以下，sAlwaysRemeasureExactly=true
        //是否需要重新布局
        final boolean needsLayout = specChanged
                && (sAlwaysRemeasureExactly || !isSpecExactly || !matchesSpecSize);
        //如果强制布局或者需要重新布局
        if (forceLayout || needsLayout) {
            //清空"测量值已记录"标记位
            mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;
            resolveRtlPropertiesIfNeeded();
            //如果不是强制重新布局，则从缓存里取结果
            int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);
            //如果没有缓存
            if (cacheIndex < 0 || sIgnoreMeasureCache) {
                //调用onMeasure 开始测量
                onMeasure(widthMeasureSpec, heightMeasureSpec);
                //PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT 标记用来判断在layout之前是否需要重新onMeasure
                mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            } else {
                //有缓存，取出来，并记录
                long value = mMeasureCache.valueAt(cacheIndex);
                setMeasuredDimensionRaw((int) (value >> 32), (int) value);
                mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            }
            
            //PFLAG_MEASURED_DIMENSION_SET 标记没有设置，那么直接抛出异常，也就是说我们必须要记录View测量后的尺寸值
            //PFLAG_MEASURED_DIMENSION_SET 标记位 在setMeasuredDimensionRaw(xx)里赋值
            if ((mPrivateFlags & PFLAG_MEASURED_DIMENSION_SET) != PFLAG_MEASURED_DIMENSION_SET) {
                throw new IllegalStateException("View with id " + getId() + ": "
                        + getClass().getName() + "#onMeasure() did not set the"
                        + " measured dimension by calling"
                        + " setMeasuredDimension()");
            }

            //如果设置了 PFLAG_LAYOUT_REQUIRED 标记位，则会调用onLayout(xx)
            mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
        }
        
        //记录父布局给的测量结果
        mOldWidthMeasureSpec = widthMeasureSpec;
        mOldHeightMeasureSpec = heightMeasureSpec;
        //加入到缓存里
        mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |
                (long) mMeasuredHeight & 0xffffffffL); // suppress sign extension
    }
```

可以看出:

> - 父布局与子布局测量时联系的桥梁是：measure(xx)方法
> - measure(xx) 不能重写

## 小结

以上分析了FrameLayout Measure过程，FrameLayout作为ViewGroup子类，其Measure过程比较有代表性。实际上，无论哪种ViewGroup，其测量过程的核心思想没有变：

> 1、根据子布局的layout_xx参数，结合从父布局拿到测量结果，生成子布局的测量结果
>  2、将为子布局生成的测量结果传递给子布局，子布局进行1步骤。这是个递归的过程，当遇到的子布局是View时，递归结束，开始回溯
>  3、根据子布局自己测量后的结果，结合父布局给自己的测量结果，记录下自己的测量值，至此一个ViewGroup测量完毕

用图表示其测量流程：

![img](https://upload-images.jianshu.io/upload_images/19073098-2be23ffab321e009.png?imageMogr2/auto-orient/strip|imageView2/2/w/378/format/webp)

![img](https://upload-images.jianshu.io/upload_images/19073098-e6572603e2dff7e5.png?imageMogr2/auto-orient/strip|imageView2/2/w/962/format/webp)

再次来区分ViewGroup/View onMeasure(xx)区别：

> 1、View onMeasure(xx) 负责将ViewGroup给的测量结果，经过一些权衡后记录测量后的尺寸值到成员变量里
>  2、ViewGroup onMeasure(xx) 首先遍历测量其子布局，然后根据子布局测量结果，结合其父布局给的测量结果，经过一些权衡后记录测量后的尺寸值到成员变量里
>  3、继承自ViewGroup，必须要重写onMeasure(xx)方法才能为里边子布局测量结果

