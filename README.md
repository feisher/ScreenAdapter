# 五行代码搞定安卓全屏幕适配——简单粗暴-低入侵，无继承，简单高效
 
## 话不多说，先上解决方案
  ```java
  /**  将此文件直接复制到项目中，不要忘记清单文件配置Application，另 布局中使用pt
  *	（例如： android:layout_height="300pt" 用错可不适配哦！）
  *	feisher  @2017年8月11日14:52:27 二次整理，原稿 为新浪大牛 布隆  
  *	458079442@qq.com
  */
  public class MyApplication extends Application{

  	public final static float DESIGN_WIDTH = 750; //绘制页面时参照的设计图宽度

  	@Override
  	public void onCreate() {
  		super.onCreate();
  		resetDensity();//注意不要漏掉
  	}

  	@Override
  	public void onConfigurationChanged(Configuration newConfig) {
          super.onConfigurationChanged(newConfig);
          resetDensity();//这个方法重写也是很有必要的
  	}

      public void resetDensity(){
          Point size = new Point();
          ((WindowManager)getSystemService(WINDOW_SERVICE)).getDefaultDisplay().getSize(size);
          getResources().getDisplayMetrics().xdpi = size.x/DESIGN_WIDTH*72f;
      }
  }

  ```
## 为什么会有这么一篇文章呢？  

**现状**

由于Android碎片化严重，屏幕适配一直是开发中较为头疼的问题。面对市面上五花八门的屏幕大小与分辨率，Android基于dp与res目录名称来适配的方案已无法满足一次编写全屏幕适配的需求，为了达到最优的视觉效果，开发过程中总是需要花费较多资源进行适配。也有开发者给出了一些自己的解决方案。首先来分析一下一些常见的解决方案的现状：

1. ### **官方适配方案**

- dp。dp是Android开发中特有的一个单位。与px不同，dp是基于屏幕像素密度的一种单位。在密度低的屏幕上或许1dp=1px，但在密度高的屏幕上可能1dp=4px。编写布局xml时，如果一个控件的长宽都使用dp来指定，那么能确保该控件在各种大小与分辨率的屏幕下的绝对大小都大致相当。也就是说无论在pad下还是大小屏手机下，我们实际看到的该控件的大小是差不多的：

