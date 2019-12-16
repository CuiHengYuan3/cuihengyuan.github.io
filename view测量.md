

activity的生命周期是有系统服务所触发,由系统服务发起handle调用到handleResumeActivity()开始绘制流程然后最终交由ViewRootImpl调用到performTraversals()

 然后依次之行了我们UI的实际绘制流程  measure(测量)，layout(布局摆放),Draw(具体绘制) 



在performMeasure之前

```java
    int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
            int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
             performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
```

```java
private static int getRootMeasureSpec(int windowSize, int rootDimension) {
    int measureSpec;
    switch (rootDimension) {

    case ViewGroup.LayoutParams.MATCH_PARENT:
        // Window can't resize. Force root view to be windowSize.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
        break;
    case ViewGroup.LayoutParams.WRAP_CONTENT:
        // Window can resize. Set max size for root view.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
        break;
    default:
        // Window wants to be an exact size. Force root view to be that size.
        measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
        break;
    }
    return measureSpec;
}
```

这里有我们很熟悉的东西，MeasureSpec
 MeasureSpec的作用是在在Measure流程中，系统将View的LayoutParams根据父容器所施加的规则转换成对应的MeasureSpec（规格）
 然后在onMeasure中根据这个MeasureSpec来确定view的测量宽高

```java
   public static class MeasureSpec {
    private static final int MODE_SHIFT = 30;
    private static final int MODE_MASK  = 0x3 << MODE_SHIFT; 
     /**
      * UNSPECIFIED 模式：
      * 父View不对子View有任何限制，子View需要多大就多大
      */ 
    public static final int UNSPECIFIED = 0 << MODE_SHIFT;

    /**
      * EXACTYLY 模式：
      * 父View已经测量出子Viwe所需要的精确大小，这时候View的最终大小
      * 就是SpecSize所指定的值。对应于match_parent和精确数值这两种模式
      */ 
    public static final int EXACTLY     = 1 << MODE_SHIFT;

    /**
      * AT_MOST 模式：
      * 子View的最终大小是父View指定的SpecSize值，并且子View的大小不能大于这个值，
      * 即对应wrap_content这种模式
      */ 
    public static final int AT_MOST     = 2 << MODE_SHIFT;

    //将size和mode打包成一个32位的int型数值
    //高2位表示SpecMode，测量模式，低30位表示SpecSize，某种测量模式下的规格大小
    public static int makeMeasureSpec(int size, int mode) {
        if (sUseBrokenMakeMeasureSpec) {
            return size + mode;
        } else {
            return (size & ~MODE_MASK) | (mode & MODE_MASK);
        }
    }

    //将32位的MeasureSpec解包，返回SpecMode,测量模式
    public static int getMode(int measureSpec) {
        return (measureSpec & MODE_MASK);
    }

    //将32位的MeasureSpec解包，返回SpecSize，某种测量模式下的规格大小
    public static int getSize(int measureSpec) {
        return (measureSpec & ~MODE_MASK);
    }
    //...
}

测量模式
    EXACTLY ：父容器已经测量出所需要的精确大小，这也是childview的最终大小
            ------match_parent，精确值是父容器的

    ATMOST : child view最终的大小不能超过父容器的给的
            ------wrap_content 精确值不超过父容器

    UNSPECIFIED: 不确定，源码内部使用
            -------一般在ScrollView，ListView 
```

这里使用到了位运算，包括打包和解包的部分

所以，View的测量流程中，通过makeMeasureSpec来保存宽高信息，在其他流程通过getMode或getSize得到模式和宽高。在得到了DecorView的MeasureSpec后（DecorView的父布局是系统的布局），来看看干了什么吧

