---
title: 深度解析属性动画的思想 - 带你手动实现属性动画框架
date: 2019-05-09 14:32:07
tags: 
	- android 进阶
---

属性动画在我们日常使用频率也是比较多的，源码解析网上也是比较多的，但是很多同学表示在看源码的时候看的云里雾里，看到最后其实也并不能理解到源码架构的原理和精髓所在，所以我们这里脱开在源码里的纠缠，我带大家从参考源码的角度，如何设计一个精简版的属性动画框架，用以更好的理解属性动画源码设计的原理和思想。

### 属性动画是什么？

首先，我们既然要写一个属性动画，我们必然要了解属性动画究竟是个啥玩意儿？它能起到什么样的作用？

##### 动画的本质

在原先 Android 3.0 版本之前的定义上来说，Android的补间动画和逐帧动画都是对图片或View 进行动画。而 3.0 版本之后，Google 爸爸推出了 属性动画。 我们首先将这个词分拆开来，属性和动画，属性就是一个对象的属性，动画就是在一定时间内以一定的速度改变View（或一个对象） 的属性状态，包括但不限于View 的位置、大小、透明度等。

##### 属性动画和补间动画的区别？

在3.0之前，补间动画能够帮我们实现对View的移动、缩放、旋转和淡入淡出，但这也仅仅局限于继承自View对象。可能有的同学比较奇怪的是，除了View在显示上需要动画，其他还有什么场景下需要用到动画呢？比如在我们的自定义View控件中，如果我们在onDraw()方法中绘制了 Point，我们可以对这个 Point 对象的进行动画，又或者我只需要修改一个View的背景色的透明度，补间动画也只能望View兴叹。

其次我们在动画过程中，补间动画只能改变View显示的状态，并没有修改 View 真正的属性，比如我们将一个按钮从屏幕上方移动到了屏幕下方，但是你如果在动画完成后点击Button显示的区域，这时候你会发现，无法点击，因为按钮的坐标属性并未得到修改。

所以总而言之，属性动画也就是 可以对一个对象其中的属性进行动画操作，而不单单局限于View。这就是属性动画的核心思想。

##### 属性动画的使用

古话说的好，要知其然，也要知其所以然。那我们在知其所以然之前，首先知其然。看看属性动画如何调用。

我们举个最简单的例子，在将一个TextView横向缩放到1.5倍大小。

```java
TextView tvTip = findViewById(R.id.tv_tip);
ObjectAnimator objectAnimator = ObjectAnimator
                .ofFloat(tvTip, "scaleX", 1f, 1.5f);
objectAnimator.start();
```

这就是属性动画最简单的调用方法了，我们在这里也可以完全看到刚刚说的属性动画的定义: 针对 `tvTip`属性 `scaleX`从1倍缩放到1.5倍。

### 自行设计一个属性动画框架

下面我们就来仿照 Android 源码里的属性动画的实现，来自行实现一个简单版本的属性动画，让我们更好的理解属性动画运作的原理。

##### 设计一个动画框架，我们需要考虑哪些因素？

1. 首先，要考虑到调用API的简洁易用性，应该说越简单直接越好
2. 每个View（对象）可以有很多个动画，但同时只有一个动画在运行
3. 因为动画执行的过程中需要时间来完成，动画的执行不能依赖自身的for循环代码，这样会造成极大的资源损耗
4. 如何让动画平滑的动起来？

##### 开始我们的设计之旅

带着这些问题，我们开始来自行设计一个属性动画框架。

###### 架构设计

首先，我们需要考虑一个动画任务包含哪些元素？

1. 对象本身，大部分情况下都是View
2. 动画时长
3. 动画的起始值和结束值
4. 动画运行的速度效果，也就是我们所熟知的插值器

其次，我们需要先了解到几个重要概念：

1. 关键帧

   在动画运行之前，我们会将一个动画任务分解成若干个关键帧。这也就类似于我们需要规划从南京到上海，中间需要规划大致什么时间经过常州、无锡、苏州，初步估算后经过多久能够到达上海。

   动画也是这样，需要时间去完成，而在开始之前，我们需要将这个动画分拆到不同的时间节点的状态，有了目标，才能有方向，这就是关键帧。

2. 插值器：TimeInterpolator

   时间插值器，作用是根据时间节点的不同计算出当前属性改变的百分比，例举几个常用的插值器：LinearInterpolator(线性插值器)、AccelerateDecelerateInterpolator(加速减速插值器)、DecelerateInterpolator(减速插值器)。

   依然用南京到上海的例子，我们计划4小时从南京开到上海，如果是线性插值器，则以匀速的速度从南京到达上海。如果是使用减速插值器，则南京出发后，因为就以高速驾驶，距离上海越近则开的越慢。