![img](https://github.com/feisher/feisher.github.io/blob/master/dp.png)

- 资源目录名。上图可见虽然使用dp确保了控件在不同屏幕中的绝对大小一致。这样的好处在于，在大小相近的屏幕中，无论分辨率多大都不会对布局造成影响；但是当屏幕大小相差较大时，仅保证控件的绝对大小看起来就有些问题了。在res目录下可以给各资源目录都加上例如'-1920x1080'等后缀来适配不同的屏幕，具体规则可见官网文档。这样可以针对不同的屏幕提供不同的布局，甚至针对pad与手机提供两套完全不同的布局样式。但是通常情况下，设计师并不会对不同屏幕提供不同的设计图，他们的需求仅仅是不同屏幕下控件对屏幕的相对大小一致，所以dp并不能满足这一点，而对各种屏幕适配一遍又显得略为繁琐，并且修改也较为麻烦。通常我们需要的适配是这样的：

![img](https://github.com/feisher/feisher.github.io/blob/master/hope.png)

- 百分比布局支持库。没有使用过，但是deprecated in API level 26.0.0-beta1。
- ConstraintLayout。百分比支持库deprecated之后推荐使用的布局，看起来似乎略复杂。

### **玩家适配方案**

广大玩家的适配目的很明确，目的就是要确保控件在不同屏幕的相对大小一致，看起来一毛一样的。以一位大神玩家的两种适配方案为例：

- 方案一。编写脚本将长度转换成各分辨率下的长度，缺点是难以覆盖市面上的所有分辨率。
- 方案二。AutoLayout支持库（安卓中**鸿洋**大神的开源）。该库的想法非常好：对照设计图，使用px编写布局，不影响预览；绘制阶段将对应设计图的px数值计算转换为当前屏幕下适配的大小；为简化接入，inflate时自动将各Layout转换为对应的AutoLayout，从而不需要在所有的xml中更改。但是同时该库也存在以下等问题：


- 扩展性较差。对于每一种ViewGroup都要对应编写对应的AutoLayout进行扩展，对于各View的每个需要适配的属性都要编写代码进行适配扩展；
- 在onMeasure阶段进行数值计算。这对于非LayoutParams中的属性存在较多不合理之处。比如在onMeasure时对TextView的textSize进行换算并setTextSize，那么玩家在代码中动态设置的textSize都会失效，因为在每次onMesasure时都会重新被AutoLayout重新设置覆盖。
- issue较多并且作者已**不再维护。**

2

### 进行一下探索和尝试如何？

个人觉得AutoLayout的设计思想非常优秀，但是将LayoutParams与属性作为切入口在mesure过程中进行转换计算的方案存在效率与扩展性等方面的问题。那么Android计算长度的收口在哪里，能不能在Android计算长度时进行换算呢？如果能在Android计算长度时进行换算，那么就不需要一系列多余的计算以及适配，一切问题就都迎刃而解了。

经过一番寻觅，发现系统进行长度计算的收口为TypedValue中的applyDimension函数，传入单位与value将其计算为对应的px数值。

```java
public static float applyDimension(int unit, float value,DisplayMetrics metrics){
      switch (unit) {
          case COMPLEX_UNIT_PX:
          return value;
          case COMPLEX_UNIT_DIP:
          return value * metrics.density;
          case COMPLEX_UNIT_SP:
          return value * metrics.scaledDensity;
          case COMPLEX_UNIT_PT:
          return value * metrics.xdpi * (1.0f/72);
          case COMPLEX_UNIT_IN:
          return value * metrics.xdpi;
          case COMPLEX_UNIT_MM:
          return value * metrics.xdpi * (1.0f/25.4f);
      }
	return 0;
}

```



- 可以看见换算方法非常简单，而DisplayMetrics的所有属性都是public的，不用反射就能修改；
- pt的原意是长度单位磅，根据当前屏幕与设计图尺寸将metrics.xdpi进行修改就可以实现将pt这个单位重定义成我们所需要的相对长度单位，使修改之后计算出的1pt实际对应的px/屏幕宽度px=1px/设计图宽度px。
- 而这个DisplayMetrics从哪来？从源码中可以看出一般为mContext.getResources().getDisplayMetrics()，这个mContext即为所在Activity；
- Activity中所拿到的DisplayMetrics与Application中拿到的DisplayMetrics虽然不是一个实例，但是所有数值都相同，在Application中进行更改也会影响到所有Activity中；
- Configuration的变化会导致DisplayMetrics的重新计算还原，需要handle；
- px,dp与sp都是平时常用的单位，而pt,in与mm几乎没有看见过，从这些不常见的单位下手正好可以不影响其他常用的单位。

基于以上几点，便有了以下方案。

3

## 方案 （**注意划重点了**）

适配的目标是：完全按照设计图上标注的尺寸来编写页面，所编写的页面在所有大小与分辨率的屏幕上都表现一致，即控件在所有屏幕上相对于整个屏幕的相对大小都一致（看起来只是将设计图缩放至屏幕大小）。

- 核心。使用冷门的pt作为长度单位。
- 绘制。编写xml时完全对照设计稿上的尺寸来编写，只不过单位换为pt。


- 需要在代码中动态转换成px时使用TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_PT, value, metrics)。

  *UI给我们提供的设计图是这样的*  
  
  ![img](https://github.com/feisher/feisher.github.io/blob/master/UI.jpg)  
  
  **创建什么样的预览使用的设备以设计图为准**  
  
  - 预览。实时预览时绘制页面是很重要的一个环节。以1334x750的设计图为例，为了实现于正常绘制时一样的预览功能，创建一个长为1334磅，宽为750磅的设备作为预览，经换算约为21.5英寸((sqrt(1334^2+750^2))/72)。**此处直接按照iphone 6尺寸设置4.7也没影响**。预览时选择这个设备即可。

![img](https://github.com/feisher/feisher.github.io/blob/master/RomSetting.jpg）
- 怎么创建那个750设计稿分辨率的设备呐？看图，_知道的同学请跳过_
![img](https://github.com/feisher/feisher.github.io/blob/master/creatRom.png)

- 代码处理。在Application的onCreate中与onConfigurationChanged中更改DisplayMetrics（其中DESIGN_WIDTH是绘制页面时参照的设计图宽度）：

  ```java
  /**  将此文件直接复制到项目中，不要忘记清单文件配置Application，另 布局中使用pt
  *	（例如： android:layout_height="300pt" 用错可不适配哦！）
  *	feisher  @2017年8月11日14:52:27 二次整理，
  *	458079442@qq.com
  */
  public class MyApplication extends Application{

  	public final static float DESIGN_WIDTH = 750; //绘制页面时参照的设计图宽度

  	@Override
  	public void onCreate() {
  		super.onCreate();
  		resetDensity();//注意不要漏掉
  	}

  	@Override
  	public void onConfigurationChanged(Configuration newConfig) {
          super.onConfigurationChanged(newConfig);
          resetDensity();//这个方法重写也是很有必要的
  	}

      public void resetDensity(){
          Point size = new Point();
          ((WindowManager)getSystemService(WINDOW_SERVICE)).getDefaultDisplay().getSize(size);
          getResources().getDisplayMetrics().xdpi = size.x/DESIGN_WIDTH*72f;
      }
  }

  ```


这样绘制出来的页面就跟设计图几乎完全一样，无论大小屏上看起来就只是将设计图缩放之后的结果。

适配前：

![img](https://github.com/feisher/feisher.github.io/blob/master/old.jpg)

适配后：

![img](https://github.com/feisher/feisher.github.io/blob/master/result.jpg)

*ps:引用自新浪微博  布隆  博客，感谢辛勤的开创者*

再次感谢辛勤的各位开发者