```java
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
    try {
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

这里明显调用了view的measure方法，从顶层开始了测量。

```java
 public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    boolean optical = isLayoutModeOptical(this);
    if (optical != isLayoutModeOptical(mParent)) {
    ...
    if ((mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ||
            widthMeasureSpec != mOldWidthMeasureSpec ||
            heightMeasureSpec != mOldHeightMeasureSpec) {
        ...
        if (cacheIndex < 0 || sIgnoreMeasureCache) {
            // measure ourselves, this should set the measured dimension flag back
            onMeasure(widthMeasureSpec, heightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        } 
    ...
 }
```

可以看到，它在内部调用了onMeasure方法所以这里开始调用onMeasure
 那么注意，到此为止，我们的布局容器的核心就在这里了，
 不管是LinearLayout或者是FreamLayout还是其他布局， 他们都是通过测量组件，实现我们的布局定位，每一个Layout的onMeasure实现都不一样

这里首先测量的是一个 FramLayout

```dart
  参照FreamLayout作为案例
 
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
//获取当前布局内的子View数量
int count = getChildCount();

//判断当前布局的宽高是否是match_parent模式或者指定一个精确的大小，如果是则置measureMatchParent为false.
final boolean measureMatchParentChildren =
        MeasureSpec.getMode(widthMeasureSpec) != MeasureSpec.EXACTLY ||
        MeasureSpec.getMode(heightMeasureSpec) != MeasureSpec.EXACTLY;
mMatchParentChildren.clear();

int maxHeight = 0;
int maxWidth = 0;
int childState = 0;

//遍历所有类型不为GONE的子View
for (int i = 0; i < count; i++) {
    final View child = getChildAt(i);
    if (mMeasureAllChildren || child.getVisibility() != GONE) {
        //对每一个子View进行测量
        measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
        final LayoutParams lp = (LayoutParams) child.getLayoutParams();
        //寻找子View中宽高的最大者，因为如果FrameLayout是wrap_content属性
        //那么它的大小取决于子View中的最大者
        maxWidth = Math.max(maxWidth,
                child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
        maxHeight = Math.max(maxHeight,
                child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
        childState = combineMeasuredStates(childState, child.getMeasuredState());
        //如果FrameLayout是wrap_content模式，那么往mMatchParentChildren中添加
        //宽或者高为match_parent的子View，因为该子View的最终测量大小会受到FrameLayout的最终测量大小影响
        if (measureMatchParentChildren) {
            if (lp.width == LayoutParams.MATCH_PARENT ||
                    lp.height == LayoutParams.MATCH_PARENT) {
                mMatchParentChildren.add(child);
            }
        }
    }
}

// Account for padding too
maxWidth += getPaddingLeftWithForeground() + getPaddingRightWithForeground();
maxHeight += getPaddingTopWithForeground() + getPaddingBottomWithForeground();

// Check against our minimum height and width
maxHeight = Math.max(maxHeight, getSuggestedMinimumHeight());
maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());

// Check against our foreground's minimum height and width
final Drawable drawable = getForeground();
if (drawable != null) {
    maxHeight = Math.max(maxHeight, drawable.getMinimumHeight());
    maxWidth = Math.max(maxWidth, drawable.getMinimumWidth());
}

//保存测量结果
setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
        resolveSizeAndState(maxHeight, heightMeasureSpec,
                childState << MEASURED_HEIGHT_STATE_SHIFT));

//子View中设置为match_parent的个数
count = mMatchParentChildren.size();
//只有FrameLayout的模式为wrap_content的时候才会执行下列语句
if (count > 1) {
    for (int i = 0; i < count; i++) {
        final View child = mMatchParentChildren.get(i);
        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

        //对FrameLayout的宽度规格设置，因为这会影响子View的测量
        final int childWidthMeasureSpec;

        /**
          * 如果子View的宽度是match_parent属性，那么对当前FrameLayout的MeasureSpec修改：
          * 把widthMeasureSpec的宽度规格修改为:总宽度 - padding - margin，这样做的意思是：
          * 对于子Viw来说，如果要match_parent，那么它可以覆盖的范围是FrameLayout的测量宽度
          * 减去padding和margin后剩下的空间。
          *
          * 以下两点的结论，可以查看getChildMeasureSpec()方法：
          *
          * 如果子View的宽度是一个确定的值，比如50dp，那么FrameLayout的widthMeasureSpec的宽度规格修改为：
          * SpecSize为子View的宽度，即50dp，SpecMode为EXACTLY模式
          * 
          * 如果子View的宽度是wrap_content属性，那么FrameLayout的widthMeasureSpec的宽度规格修改为：
          * SpecSize为子View的宽度减去padding减去margin，SpecMode为AT_MOST模式
          */
        if (lp.width == LayoutParams.MATCH_PARENT) {
            final int width = Math.max(0, getMeasuredWidth()
                    - getPaddingLeftWithForeground() - getPaddingRightWithForeground()
                    - lp.leftMargin - lp.rightMargin);
            childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(
                    width, MeasureSpec.EXACTLY);
        } else {
            childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                    getPaddingLeftWithForeground() + getPaddingRightWithForeground() +
                    lp.leftMargin + lp.rightMargin,
                    lp.width);
        }
        //同理对高度进行相同的处理，这里省略...

        //对于这部分的子View需要重新进行measure过程
        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
}
```

 

总结一下：首先，FrameLayout根据它的MeasureSpec来对每一个子View进行测量，即调用measureChildWithMargin方法，这个方法下面会详细说明；对于每一个测量完成的子View，会寻找其中最大的宽高，那么FrameLayout的测量宽高会受到这个子View的最大宽高的影响(wrap_content模式)，接着调用setMeasureDimension方法，把FrameLayout的测量宽高保存。最后则是特殊情况的处理，即当FrameLayout为wrap_content属性时，如果其子View是match_parent属性的话，则要重新设置FrameLayout的测量规格，然后重新对该部分View测量。

在上面提到setMeasureDimension方法，该方法用于保存测量结果，在上面的源码里面，该方法的参数接收的是resolveSizeAndState方法的返回值那么我们直接看View#resolveSizeAndState方法：

```java
 public static int resolveSizeAndState(int size, int measureSpec, int childMeasuredState) {
final int specMode = MeasureSpec.getMode(measureSpec);
final int specSize = MeasureSpec.getSize(measureSpec);
final int result;
switch (specMode) {
    case MeasureSpec.AT_MOST:
        if (specSize < size) {
            result = specSize | MEASURED_STATE_TOO_SMALL;
        } else {
            result = size;
        }
        break;
    case MeasureSpec.EXACTLY:
        result = specSize;
        break;
    case MeasureSpec.UNSPECIFIED:
    default:
        result = size;
}
return result | (childMeasuredState & MEASURED_STATE_MASK);
 }
```

可以看到该方法的思路是相当清晰的，当specMode是EXACTLY时，那么直接返回MeasureSpec里面的宽高规格，作为最终的测量宽高；当specMode时AT_MOST时，那么取MeasureSpec的宽高规格和size的最小值

子View测量

上面在FrameLayout测量内提到的measureChildWithMargin方法，它接收的主要参数是子View以及父容器的MeasureSpec，所以它的作用就是对子View进行测量，那么我们直接看这个方法

```java
  protected void measureChildWithMargins(View child,
    int parentWidthMeasureSpec, int widthUsed,
    int parentHeightMeasureSpec, int heightUsed) {
final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
        mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                + widthUsed, lp.width);
final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
        mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                + heightUsed, lp.height);

child.measure(childWidthMeasureSpec, childHeightMeasureSpec); // 1
 }
```

里面调用了getChildMeasureSpec方法，把父容器的MeasureSpec以及自身的layoutParams属性传递进去来获取子View的MeasureSpe，在这里我们可以看到直接又调用了子类的measure

总结：那么现在我们能得到整体的测量流程：在performTraversals开始获得DecorView种的系统布局的尺寸，然后在performMeasure方法中开始测量流程，对于不同的layout布局有着不同的实现方式，但大体上是在onMeasure方法中，对每一个子View进行遍历，根据ViewGroup的MeasureSpec及子View的layoutParams来确定自身的测量宽高，然后最后根据所有子View的测量宽高信息再确定爸爸的宽高
 不断的遍历子View的measure方法，根据ViewGroup的MeasureSpec及子View的LayoutParams来决定子View的MeasureSpec，进一步获取子View的测量宽高，然后逐层返回，不断保存ViewGroup的测量宽高。

