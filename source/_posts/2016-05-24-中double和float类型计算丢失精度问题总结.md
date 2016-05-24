title: java 中double和float类型计算丢失精度问题总结
categories: Java 
tags: Java
keywords: 微信支付之扫码支付
date: 2016-05-24 17:35:11
---

## 诶，我怎么不能全额退款了？
> 问题发生在某天中午，当我订单付完款后，不想要了就点击了全额退款，但是给我的提示确实 “您输入的金额不正确”，我就纳闷了，为什么不能退?按了下F12看了下请求返回的信息，看了下代码，然后就发现了问题...



## 1、bigdecimal 转换成小数计算有误差
- *真实项目中校验退款金额是否超过订单实付款金额代码如下截图：*

![错误代码](http://o6skwsce0.bkt.clouddn.com/aaa.png)

- *模拟以上的代码截图如下：*

![模拟代码](http://o6skwsce0.bkt.clouddn.com/bbb.png)

- *float和double做四则运算误差：*
```
    public static void main( String[] args ) {
    	 System.out.println(0.05+0.01);
         System.out.println(1.0-0.42);
         System.out.println(4.015*100);
         System.out.println(123.3/100);
    }
    
    //输出
    0.060000000000000005
    0.5800000000000001
    401.49999999999994
    1.2329999999999999

```

<!--more-->

- *问题的根源：*
为什么会出现精度丢失，得从计算机说起，计算机并不能识别除了二进制数据以外的任何数据。无论我们使用何种编程语言，在何种编译环境下工作，都要先把源程序翻译成二进制的机器码后才能被计算机识别。以上面提到的情况为例，我们源程序里的2.4是十进制的，计算机不能直接识别，要先编译成二进制。但问题来了，2.4的二进制表示并非是精确的2.4，反而最为接近的二进制表示是2.3999999999999999。原因在于浮点数由两部分组成：指数和尾数，这点如果知道怎样进行浮点数的二进制与十进制转换，应该是不难理解的。如果在这个转换的过程中，浮点数参与了计算，那么转换的过程就会变得不可预 知，并且变得不可逆。我们有理由相信，就是在这个过程中，发生了精度的丢失。而至于为什么有些浮点计算会得到准确的结果，应该也是碰巧那个计算的二进制与 十进制之间能够准确转换。而当输出单个浮点型数据的时候，可以正确输出，如double d = 2.4;System.out.println(d);
输出的是2.4，而不是2.3999999999999999。也就是说，不进行浮点计算的时候，在十进制里浮点数能正确显示。这更印证了我以上的想法，即如果浮点数参与了计算，那么浮点数二进制与十进制间的转换过程就会变得不可预知，并且变得不可逆。

  事实上，浮点数并不适合用于精确计算，而适合进行科学计算。这里有一个小知识：既然float和double型用来表示带有小数点的数，那为什么我们不称 它们为“小数”或者“实数”，要叫浮点数呢？因为这些数都以科学计数法的形式存储。当一个数如50.534，转换成科学计数法的形式为5.053e1，它 的小数点移动到了一个新的位置（即浮动了）。可见，浮点数本来就是用于科学计算的，用来进行精确计算实在太不合适了。

## 2、bigdecimal构造函数使用不当时，结果仍然异常
```
	public static void main(String[] args) {
		
		BigDecimal aa = new BigDecimal(0.31);
		BigDecimal bb = new BigDecimal(0.2);
		BigDecimal cc = new BigDecimal(0.11);
		
		
		System.out.println(aa.subtract(bb).doubleValue()); //转换成double
		System.out.println(aa.subtract(bb));
		System.out.println(cc.compareTo(aa.subtract(bb))); 
		
	}
	
	//输出
	0.10999999999999999
    0.109999999999999986677323704498121514916419982910156250
    1 
	
```
BigDecimal其中一个构造函数以双精度浮点数作为输入，然而得到的结果仍然不是我们想要的。要小心使用 BigDecimal(double)构造函数，因为如果不了解它，会在计算过程中产生舍入误差。请使用基于整数或 String 的构造函数。

## 3、解决double和float精确计算有误差的方法
在《Effective Java》这本书中也提到这个原则，float和double只能用来做科学计算或者是工程计算，在商业计算中我们要用java.math.BigDecimal。使用BigDecimal并且一定要用String来够造。

BigDecimal用哪个构造函数？

BigDecimal(double val) 

BigDecimal(String val)  
上面的API简要描述相当的明确，而且通常情况下，上面的那一个使用起来要方便一些。我们可能想都不想就用上了，会有什么问题呢？等到出了问题的时候，才发现参数是double的构造方法的详细说明中有这么一段：

Note: the results of this constructor can be somewhat unpredictable. One might assume that new BigDecimal(.1) is exactly equal to .1, but it is actually equal to .1000000000000000055511151231257827021181583404541015625. This is so because .1 cannot be represented exactly as a double (or, for that matter, as a binary fraction of any finite length). Thus, the long value that is being passed in to the constructor is not exactly equal to .1, appearances nonwithstanding. 

The (String) constructor, on the other hand, is perfectly predictable: new BigDecimal(".1") is exactly equal to .1, as one would expect. Therefore, it is generally recommended that the (String) constructor be used in preference to this one.

原来我们如果需要精确计算，非要用String来够造BigDecimal不可！

代码示例：
```
    public static void main(String[] args) {
		
		BigDecimal aa = new BigDecimal("0.31");
		BigDecimal bb = new BigDecimal("0.2");
		BigDecimal cc = new BigDecimal("0.11");
		
		
		System.out.println(aa.subtract(bb).doubleValue()); 
		System.out.println(aa.subtract(bb));
		System.out.println(cc.compareTo(aa.subtract(bb))); 
		
	}
	
	//输出
	0.11
    0.11
    0
	
```

## 4、bigdecimal比较数值大小的方法
如浮点类型一样，BigDecimal 也有一些令人奇怪的行为。尤其在使用 equals() 方法来检测数值之间是否相等时要小心。 equals() 方法认为，两个表示同一个数但换算值不同（例如， 100.00 和 100.000 ）的 BigDecimal 值是不相等的。然而， compareTo() 方法会认为这两个数是相等的，所以在从数值上比较两个 BigDecimal 值时，应该使用 compareTo() 而不是 equals() 。

另外还有一些情形，任意精度的小数运算仍不能表示精确结果。例如，1 除以 9 会产生无限循环的小数 .111111... 。出于这个原因，在进行除法运算时，BigDecimal 可以让您显式地控制舍入。 movePointLeft() 方法支持 10 的幂次方的精确除法。

```
    public static void main(String[] args) {
		
		BigDecimal aa = new BigDecimal("0.31");
		BigDecimal bb = new BigDecimal("0.2");
		BigDecimal cc = new BigDecimal("0.11");
		
		
		int r = cc.compareTo(aa.subtract(bb));
		
		if(r==0) //等于
        if(r==1) //大于
        if(r==-1) //小于
		
	}
	
```

## 5、小结
本人真实的项目中发现了此问题，而写这些代码的都是 “高级” 开发工程师，我不知道为什么。这能对得起 “高级工程师” 称号？（笑） 