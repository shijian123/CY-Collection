

# Core Animation

 众所周知，绚丽动画效果是iOS系统的一大特点，通过UIView层封装的动画，基本可以满足我们应用开发的所有需求，但若需要更加自由的控制动画的展示，我们就需要使用CoreAnimation框架中的一些类与方法

# Core Animation基础知识

Core Animation是iOS和OS X上图形渲染和动画的基础结构，可用于为视图和应用程序的其他可视元素设置动画。Core Animation的实现逻辑是将大部分实际绘图工作交给专用图形硬件加速渲染，以实现高帧率和流畅的动画，而不会给CPU带来负担并降低应用程序的速度。

下图描述了CoreAnimation与UIKit框架的关系
![核心动画架构图](https://user-gold-cdn.xitu.io/2019/5/8/16a9547cdd4d8f10?w=552&h=414&f=png&s=25375)	
Core Animation开发动画的本质就是将CALayer中的内容转化为位图从而供硬件操作，所以想熟练掌握动画操作必须了解CALayer

# CALayer

CALayer跟UIView概念上很相似，同样都是被层级管理树管理的一些矩形块，同样可以包含内容，管理子图层，可以做动画和变换。但是最大的不同是UIView可以处理用户的交互，而CALayer是不能够响应事件的，即使它提供了一些判断触点是否在图层范围内的方法。每一个UIView视图内部都封装了一个CALayer图层，我们通过UIView的layer属性访问这个图层。其实对于UIView来说负责内容展示的就是它内部的CALayer，UIView只不过是将自身的展示任务交给了内部的CALayer完成，而它还肩负着一些其它的任务，比如说用户的交互响应，提供一些Core Animation底层方法的高级接口等。

那为什么不把这些任务放在一个类中处理而是把他们作为平行关系同时存在呢？很重要的原因是要将职责分离，这样可以避免很多重复的代码，由于iOS平台和MacOS平台上用户的交互方式有着本质的不同，在iOS系统中我们使用的是UIKit和UIView，而在MacOS系统中我们使用的是AppKit和NSView，所以在这种情况下将展示部分分离出来会给苹果的多平台系统开发带来便捷。


```
@interface CALayer : NSObject <NSSecureCoding, CAMediaTiming>
{
@private
  struct _CALayerIvars {
    int32_t refcount;
    uint32_t magic;
    void *layer;
#if TARGET_OS_MAC && !TARGET_RT_64_BIT
    void * _Nonnull unused1[8];
#endif
  } _attr;
}
```


**CALayer属性**：CALayer层的主要工作是管理您提供的视觉内容，图层本身具有可设置的可视属性，例如背景颜色，边框和阴影。除了管理视觉内容之外，还保留有关其内容的几何形状的信息（例如其位置，大小和变换），用于在屏幕上呈现该内容。

![CAlayer属性图](https://user-gold-cdn.xitu.io/2019/5/8/16a9547cc92ca1d2?w=783&h=861&f=png&s=161555)

# Core Animation

接下来我们将讲解下Core Animation的`CAAnimation、CAPropertyAnimation、CABasicAnimation、CAKeyframeAnimation、CASpringAnimation、CATransition、CAAnimationGroup`及他们之间的关系，其间也穿插了`CALayer动画运行的原理、Animation-KeyPath值、CATransaction事务类、检测动画的结束、暂停和恢复图层的动画`等内容
![Core Animation类图](https://user-gold-cdn.xitu.io/2019/5/8/16a9547cc939826c?w=1000&h=422&f=png&s=75854)

## CAAnimation
CAAnimation是核心动画的基类，不能直接使用，主要负责动画的时间、速度等，本身实现了CAMediaTiming协议。

```
@interface CAAnimation : NSObject
    <NSSecureCoding, NSCopying, CAMediaTiming, CAAction>
{
@private
  void *_attr;
  uint32_t _flags;
}
```


CAAnimation属性	|	说明
---------------	| ---------------
timingFunction	|	CAMediaTimingFunction速度控制函数，控制动画运行的节奏
removedOnCompletion	|	默认为YES，代表动画执行完毕后就从图层上移除，图形会恢复到动画执行前的状态。如果想让图层保持显示动画执行后的状态，那就设置为NO，不过还要设置fillMode为kCAFillModeForwards
delegate			|	代理（animationDidStart、animationDidStop）

ps：**CAMediaTimingFunction介绍**

```
kCAMediaTimingFunctionLinear（线性）：匀速，给你一个相对静态的感觉
kCAMediaTimingFunctionEaseIn（渐进）：动画缓慢进入，然后加速离开
kCAMediaTimingFunctionEaseOut（渐出）：动画全速进入，然后减速的到达目的地
kCAMediaTimingFunctionEaseInEaseOut（渐进渐出）：动画缓慢的进入，中间加速，然后减速的到达目的地。这个是默认的动画行为。
```

### CAMediaTiming协议
像`duration，beginTime、repeatCount、speed、timeOffset、repeatDuration、autoreverses`这些时间相关的属性都在这个类中。协议中的这些属性通过一些方式结合在一起，准确的控制着时间。


CAMediaTiming属性	|	说明
---------------	| ---------------
beginTime			|	指定动画开始的时间。从开始延迟几秒的话，设置为【CACurrentMediaTime() + 秒数】 的方式
duration			|	动画的时长
speed				|	动画运行速度（如果把动画的duration设置为3秒，而speed设置为2，动画将会在1.5秒结束，因为它以两倍速在执行）
timeOffset		|	结合一个暂停动画（speed=0）一起使用来控制动画的“当前时间”。暂停的动画将会在第一帧卡住，然后通过改变timeOffset来随意控制动画进程
repeatCount		|	重复的次数。不停重复设置为 HUGE_VALF
repeatDuration	|	设置动画的时间。在该时间内动画一直执行，不计次数。
autoreverses		|	动画结束时是否执行逆动画，如果duration为1s，则完成一次autoreverse就需要2s。
fillMode			|	CAMediaTimingFillMode枚举

ps：**CAMediaTimingFillMode介绍**

```
kCAFillModeRemoved：这个是默认值，也就是说当动画开始前和动画结束后，动画对layer都没有影响，动画结束后，layer会恢复到之前的状态
kCAFillModeForwards：当动画结束后，layer会一直保持着toValue的状态
kCAFillModeBackwards：如果要让动画在开始之前（延迟的这段时间内）显示fromValue的状态
kCAFillModeBoth：这个其实就是上面两个的合成.动画加入后开始之前，layer便处于动画初始状态，动画结束后layer保持动画最后的状态

注意必须配合animation.removeOnCompletion = NO才能达到以上效果

```


## CAPropertyAnimation	
继承自CAAnimation，不能直接使用，要想创建动画对象，应该使用它的两个子类：CABasicAnimation和CAKeyframeAnimation

```
You do not create instances of CAPropertyAnimation: to animate the properties of a Core Animation layer, create instance of the concrete subclasses CABasicAnimation or CAKeyframeAnimation.
```	

CAPropertyAnimation属性 |	说明
---------------	| ---------------
keyPath			|	通过指定CALayer的一个属性名称为keyPath（NSString类型），并且对CALayer的这个属性的值进行修改，达到相应的动画效果。比如，指定@“position”为keyPath，就修改CALayer的position属性的值，以达到平移的动画效果
			
					


## CABasicAnimation 

CABasicAnimation是核心动画类簇中的一个类，其父类是CAPropertyAnimation，其子类是CASpringAnimation，它的祖父是CAAnimation。它主要用于制作比较单一的动画，例如，平移、缩放、旋转、颜色渐变、边框的值的变化等，也就是将layer的某个属性值从一个值到另一个值的变化


CABasicAnimation属性 |	说明
---------------	| ---------------
fromValue			|	所改变属性的起始值
toValue			|	所改变属性的结束时的值
byValue			|	所改变属性相同起始值的改变量

代码如下

```
        let baseAnim = CABasicAnimation(keyPath: "position")
        baseAnim.duration = 2;
        //开始的位置
        baseAnim.fromValue = NSValue(cgPoint: (imgView?.layer.position)!)
        baseAnim.toValue = NSValue(cgPoint: CGPoint(x: 260, y: 260))
//        baseAnim.isRemovedOnCompletion = false
//        baseAnim.fillMode = CAMediaTimingFillMode.forwards
        imgView?.layer.add(baseAnim, forKey: "baseAnim-position")
        imgView?.center = CGPoint(x: 260, y: 260)

```

### 防止动画结束后回到初始状态

如上面代码所示，需要添加`imgView?.center = CGPoint(x: 260, y: 260)`来防止防止动画结束后回到初始状态，网上还有另外一种方法是
设置removedOnCompletion、fillMode两个属性

```
baseAnim.removedOnCompletion = NO;
baseAnim.fillMode = kCAFillModeForwards;
```
但是这种方法会造成modelLayer没有修改，_view1的实际坐标点并没有在所看到的位置，会产生一些问题

### CALayer动画运行的原理

CALayer有两个实例方法presentationLayer（简称P）和modelLayer（简称M），

```
/* presentationLayer
 * 返回一个layer的拷贝，如果有任何活动动画时，包含当前状态的所有layer属性
 * 实际上是逼近当前状态的近似值。
 * 尝试以任何方式修改返回的结果都是未定义的。
 * 返回值的sublayers 、mask、superlayer是当前layer的这些属性的presentationLayer
 */

- (nullable instancetype)presentationLayer;

/* modelLayer
 * 对presentationLayer调用，返回当前模型值。
 * 对非presentationLayer调用，返回本身。
 * 在生成表示层的事务完成后调用此方法的结果未定义。
 */

- (instancetype)modelLayer;
```
从中可以看到P即是我们看到的屏幕上展示的状态，而M就是我们设置完立即生效的真实状态；打一个比方的话，P是个瞎子，只负责走路（绘制内容），而M是个瘸子，只负责看路（如何绘制）

**CALayer动画运行的原理**：P会在每次屏幕刷新时更新状态，当有动画CAAnimation(简称A)加入时，P由动画A控制进行绘制，当动画A结束被移除时P则再去取M的状态展示。但是由于M没有变化，所以动画执行结束又会回到起点。如果想要P在动画结束后就停在当前状态而不回到M的状态，我们就需要给A设置两个属性，一个是`A.removedOnCompletion = NO`;表示动画结束后A依然影响着P，另一个是`A.fillMode = kCAFillModeForwards`，这两行代码将会让A控制住P在动画结束后保持不变，但是此时我们的P和M不同步，我们看到的P是toValue的状态，而M则还是自己原来的状态。举个栗子，我们初始化一个view，它的状态为1，我们给它的layer加个动画，from是0，to是2，设置`fillMode为kCAFillModeForewards`，则动画结束后P的状态是2，M的状态是1，这可能会导致一些问题出现。比如你点P所在的位置点不动，因为响应点击的是M。所以我们应该让P和M同步，如上代码`imgView?.center = CGPoint(x: 260, y: 260)`需要提一点的是：对M赋值，不会影响P的显示，当P想要显示的时候，它已经被A控制了,并不会先闪现一下。


### Animation-KeyPath值
上面的动画的KeyPath值我们只使用了position，其实还有很多类型可以设置，下面我们列出了一些比较常用的

keyPath值		|	说明				|	值类型
-------------	| -----------------	| ------------
position		| 移动位置				| CGPoint
opacity		| 透明度				| 0-1
bounds			| 变大与位置			| CGRect
bounds.size	| 由小变大				| CGSize
backgroundColor	| 背景颜色			| CGColor
cornerRadius	| 渐变圆角				| 任意数值
borderWidth	| 改变边框border的大小（(图形周围边框，border默认为黑色)）	| 任意数值
contents		| 改变layer内容(图片)注意如果想要达到改变内容的动画效果；首先在运行动画之前定义好layer的contents contents| CGImage
transform.scale| 缩放、放大				| 0.0-1.0
transform.rotation.x| 旋转动画(翻转，沿着X轴)| M_PI*n
transform.rotation.Y| 旋转动画(翻转，沿着Y轴)| M_PI*n
transform.rotation.Z| 旋转动画(翻转，沿着Z轴)| M_PI*n
transform.translation.x | 旋转动画(翻转，沿着X轴)| 任意数值
transform.translation.y | 旋转动画(翻转，沿着Y轴)| 任意数值



## CAKeyframeAnimation 


CABasicAnimation是将属性从起始值更改为结束值，而CAKeyframeAnimation对象是允许你以线性或非线性的方式设置一组目标值的动画。关键帧动画由一组目标数据值和每个值到达的时间组成。不但可以简单的只指定值数组和时间数组，还可以按照路径进行更改图层的位置。动画对象采用您指定的关键帧，并通过在给定时间段内从一个值插值到下一个值来构建动画。

CAKeyframeAnimation属性 |	说明
---------------	| ---------------
values				|	关键帧值表示动画必须执行的值，此属性中的值仅在path属性的值为nil时才使用。根据属性的类型，您可能需要用NSValue对象的NSNumber包装这个数组中的值。对于一些核心图形数据类型，您可能还需要将它们转换为id，然后再将它们添加到数组中。将给定的关键帧值应用于该层的时间取决于动画时间，由calculationMode、keyTimes和timingFunctions属性控制。关键帧之间的值是使用插值创建的，除非将计算模式设置为kcaanimation离散
path				|	基于点的属性的路径，对于包含CGPoint数据类型的层属性，您分配给该属性的路径对象定义了该属性在动画长度上的值。如果指定此属性的值，则忽略值属性中的任何数据
keyTimes			|	keyTimes的值与values中的值一一对应指定关键帧在动画中的时间点，取值范围为[0,1]。当keyTimes没有设置的时候,各个关键帧的时间是平分的
timingFunctions	|	一个可选的CAMediaTimingFunction对象数组，指定每个关键帧之间的动画缓冲效果
calculationMode	|	关键帧间插值计算模式
rotationMode		|	定义沿路径动画的对象是否旋转以匹配路径切线


ps：

**timingFunctions**:动画缓冲效果

```
kCAMediaTimingFunctionLinear：线性起搏，使动画在其持续时间内均匀地发生
kCAMediaTimingFunctionEaseIn：使一个动画开始缓慢，然后加速，随着它的进程
kCAMediaTimingFunctionEaseOut：使动画快速开始，然后缓慢地进行
kCAMediaTimingFunctionEaseInEaseOut：使动画开始缓慢，在其持续时间的中间加速，然后在完成之前再放慢速度
kCAMediaTimingFunctionDefault：默认，确保动画的时间与大多数系统动画的匹配
```

**calculationMode**：动画计算方式

```
kCAAnimationLinear：默认差值
kCAAnimationDiscrete：逐帧显示
kCAAnimationPaced：匀速 无视keyTimes和timingFunctions设置
kCAAnimationCubic：keyValue之间曲线平滑 可用 tensionValues,continuityValues,biasValues 调整
kCAAnimationCubicPaced：keyValue之间平滑差值 无视keyTimes

```

**rotationMode**：旋转方式

```
kCAAnimationRotateAuto：自动
kCAAnimationRotateAutoReverse：自动翻转 不设置则不旋转
```

代码1、用values属性

```
        //创建动画对象
        let keyAnim = CAKeyframeAnimation(keyPath: "position")
        //设置values
        keyAnim.values = [NSValue(cgPoint: CGPoint(x: 100, y: 200)),
                          NSValue(cgPoint: CGPoint(x: 200, y: 200)),
                          NSValue(cgPoint: CGPoint(x: 200, y: 300)),
                          NSValue(cgPoint: CGPoint(x: 100, y: 300)),
                          NSValue(cgPoint: CGPoint(x: 100, y: 400)),
                          NSValue(cgPoint: CGPoint(x: 200, y: 500))]
        //重复次数 默认为1
        keyAnim.repeatCount = MAXFLOAT
        //设置是否原路返回 默认为false
        keyAnim.autoreverses = true
        //设置移动速度，越小越快
        keyAnim.duration = 4.0
        
        keyAnim.isRemovedOnCompletion = false
        keyAnim.fillMode = .forwards
        
        keyAnim.timingFunctions = [CAMediaTimingFunction(name: .easeInEaseOut)]
        
        imgView?.layer.add(keyAnim, forKey: "keyAnim-Values")
```


代码2、用path属性

```
        //创建动画对象
        let keyAnim = CAKeyframeAnimation(keyPath: "position")
        
        //创建一个CGPathRef对象，就是动画的路线
        let path = CGMutablePath()
        //自动沿着弧度移动
        path.addEllipse(in: CGRect(x: 150, y: 200, width: 200, height: 100))
        //设置开始位置
        path.move(to: CGPoint(x: 100, y: 100))
        //沿着直线移动
        path.addLine(to: CGPoint(x: 200, y: 100))
        path.addLine(to: CGPoint(x: 200, y: 200))
        path.addLine(to: CGPoint(x: 100, y: 200))
        path.addLine(to: CGPoint(x: 100, y: 300))
        path.addLine(to: CGPoint(x: 200, y: 400))
        
        //沿着曲线移动
        path.addCurve(to: CGPoint(x: 50.0, y: 275.0), control1: CGPoint(x: 150.0, y: 275.0), control2: CGPoint(x: 70.0, y: 120.0))
        path.addCurve(to: CGPoint(x: 150.0, y: 275.0), control1: CGPoint(x: 250.0, y: 275.0), control2: CGPoint(x: 90.0, y: 120.0))
        path.addCurve(to: CGPoint(x: 250.0, y: 275.0), control1: CGPoint(x: 350.0, y: 275.0), control2: CGPoint(x: 110, y: 120.0))
        path.addCurve(to: CGPoint(x: 350.0, y: 275.0), control1: CGPoint(x: 450.0, y: 275.0), control2: CGPoint(x: 130, y: 120.0))

        keyAnim.path = path
        //重复次数 默认为1
        keyAnim.repeatCount = MAXFLOAT
        //设置是否原路返回 默认为false
        keyAnim.autoreverses = true
        //设置移动速度，越小越快
        keyAnim.duration = 4.0

        keyAnim.isRemovedOnCompletion = false
        keyAnim.fillMode = .forwards
        
        keyAnim.timingFunctions = [CAMediaTimingFunction(name: .easeInEaseOut)]
        
        imgView?.layer.add(keyAnim, forKey: "keyAnim-Path")
        
```


## CASpringAnimation
iOS9新出的CASpringAnimation继承于CABaseAnimation，是苹果专门解决开发者关于弹簧动画的这个需求而封装的类


CASpringAnimation属性	|	说明
----------------------	| ---------------
mass						|	质量，影响图层运动时的弹簧惯性，质量越大，弹簧拉伸和压缩的幅度越大，默认值：1
stiffness					|	刚度系数(劲度系数/弹性系数)，刚度系数越大，形变产生的力就越大，运动越快。默认值： 100
damping					|	阻尼系数，阻止弹簧伸缩的系数，阻尼系数越大，停止越快。默认值：10；
initialVelocity			|	初始速率，动画视图的初始速度大小。默认值：0；速率为正数时，速度方向与运动方向一致，速率为负数时，速度方向与运动方向相反；
settlingDuration			|	估算时间 返回弹簧动画到停止时的估算时间，根据当前的动画参数估算；


代码如下

```
        let springAnim = CASpringAnimation(keyPath: "bounds")

        springAnim.toValue = NSValue(cgRect:CGRect(x: 200, y: 230, width: 140, height: 140))
        
        //mass模拟的是质量，影响图层运动时的弹簧惯性，质量越大，弹簧拉伸和压缩的幅度越大 默认值：1
        springAnim.mass = 5
        //stiffness刚度系数(劲度系数/弹性系数)，刚度系数越大，形变产生的力就越大，运动越快。默认值： 100
        springAnim.stiffness = 80
        //damping阻尼系数，阻止弹簧伸缩的系数，阻尼系数越大，停止越快。默认值：10；
        springAnim.damping = 8
    //initialVelocity初始速率，动画视图的初始速度大小。默认值：0；速率为正数时，速度方向与运动方向一致，速率为负数时，速度方向与运动方向相反；
        springAnim.initialVelocity = 10
        //估算时间 返回弹簧动画到停止时的估算时间，根据当前的动画参数估算；
        springAnim.duration = springAnim.settlingDuration
        
        imgView?.layer.add(springAnim, forKey: "springAnim")
```


## CATransition

CATransition是CAAnimation的子类，用于做转场动画，能够为图层提供移出屏幕和移入屏幕的动画效果。

CATransition属性			|	说明
----------------------	| ---------------
type						|	CATransitionType，动画过渡类型
subtype					|	CATransitionSubtype，动画移动方向
startProgress				|	动画起点(在整体动画的百分比)
endProgress				|	动画终点(在整体动画的百分比)

ps:如果不需要动画执行整个过程(动画执行到中间部分就停止)，可以指定startProgress，endProgress属性。

**CATransitionType**：动画过渡类型

```
kCATransitionFade：渐变
kCATransitionMoveIn：覆盖
kCATransitionPush：推出
kCATransitionReveal：揭开
```

除此之外，还支持以下私有方法

```
cube：立方体旋转
suckEffect：吮吸
oglFlip：翻转
rippleEffect：水波动画
pageCurl：翻页
pageUnCurl：反翻页
cemeraIrisHollowOpen：开镜头
cameraIrisHollowClose：关镜头

```

**CATransitionSubtype**：控制动画方向，上/下/左/右 4个不同方向的动画

```
kCATransitionFromRight
kCATransitionFromLeft
kCATransitionFromTop
kCATransitionFromBottom
```





## CAAnimationGroup

单一的动画并不能满足某些特定需求，这时就需要用到CAAnimationGroup，其是CAAnimation的子类，默认情况下，一组动画对象是同时运行的，也可以通过设置动画对象的beginTime属性来更改动画的时间

CATransition属性			|	说明
----------------------	| ---------------
animations				|	[CAAnimation]，动画组


代码如下

```
        let groupAnim = CAAnimationGroup()
        
        //创建keyAnim
        let keyAnim = CAKeyframeAnimation(keyPath: "position")
        //设置values
        keyAnim.values = [NSValue(cgPoint: CGPoint(x: 100, y: 200)),
                          NSValue(cgPoint: CGPoint(x: 200, y: 200)),
                          NSValue(cgPoint: CGPoint(x: 200, y: 300)),
                          NSValue(cgPoint: CGPoint(x: 100, y: 300)),
                          NSValue(cgPoint: CGPoint(x: 100, y: 400)),
                          NSValue(cgPoint: CGPoint(x: 200, y: 500))]
        keyAnim.duration = 4.0
        
        keyAnim.timingFunctions = [CAMediaTimingFunction(name: .easeInEaseOut)]
        
        //创建渐变圆角
        let animation = CABasicAnimation(keyPath: "cornerRadius")
        animation.toValue = 40
        animation.duration = 4.0
        imgView?.layer.masksToBounds = true
        
        groupAnim.animations = [keyAnim, animation]
        groupAnim.duration = 4.0
        groupAnim.repeatCount = MAXFLOAT
        groupAnim.autoreverses = true

        imgView?.layer.add(groupAnim, forKey: "groupAnim")
        
```

将动画分组在一起的更高级方法是使用事务对象。通过允许您创建嵌套的动画集并为每个动画分配不同的动画参数，事务提供了更大的灵活性。

## CATransaction事务类
CATransaction事务类可以对多个layer的属性同时进行修改，它分隐式事务,和显式事务。
当我们向图层添加显式或隐式动画时，Core Animation都会自动创建隐式事务。但是，我们还可以创建显式事务以更精确地管理这些动画。

* 区分隐式动画和隐式事务：隐式动画通过隐式事务实现动画 。
* 区分显式动画和显式事务：显式动画有多种实现方式，显式事务是一种实现显式动画的方式。
* 除显式事务外,任何对于CALayer属性的修改,都是隐式事务.

**隐式事务**

```
//创建layer
let layer = CALayer()
layer.bounds = CGRect(x: 0, y: 0, width: 100, height: 100)
layer.position = CGPoint(x: 100, y: 350)
layer.backgroundColor = UIColor.red.cgColor
layer.borderColor = UIColor.black.cgColor
layer.opacity = 1.0
view.layer.addSublayer(layer)

//触发动画

// 设置变化动画过程是否显示，默认为true不显示
CATransaction.setDisableActions(false)
layer.cornerRadius = (layer.cornerRadius == 0.0) ? 30.0 : 0.0
layer.opacity = (layer.opacity == 1.0) ? 0.5 : 1.0


```


**显式事务**：通过明确的调用begin，commit来提交动画

```
CATransaction.begin()
layer.zPosition = 200.0
layer.opacity = 0.0
CATransaction.commit()
```

**使用事务的主要原因之一**是在显式事务的范围内，我们可以更改持续时间，计时功能和其他参数。还可以为整个事务分配完成块，以便在动画组完成时通知应用。

例如，将动画的默认持续时间更改为8秒，使用```setValue:forKey:```方法进行修改，目前支持的属性包括： `"animationDuration", "animationTimingFunction","completionBlock", "disableActions"`.

```
CATransaction.begin()
CATransaction.setValue(8.0, forKey: "animationDuration")
//执行动画
CATransaction.commit()
```

**嵌套事务**:
当我们要为不同动画集提供不同默认值的情况下可以使用**嵌套事务**。要将一个事务嵌套在另一个事务中，只需再次调用begin，且每个begin调用必须一一对应一个commit方法。只有在为最外层事务提交更改后，Core Animation才会开始关联的动画。


嵌套显式事务代码

```
//事务嵌套
CATransaction.begin()   // 外部transaction
CATransaction.setValue(2.0, forKey: "animationDuration")
layer.position = CGPoint(x: 140, y: 140)
        
CATransaction.begin()   // 内部transaction
CATransaction.setValue(5.0, forKey: "animationDuration")
layer.zPosition = 200.0
layer.opacity = 0.0
        
CATransaction.commit()  // 内部transaction
CATransaction.commit()  // 外部transaction

```


## 检测动画的结束

核心动画支持检测动画开始或结束的时间。这些通知是进行与动画相关的任何内务处理任务的好时机。例如，您可以使用开始通知来设置一些相关的状态信息，并使用相应的结束通知来拆除该状态。

有两种不同的方式可以通知动画的状态：

* 使用```setCompletionBlock:```方法将完成块添加到当前事务。当事务中的所有动画完成后，事务将执行完成块。
* 将委托分配给CAAnimation对象并实现```animationDidStart:```和```animationDidStop:finished:```委托方法。

如果要让两个动画链接在一起，以便在另一个完成时启动，请不要使用动画通知。而是使用动画对象的`beginTime`属性按照所需的时间启动每个动画对象。将两个动画链接在一起，只需将第二个动画的开始时间设置为第一个动画的结束时间。

每个图层都有自己的本地时间，用于管理动画计时。通常，两个不同层的本地时间足够接近，您可以为每个层指定相同的时间值，用户可能不会注意到任何内容。但是由于superLayer或其本身Layer的时序参数设置，层的本地时间会发生变化。例如，更改Layer的`speed`属性会导致该Layer（及其子Layer）上的动画持续时间按比例更改。

为了确保Layer的时间值合适，CALayer类定义了`convertTime:fromLayer:`和`convertTime:toLayer:`方法。我们可以使用这些方法将固定时间值转换为Layer的本地时间或将时间值从一个Layer转换为另一个Layer。这些方法可能影响图层本地时间的媒体计时属性，并返回可与其他图层一起使用的值。

可使用下面示例来获取图层的当前本地时间。CACurrentMediaTime函数返回计算机的当前时钟时间，该方法将本机时间并转换为图层的本地时间。

获取图层的当前本地时间

```
CFTimeInterval localLayerTime = [myLayer convertTime：CACurrentMediaTime（）fromLayer：nil];
```

在图层的本地时间中有时间值后，可以使用该值更新动画对象或图层的与时序相关的属性。使用这些计时属性，您可以实现一些有趣的动画行为，包括：

* **beginTime**属性设置动画的开始时间。通常动画开始下一个周期的时候，我们可以使用beginTime将动画开始时间延迟几秒钟。将两个动画链接在一起的方法是将一个动画的开始时间设置为与另一个动画的结束时间相匹配。如果延迟动画的开始，则可能还需要将fillMode属性设置为kCAFillModeBackwards。即使图层树中的图层对象包含不同的值，此填充模式也会使图层显示动画的起始值。如果没有此填充模式，您将看到在动画开始执行之前跳转到最终值。其他填充模式也可用。

* **autoreverses**属性使动画在指定时间内执行，然后返回到动画的起始值。我们可以将autoreverses与repeatCount组合使用，就可以起始值和结束值之间来回动画。将重复计数设置为自动回转动画的整数（例如1.0）会导致动画停止在其起始值上。添加额外的半步（例如重复计数为1.5）会导致动画停止在其结束值上。
使用timeOffset具有组动画的属性可以在稍后的时间启动某些动画。




## 暂停和恢复图层的动画

```
/**
 layer 暂停动画
 */
+ (void)pauseLayer:(CALayer*)layer {
    CFTimeInterval pausedTime = [layer convertTime:CACurrentMediaTime() fromLayer:nil];
    layer.speed = 0.0;
    layer.timeOffset = pausedTime;
}

/**
 layer 继续动画
 */
+ (void)resumeLayer:(CALayer*)layer {
    CFTimeInterval pausedTime = [layer timeOffset];
    layer.speed = 1.0;
    layer.timeOffset = 0.0;
    layer.beginTime = 0.0;
    CFTimeInterval timeSincePause = [layer convertTime:CACurrentMediaTime() fromLayer:nil] - pausedTime;
    layer.beginTime = timeSincePause;
}
```


# 代码及参考资料

[github源码地址](https://github.com/shijian123/CAAnimationDome)


参考资料：

[核心动画-官方文档](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/CoreAnimation_guide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40004514)

[iOS CALayer图层漫谈（一）](https://www.jianshu.com/p/c6924e2ab232)

[Core Animation系列之CATransaction](https://www.jianshu.com/p/9f0667c883f8)

[iOS动画篇_CoreAnimation(超详细解析核心动画)](http://www.cocoachina.com/ios/20170623/19612.html)

[iOS动画，原理与实操](https://www.jianshu.com/p/9fa025c42261)

[iOS动画全面解析](https://juejin.im/post/5bd140abf265da0ae6778180#heading-11)

[iOS CoreAnimation专题——原理篇（三） CALayer的模型层与展示层](https://blog.csdn.net/u013282174/article/details/50388546)