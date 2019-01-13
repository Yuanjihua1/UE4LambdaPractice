## UE4 C++ 如何通过Lambda使用定时器

### 准备
首先，我会创建一个继承 Actor 的 c++类 CppActor:

![1.png](https://raw.githubusercontent.com/Yuanjihua1/UE4LambdaPractice/master/img/1.png)

然后创建继承c++的蓝图 BlueprintActor

![2.png](https://raw.githubusercontent.com/Yuanjihua1/UE4LambdaPractice/master/img/2.png)

将BlueprintActor拖入到场景中，然后我会在 VS 的 AppActor.cpp的BeginPlay中编写代码。

![3.png](https://raw.githubusercontent.com/Yuanjihua1/UE4LambdaPractice/master/img/3.png)

### 一、先看定时器常见使用的最多的方法
1. 第一步你需要在 .h 文件中创建一个FTimerHandle的属性，和一个TimerCallback的函数

![4.png](https://raw.githubusercontent.com/Yuanjihua1/UE4LambdaPractice/master/img/4.png)

2. 第二步在.cpp中设置一个定时器。

![5.png](https://raw.githubusercontent.com/Yuanjihua1/UE4LambdaPractice/master/img/5.png)

3. 第三步需要实现TimerCallback的具体代码(见上图)。

*这种写法步骤会很多，当一个类中需要使用很多定时器的时候，会显得代码非常的臃肿，而且命令也是一个另人头疼的事*

### 二、用Lambda的方式来使用定时器
1. 

![7.png](https://raw.githubusercontent.com/Yuanjihua1/UE4LambdaPractice/master/img/7.png)

![6.png](https://raw.githubusercontent.com/Yuanjihua1/UE4LambdaPractice/master/img/6.png)

2. 通过日志发现，打印的顺序并没有按照 ① ② 的顺序去打印。而是先打印SetTimer, 然后打印Lambda里的内容。这说明定时器的延时效果是成功的。

#### 如何在Lambda中使用外部的变量,如何给Lambda传递参数,使它跟js, swift...等一类语言的类似写法一样功能更完善。

### 三、Lambda的构成
1. Lambda分三部分构成:
    -  [] 捕获列表: 用来捕获Lambda外的变量。
    -  () 参数列表: 在调用Lambda时，传递过去的参数。
    -  {} 代码块: Lambda的具体实现代码的部分。
2. 具体的用法
``` cpp
// 只读取外部的值
int a = 1;
auto Lambda = [a]() {
    UE_LOG(LogTemp, Log, TEXT("a: %d"), a);
);
Lambda();
//在Log中，a: 1
```

``` cpp
// 读取外部的值，并+1
int a = 1;
auto Lambda = [a]() {
   a = a + 1;
   UE_LOG(LogTemp, Log, TEXT("in lambda, a: %d"), a);
   };
Lambda();
UE_LOG(LogTemp, Log, TEXT("out lambda, a: %d "), a);
//编译会出问题，显示错误为: 无法在非可变 lambda 中修改通过复制捕获
```

*所以可变的Lambda的写法：*
``` cpp
// 读取外部的值，并+1
int a = 1;
auto Lambda = [a]() mutable { //加上 mutable
   a = a + 1;
   UE_LOG(LogTemp, Log, TEXT("in lambda, a: %d"), a);
};
Lambda();
UE_LOG(LogTemp, Log, TEXT("out lambda, a: %d "), a);
// 在加上 mutable 之后， 就可以对 a 进行赋值。
// 但是打印出来的结果: 
// in lambda, a: 2
// out lambda, a: 1
// 在 lambda 内部确实对a进行了赋值
// 在 lambda 外部却没有改变a的值
// 对于js,swift...这类的语言开发者来说会感到困惑，和预期的并不一样。
```

*所以应该怎么写才能改变 lambda 外部的值，也就是a的值*

*那么就是使用引用捕获*

``` cpp
int a = 1;
int b = 2;
int c = 5;
// 引用捕获就是在 变量 的前面加上&
auto Lambda = [&a,&b,&c]() {

   b = b + 1;
   c = c + 1;

   a = a + 1;
   UE_LOG(LogTemp, Log, TEXT("in lambda, a: %d"), a);
};
Lambda();
UE_LOG(LogTemp, Log, TEXT("out lambda, a: %d "), a);
// Log的结果中
// in lambda, a: 2
// out lambda, a: 2
// 此时，确实符合我们的预期
// ps: 
// [&a,&b,&c] 等同于 [&]
// 当捕获的变量都是通过引用捕获时, 就可以简写成 [&]
```

*改变外部参数的预期已经完成，接来下就是向lambda函数内部传递参数*

比如要写成这个样子：

![8.png](https://raw.githubusercontent.com/Yuanjihua1/UE4LambdaPractice/master/img/8.png)

那么就得这么写：
``` cpp
int a = 3;

auto Lambda = [&a]( FString passStr ) {
    // (FString passStr) 表示在调用这个 lambda 的时候需要在传入一个 FString 类型的参数

   a = a + 1;
   UE_LOG(LogTemp, Log, TEXT("in lambda, passStr: %s"), *passStr);
};

FString str = "I am a string from lambda outside";
Lambda(str);

// 如果此时使用的是 Lambda() , 没有向里面传值的话就会报错。
// 在调用 Lambda 的时候就必须传入一个 (FString)参数列表 里的同类型 FString 的变量；如果 (FString,int,float)参数列表 里的参数有多个, 那么也必须传入多个同类型的参数。

// 如果参数列表没有 即 () 。
// 那么就可以直接使用 Lambda()。
```

*如果你已经有了编程基础，并且你的编程语言中有 block， 闭包， 回调 。。。等此类说法时，你此时的心里应该是这样的:*

![m1.JPG](https://raw.githubusercontent.com/Yuanjihua1/UE4LambdaPractice/master/img/m1.JPG)

*此时在你脑海浮现的就是: 使用定时器的时候在InbLoop这个参数上传入true,使定时器可以循环调用Lambda,当满足一定条件时，调用 ClearTimer 结束循环。*

那么我们就开始 *"试验是检验真理的唯一标准"*

![10.png](https://raw.githubusercontent.com/Yuanjihua1/UE4LambdaPractice/master/img/10.png)

运行完的结果:

![9.png](https://raw.githubusercontent.com/Yuanjihua1/UE4LambdaPractice/master/img/9.png)

![m2.JPG](https://raw.githubusercontent.com/Yuanjihua1/UE4LambdaPractice/master/img/m2.JPG)

*我们发现，它并没有按照我们的预想的结果6，7，8的走下去然后清除timer*

*我们直接注释掉判断语句*

![11.png](https://raw.githubusercontent.com/Yuanjihua1/UE4LambdaPractice/master/img/11.png)

![12.png](https://raw.githubusercontent.com/Yuanjihua1/UE4LambdaPractice/master/img/12.png)

*它也没有清除掉Timer，还是会一直循环调用*

![m3.JPG](https://raw.githubusercontent.com/Yuanjihua1/UE4LambdaPractice/master/img/m3.JPG)

*那么 timer 和 a 到底去哪了？*

---

大部分的语言是没有指针或者弱化了指针的概念，没有栈和堆的概念。如果是其他的语言开发者, 会先入为主, 误入歧途。

![m4.jpg](https://raw.githubusercontent.com/Yuanjihua1/UE4LambdaPractice/master/img/m4.jpg)


 真相只有一个:

``` cpp
FTimerHandle Timer;
int a = 6;

// 这种写法的变量会存放在栈中
// 当变量的所在的函数中运行完成时, 也就是运行到 } 后， 这些变量就会被释放掉
// 那么 lambda 中捕获到的引用所指向的值就是被释放掉的值，就是不存在的值。
```

*那么在c++中就应该在堆中存放值，如果不主动释放，这些值会永远存在, 即使函数运行到 ```}``` 之后*

![13.png](https://raw.githubusercontent.com/Yuanjihua1/UE4LambdaPractice/master/img/13.png)

![14.png](https://raw.githubusercontent.com/Yuanjihua1/UE4LambdaPractice/master/img/14.png)

*运行的最后结果确实符合预期*

![m5.gif](https://raw.githubusercontent.com/Yuanjihua1/UE4LambdaPractice/master/img/m5.GIF)

``` cpp
FTimerHandle * Timer = new FTimerHandle();
int * a = new int(6);
// 使用 new 即在堆中永久存储了这个值
// 使用 *a, *Timer 就是获取这个堆中获取到这个值
// 堆中的值不会自动释放，需要 调用 delete a, delete Timer  主动释放

// 但是 UE4 中的对象 都继承自 UObject , 它们都带有CG, 不需要我们调用 delete。 


// ps: 这里的捕获列表使用的是 [this,Timer,a]
// 为什么 没有用 &
// 因为 this, * ，本身就是指针, 本身就是引用类型，所以不需要&

// 其实这里 [this,Timer,a] 还可以写成 [=]
// ???
// 读者可以自行上网查资料。自己探索，知识才能内化成自己的东西。
```