3. 估值器：TypeEvaluator

   不论怎么样，我们终究是通过改变对象的属性值去完成动画，而设置属性值则是一个具体的数值。所以估值器的作用就是根据插值器计算当前时间节点改变的百分比去计算出最终的具体改变的属性值。

   

我们仿照源码中的属性动画的原理，搭建了我们自己的属性动画架构：

![大致的流程架构图（画的丑请见谅）](https://ws2.sinaimg.cn/large/006tNc79gy1g2xkvrjptnj30p80k7di0.jpg)

1. 首先，在我们初始化一个动画任务`MyObjectAnimator`的时候，会去生成一个属性设置助理 `MyFloatPropertyValuesHolder` ，用于管理我们需要去设置的属性值和关键帧。而 `MyFloatPropertyValuesHolder` 会去生成关键帧管理类`MyKeyframeSet`, 并在开始之前生成若干关键帧`MyFloatKeyframe`。
2. 当动画任务开始后，在系统中，会在属性动画中通过监听 VSync 信号，进行动画触发，我们这里会使用`VSYNCManager` 模拟VSync 信号，并在动画开始时，添加监听。
3. 当在动画任务中监听到VSync 信号后，我们将通过执行次数以及插值器计算到当前的百分比，并将其传入`MyFloatPropertyValuesHolder`通过关键帧管理类`MyKeyframeSet`计算到最终此时此刻需要设置的属性值,并设置。
4. 当动画完成后，动画任务类`MyFloatPropertyValuesHolder` 会重置当前执行的状态，并根据动画是否重复，清空对 VSync 信号的监听。

这就是一套完整的执行流程，可能有的同学看到这，还是一头雾水，这究竟是个什么流程，不要紧，下面我们 Show your the code，用代码理清思路。

###### 代码实现

1. 首先是初始化一个动画任务`MyObjectAnimator`，这里和原生属性动画的初始化无差。

   ```java
   /**
    * 初始化一个我们自己写的动画任务
    */
   MyObjectAnimator objectAnimator = MyObjectAnimator
                   .ofFloat(tv_tip, "scaleX", 1f, 2f);
   objectAnimator.setDuration(500);
   objectAnimator.setRepeat(false);
   objectAnimator.setTimeInterpolator(new LineInterpolator());
   objectAnimator.start();
   ```

2. 新建`MyObjectAnimator`类，并在`MyObjectAnimator.ofFloat()`方法中实现对`MyObjectAnimator` 的初始化，以及对插值器，时长和重复Flag 的设置。

   ```java
   /**
    * 动画任务
    */
   public class MyObjectAnimator {
       /**
        * 当前操作对象
        */
       private WeakReference<Object> target;
   
       /**
        * 是否重复
        */
       private boolean repeat = false;
   
       /**
        * 运行时长
        */
       private long mDuration = 300;
   
       /**
        * 差值器
        */
       private TimeInterpolator timeInterpolator;
   
       /**
        * 动画助理，用于设置属性值
        */
       private MyFloatPropertyValuesHolder myFloatPropertyValuesHolder;
   
       /**
        * 构造方法
        * <p>
        * 初始化动画属性助理 MyFloatPropertyValuesHolder，用于设置参数，并将动画进行分解成N个关键帧，
        *
        * @param target       动画对象
        * @param propertyName 需要修改的属性名, View 中的属性名且必须有对应 setter 和 getter
        * @param values       关键帧的节点参数
        */
       public MyObjectAnimator(Object target, String propertyName, float... values) {
           this.target = new WeakReference<>(target);
           myFloatPropertyValuesHolder = new MyFloatPropertyValuesHolder(propertyName, values);
       }
   
       /**
        * 初始化动画任务
        *
        * @param target
        * @param propertyName
        * @param values
        * @return
        */
       public static MyObjectAnimator ofFloat(Object target, String propertyName, float... values) {
           MyObjectAnimator anim = new MyObjectAnimator(target, propertyName, values);
           return anim;
       }
   
       /**
        * 设置时间插值器
        *
        * @param timeInterpolator
        */
       public void setTimeInterpolator(TimeInterpolator timeInterpolator) {
           this.timeInterpolator = timeInterpolator;
       }
       /**
        * 设置是否重复
        * @param repeat
        */
       public void setRepeat(boolean repeat) {
           this.repeat = repeat;
       }
   
       /**
        * 设置时长
        * @param duration
        */
       public void setDuration(long duration) {
           this.mDuration = duration;
       }
     
       /**
        * 启动动画
        */
       public void start() {
         	// 第6步 进行实现
       }
   }
   ```

3. 下面我们新建动画属性助理类`MyFloatPropertyValuesHolder`, 并完成构造方法，构造方法中，会根据需要修改的属性名生成对应的 Setter 方法，所以在我们的属性动画中，传入的属性名必须是在所属对象类中有对应的Setter方法才行。

   ```java
   /**
    * 动画 属性"助理"
    * 【对应源码里 PropertyHolder】
    * 用于对当前对象进行设值，反射设置
    */
   public class MyFloatPropertyValuesHolder {
       /**
        * 属性名
        */
       String mPropertyName;
   
       /**
        * float 类型 Class
        */
       Class mValueType;
   
       /**
        * 关键帧管理类
        */
       MyKeyframeSet myKeyframeSet;
   
     	/**
     	 * 设置属性的 Setter 方法，通过反射方法Object.class.getMethod()方法生成
     	 */
    		Method mSetter = null;
     
       /**
        * 构造方法
        * 初始化关键帧管理类
        *
        * @param propertyName 属性名
        * @param values 关键帧节点属性参数
        */
       public MyFloatPropertyValuesHolder(String propertyName, float... values) {
           this.mPropertyName = propertyName;
           mValueType = float.class;
           myKeyframeSet = MyKeyframeSet.ofFloat(values);
         
          	setupSetter();
       }
     
      /**
        * 通过反射方法 Object.class.getMethod()方法生成对应的Setter方法
        */
       public void setupSetter() {
           // 获取对应属性的 Setter
           char firstLetter = Character.toUpperCase(mPropertyName.charAt(0));
           String theRest = mPropertyName.substring(1);
           String methodName = "set" + firstLetter + theRest;
   
           try {
               mSetter = View.class.getMethod(methodName, float.class);
   
           } catch (NoSuchMethodException e) {
               e.printStackTrace();
           }
       }
   }
   ```

4. 新建关键帧管理类`MyKeyframeSet`，并实现`MyKeyframeSet.ofFloat(values)`方法，这一步非常重要，重磅人物 关键帧 就是在这里 通过遍历传入的节点参数被初始化出来，保存在List中。

   ```java
   /**
    * 关键帧管理类
    */
   public class MyKeyframeSet {
       /**
        * 第一个关键帧
        */
       MyFloatKeyframe mFirstKeyframe;
   
       /**
        * 帧队列，mFirstKeyframe 其实就是第0个元素
        */
       List<MyFloatKeyframe> myFloatKeyframes;
   
       /**
        * 类型估值器
        *
        * 相当于差速器，用在关键帧中间穿插，进行不同速度的处理。
        */
       TypeEvaluator mTypeEvaluator;
   
       /**
        * 根据传入的节点参数，生成关键帧数组
        *
        * @param values
        * @return
        */
       public static MyKeyframeSet ofFloat(float... values) {
           // 总共有多少关键帧节点
           int frameCount = values.length;
           // 遍历关键帧参数，并生成关键帧
           MyFloatKeyframe[] myFloatKeyframes = new MyFloatKeyframe[frameCount];
           myFloatKeyframes[0] = new MyFloatKeyframe(0, values[0]);
           // 遍历关键帧节点数量，并计算关键帧处于的百分比，初始化对应的关键帧
           for (int i = 1; i < frameCount; ++i) {
               myFloatKeyframes[i] = new MyFloatKeyframe((float) i / (frameCount - 1), values[i]);
           }
           return new MyKeyframeSet(myFloatKeyframes);
       }
   
       /**
         * 构造函数
         * 初始化传入的关键帧数组，以及估值器
         * @param keyframes
         */
        public MyKeyframeSet(MyFloatKeyframe... keyframes) {
            myFloatKeyframes = Arrays.asList(keyframes);
            mFirstKeyframe = keyframes[0];
            // 这里直接使用Android原生的 Float类型估值器，android.animation.FloatEvaluator
            mTypeEvaluator = new FloatEvaluator();
        }
   }
   ```

   新建关键帧实体类: `MyFloatKeyframe`，用于保存某一时刻的关键帧的状态

   ```java
   /**
    * 关键帧 
    * 保存某一时刻的具体状态
    * ps. 初始化动画任务的时候已经完成初始化
    */
   public class MyFloatKeyframe {
       /**
        * 当前的帧所处的百分比， 范围 0 - 1
        */
       float mFraction;
       /**
        * 每一个关键帧，具体设置的参数
        */
       float mValue;
   		
       Class mValueType;
       public MyFloatKeyframe(float mFraction, float mValue) {
           this.mFraction = mFraction;
           this.mValue = mValue;
           this.mValueType = float.class;
       }
   
       public float getFraction() {
           return mFraction;
       }
   
       public float getValue() {
           return mValue;
       }
   }
   ```

5. 新建线性插值器类`LineInterpolator`, 并使其实现`TimeInterpolator` 接口。

   ```java
   /**
    * 线性插值器
    * 因为是线性运动，所以则传入百分比及输出百分比
    */
   public class LineInterpolator implements TimeInterpolator {
       @Override
       public float getInterpolation(float input) {
           return input;
       }
   }
   ```

   `TimeInterpolator` 接口：

   ```java
   /**
    * 时间差值器
    *
    * 实现此接口用于修改执行百分比，以修改运行时的状态
    */
   public interface TimeInterpolator {
       float getInterpolation(float input);
   }
   ```

   > 到这里为止，我们已经完成了对属性动画的初始化工作，初始化 动画任务类 MyObjectAnimator`后，通过初始化出来的动画属性助理`MyFloatPropertyValuesHolder又初始化了 关键帧管理类`MyKeyframeSet`,  并通过传入的关键帧参数生成了关键帧数组保存在了`MyKeyframeSet`中。
   >
   > 以上几步，就是在阐述`MyObjectAnimator.ofFloat(tv_tip, "scaleY", 1f, 2f);` 背后发生的故事，下面我们说一说 `objectAnimator.start()`执行后又发生了什么。

6. 现在我们回到`MyObjectAnimator`类中的`start()`方法

   ```java
   /**
     * 启动动画
     */
    public void start() {
        // 注册监听
        VSYNCManager.getInstance().addCallbacks(this);
    }
   ```

   你会发现这里仅仅设置了一个监听事件，那么这个监听事件里发生了什么？

   我们前面有提到，在原生属性方法中，动画运动是通过监听 VSync信号机制类监听的，但是在我们第三方APP 中，无法监听VSync信号，所以，我们写了一个 `VSYNCManager` 用来模拟这个信号，信号每16ms发出一次（至于为什么，可以自行搜索 Android VSync信号）: 

   ```java
   /**
    * 模拟 VSync 信号
    *
    * 代码中使用线程模拟VSync信号，因为在第三方中无法监听到 VSync 信号。
    */
   public class VSYNCManager {
       private static final VSYNCManager mInstance = new VSYNCManager();
   
       private List<AnimationFrameCallback> callbacks = new ArrayList<>();
   
       public static VSYNCManager getInstance() {
           return mInstance;
       }
   
       /**
        * 构造方法，启动模拟VSYNC 信号的线程
        */
       private VSYNCManager() {
           new Thread(runnable).start();
       }
   
       /**
        * 添加监听
        * @param callback
        */
       public void addCallbacks(AnimationFrameCallback callback) {
           if (callback == null) {
               return;
           }
   
           callbacks.add(callback);
       }
   
   
       /**
        * 移除监听
        * @param callback
        */
       public void removeCallbacks(AnimationFrameCallback callback) {
           if (callback == null) {
               return;
           }
   
           callbacks.remove(callback);
       }
   
       /**
        * 模拟VSYNC信号，信号每16ms 发出一次
        */
       private Runnable runnable = new Runnable() {
           @Override
           public void run() {
               while (true) {
                   try {
                       // 60Hz（16ms）绘制一次
                       Thread.sleep(16);
                   } catch (InterruptedException e) {
                       e.printStackTrace();
                   }
   
                   // 模拟 VSync 信号
                   for(AnimationFrameCallback callback:callbacks) {
                       callback.doAnimationFrame(System.currentTimeMillis());
                   }
               }
           }
       };
   
   
       /**
        * 信号监听的方法，当有信号传递的时候，进行回调
        */
       public interface AnimationFrameCallback {
           boolean doAnimationFrame(long currentTime);
       }
   }
   ```

   

7. 既然上一步在`MyObjectAnimation `类的 `start()`方法中，调用了` VSYNCManager.getInstance().addCallbacks(this)`，那么我们应该很轻易的想到，将`MyObjectAnimation `类实现 `VSYNCManager.AnimationFrameCallback`接口，以在`doAnimationFrame(long currentTime)` 方法中监听到 模拟 VSync信号。

8. 在`MyObjectAnimation `类中实现`doAnimationFrame(long currentTime)`方法，这里就是让动画动起来的精髓所在。

   ```java
   /**
    * 每隔16ms 会收到信号回调，并进行对应的属性设值
    *
    * @param currentTime
    * @return
    */
   @Override
   public boolean doAnimationFrame(long currentTime) {
       // 获得到 VSync 信号, 开始属性动画
   
       // 获得应该被执行的总次数
       float total = mDuration / 16;
   
       // 计算当前执行百分比（index++）/total
       float fraction = (index++) / total;
       if (timeInterpolator != null) {
           fraction = timeInterpolator.getInterpolation(fraction);
       }
   
       // 是否重复播放
       if (index >= total) {
           index = 0;
   
           if (!repeat) {
               // 不重复，移除监听
               VSYNCManager.getInstance().removeCallbacks(this);
               return true;
           }
       }
   
       // 通过 动画属性助理 设置 对应属性值，以完成动画操作
       myFloatPropertyValuesHolder.setAnimatedValue(target.get(), fraction);
       return false;
   }
   ```

   我们会通过总时长除以信号间隔时间16ms，得到执行的总次数，在通过当前执行的次数以及总次数，就得到了当前执行的默认百分比，如果设置了插值器，则会通过插值器计算以当前预设的运行速度的运行百分比，并将这个 百分比和需要设置的对象 一并传入 动画属性助理中进行最终设值。

9. 在`MyFloatPropertyValuesHolder` 添加 `setAnimatedValue(Object target, float fraction)` 方法：

   ```java
   public void setAnimatedValue(Object target, float fraction) {
       // 通过当前的值 以及 执行百分比 计算出 需要修改的值
       Object value = myKeyframeSet.getValue(fraction);
   
       try {
           mSetter.invoke(target, value);
       } catch (IllegalAccessException e) {
           e.printStackTrace();
       } catch (InvocationTargetException e) {
           e.printStackTrace();
       }
   }
   ```

   我们可以看到在这个方法中，我们调用了 关键帧管理类的`myKeyframeSet.getValue(fraction)`方法，这个方法为整个属性动画中的重中之重（划重点）。

   在这个方法中，使用估值器计算，如果当前运行百分比在这两帧中间，则将当前百分比的上下关键帧的对应参数和当前运行百分比传入估值器，得到当前百分比所对应的数值。还是拿南京到上海举例，我们之前计划南京到上海中间有三个关键停靠站常州、无锡、苏州，当我们到达镇江（南京和常州中间的城市）的时候，需要计算我们当前行驶了多少公里，那我们就需要将我们当前行驶的百分比和南京的在行程中相对距离（0km）以及常州的在行程中相对距离传入 `估值器`，则可以得到我们当前的行驶距离。

   ```java
   /**
    * 通过传入的百分比，计算最终需要设置的具体属性值
    * @param fraction
    * @return
    */
   public Object getValue(float fraction) {
       // 关键帧之间 的位置 根据执行时间，进行计算状态
   
       // 先拿到第一帧
       MyFloatKeyframe prevKeyframe = mFirstKeyframe;
   
       // 遍历所有关键帧
       for (int i = 1; i < myFloatKeyframes.size(); ++i) {
           // 下一帧
           MyFloatKeyframe nextKeyframe = myFloatKeyframes.get(i);
   
           // 【关键】每个关键帧与关键帧之间动画状态的计算（公式见 方法中）
           // 使用估值器计算，如果当前运行百分比在这两帧中间，则将当前百分比的上下关键帧的对应参数和当前运行百分比传入估值器，得到当前百分比所对应的数值
           if (fraction < nextKeyframe.getFraction()) {
               return mTypeEvaluator.evaluate(fraction, prevKeyframe.getValue(), nextKeyframe.getValue());
           }
   
           prevKeyframe = nextKeyframe;
       }
       return null;
   }
   ```

   当我们通过`getValue(float fraction)`方法运算得到当前运行百分比对应的具体的属性数值之后，则通过之前初始化的Setter方法，通过反射调用 invoke，则对具体属性设置成功，这样就完成了动画中的一帧运行。

   当我们的信号每16ms传递一次出来的时候，就会进行一帧动画的运转，也就是对应的属性值的设置。当运行次数达到总次数之后，我们会重置运行次数，如果动画不重复，则对VSync信号移除监听即可。

### 总结

通过复原一次简单的属性动画，我们可以深刻了解到属性动画背后的思想以及运转原理，当懂得其背后的原理之后，我相信看源码时也不会迷失其中，而且对于源码理解也会更加深刻。

源码地址：https://github.com/TYKevin/CustomObjcctAnimation

最后，如果喜欢我的文章，请扫码关注公众号，当有新文章时，可以及时收到哦。

![](https://ws3.sinaimg.cn/large/006tNc79gy1g2xlzxgcgaj31ej0goait.jpg)



