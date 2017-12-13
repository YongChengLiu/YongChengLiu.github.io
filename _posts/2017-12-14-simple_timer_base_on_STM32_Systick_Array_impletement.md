---
layout: post
title: 基于stm32 Systick 的简单定时器（裸机）-- 数组实现
key: 20171214
tags:
  - STM32
  - Timer
lang: zh-Hans
---

## **前言**
&emsp;&emsp;在嵌入式的开发中，经常需要执行定时的操作。 聪明的同学肯定会想到， 我可以配置硬件定时器， 然后利用定时器中断来执行需要定时执行的代码。然而硬件定时器的数量总是有限，不一定可以满足我们定时的需求。因此我们常常需要用到软件定时的方法。

&emsp;&emsp;事实上，关于使用软件定时，如果有用到操作系统的内核(例如uCOS、FreeRTOS等)的话是非常爽的，因为内核已经帮你做了很多工作。你只需要调用定时器的API创建定时器结构体并且编写定时器回调函数，就可以完成定时工作的代码。

&emsp;&emsp;有时我们的项目很简单，不需要使用到操作系统。或者我的SOC资源极其有限，ROM和RAM小到连操作系统的代码都跑不起来，那么就可能需要我们自己来写实现定时的代码。
<!--more-->

### **一些假设**
本文假设读者：
* 掌握C语言
* 具有类似STM32等单片机/SoC的编程经验
### **一些说明**
* 文中使用`...`来代替省略掉的代码，然后将注意力集中在我们需要讨论的内容。

### **工具/材料**
- 一个stm32开发板(笔者使用的开发板是秉火指南者，SOC是STM32F103VET6)
- keil


### **一个不通用的定时实现**
我们先来讨论一个不太通用的软件定时的实现。
- **假如我让一个LED灯定时闪烁，亮1s灭1s，要怎么实现呢？**

&emsp;&emsp;当笔者还是菜鸡的时候(现在也是)，就会想，这很简单，我直接

```C
...
int main(void)
{
    led_init();
    while(1){
        led_on();
        delay_ms(1000);
        led_off();
        delay_ms(1000);
    }
    return 0;
}
...
```

不就可以了嘛。

&emsp;&emsp;好，可以，没问题。那么我要增加难度了。
- **我现在不仅要让led亮1秒暗一秒地闪，还要处理来自各个传感器(外设)的数据，然后实时显示到LCD屏幕上。**

&emsp;&emsp;我们没有使用到类似uCOS这种抢占式可剥夺型的操作系统内核，没有任务切换，所有的代码都要放在一个死循环中，CPU会消耗大量的时间执行delay_ms，然后再执行其他代码。用上述延时的方式实现定时，必然会**很不实时**。

&emsp;&emsp;当然，如果有一款SOC，配置无限个中断，那么请忽略本文所讨论的内容。

&emsp;&emsp;我见过一种实现是这样的。利用了STM32中的嘀嗒定时器实现软件定时。在led.c中声明2个变量，这两个变量必然是全局变量。
```C
unsigned char led_state; // 用于记录led的状态，假设 0 表示led暗， 1表示led亮
unsigned int led_count; // 用于定时计数
```

&emsp;&emsp;在Main函数中，对led_state 和 led_count判断，执行不同的代码：

```C
...
int main(void)
{
    SystemInit(); // 系统初始化。如果你有留意到STM32的启动文件中的汇编代码的话，你会发现在进入main函数前，会先执行这个函数。
    SysTick_Config(SystemCoreClock/1000); // 1ms 进入一次嘀嗒中断服务函数
    led_init();
    ...
    while(1){
        ...
        if((led_count == 0) && (led_state == 0)){
            led_on();
            led_count = 1000;
        }
        if((led_count == 0) && (led_state == 1)){
            led_off();
            led_count = 1000;
        }
        ...
    }
}
```

&emsp;&emsp;然后在嘀嗒中断服务函`SysTick_Handler`中，让 led_count > 0 时自减。这个函数通常放在 `system_stm32f10x.c` 中。

```C
void SysTick_Handler(void)
{
	if(led_count != 0x00)
	{
		led_count--;
	}
}
```
&emsp;&emsp;这样的代码是可以满足我们的定时和实时性的要求的。但是，由于使用了2个全局变量`led_state`和`led_count`，并且这两个变量很可能还是跨文件的全局变量(通常而言， main.c led.c system_stm32f10x.c)，就导致了这份代码的可读性和可维护性是非常糟糕的。在阅读这份代码的时候，不得不去找这两个变量首先在哪里声明，然后有那些地方修改了这两个变量。另外一方面，假如有关led的需求变了，我需要修改相关的代码，那么很可能我不得不修改`main.c`、`led.c`、`system_stm32f10x.c`的代码。

## **基于stm32 Systick 的简单定时器(裸机)**
### **抽象**
&emsp;&emsp;我们前面讨论了一个不通用的代码，不管阅读还是维护都需要耗费极大的精力。

&emsp;&emsp;为了让代码变得更加通用，我们需要引入ADT(抽象数据类型，Abstract Data Type)的概念。即想办法将待求解的问题抽象成一个数据类型，然后思考并且实现这个数据类型支持的操作。比如说，`int` 类型的数据是C语言内建的数据类型，它可以进行`+`、`-`、`×`、`÷`的操作。我们可以尽情地使用(加减乘除)的操作处理整型数据而不用考虑加减乘除是怎么实现的。同样道理，对于新的数据类型，在我们实现它的操作集合之后，就可以使用它的操作集合处理这种新类型的数据而不需要考虑这个操作集合是怎么实现的。同时，当我们要修改操作集合的实现的时候，只要接口不变，那么应用的代码就不需要做任何更改

&emsp;&emsp;换成“面向对象编程”的说法就是，首先思考这个东西有什么属性，然后这个东西有什么行为。

&emsp;&emsp;大部分介绍数据结构的书籍，都会介绍ADT的概念。ADT及其操作集合的定义与编程语言无关，而要实现它们则需要考虑具体的编程语言的语法细节。本文无意讨论数据结构和面向对象编程。有兴趣的读者可以去阅读以下材料：
1. [《数据结构与算法分析 —— C语言描述》(《Data Structures and Algorithm Analysis in C》)](http://www.linuxidc.com/Linux/2014-04/99735.htm)
2. 宋宝华的直播视频[《C语言大型软件设计的面向对象》](http://edu.csdn.net/huiyiCourse/detail/594?utm_source=wx2)
3. 师弟的一篇练习作[单片机也能 OO？—— 串口命令解析器的实现](http://www.shaoguoji.cn/2017/11/18/c-object-oriented-command-parser/)。

接下来让我们开始一步一步对软件定时器抽象的过程。
1. **首先思考这个定时器应该怎么表示(它有什么属性)**
- 需要一个变量表示定时器的状态(state)
- 需要一个变量表示定时器的初始计数值(reload)
- 需要一个变量表示定时器的当前计数值(count)
- 当这个定时器到期之后需要执行什么操作(回调函数，callback)

根据C语言的语法，这个软件定时器可以表示成这样：

```C
typedef void (*timerCallBack_t)(void *arg)
typedef struct {
    int state;               // 记录定时器的状态
    int reload;              // 记录定时器的重装载值
    int count;               // 记录定时器的当前计数
    timerCallBack_t callBack;  // 定时器到期后的回调函数
}timer_t;

```

2. **可以对这个定时器进行怎么样的操作?**
- 创建/增加(create)
- 删除(delete)
- 开始计时(start)
- 停止计时(stop)
- 计数值复位(reset)

&emsp;&emsp;现在，我们可以定义可以对这个定时器的合法操作了(接口)

```C
timer_t *timer_create(void);
void timer_delete(timer_t * T);
void timer_start(timer_t *T);
void timer_stop(timer_t *T);
void timer_reset(timer_t *T);
```

### **数组实现**
&emsp;&emsp;通常，我们可能会有不止一个定时任务(这里的任务指需要定时执行的代码)，根据以上我们定义的定时器结构体，我们当然可以这样声明：

```C
timer_t timer1;
timer_t timer2;
...
```
&emsp;&emsp;但是这样子做会造成定时器管理困难。比如说，我要去轮询定时器是否到期，如果到期则调用回调函数，那么很可能代码要这样写：

```C
...
if(timer1.count == 0){
    timer1.callBack(arg);
}
if(timer2.count == 0){
    timer2.callBack(arg);
}
...
```
&emsp;&emsp;一个明智一点的做法是，把这些定时器变量放到一个数组保存，比如：
```C
#define N 10
timer_t timer[N];
```
&emsp;&emsp;那么，轮询定时器的代码就可以写成

```C
int i;
for(i=0; i<N; i++){
    if(timer[i].count == 0){
        timer[i].callBack(arg);
    }
}
```
&emsp;&emsp;现在，我们决定使用数组来存储定时器，然后思考实现“创建/添加定时器"的操作。“创建/添加定时器"即将定时器的各个成员的值填入到定时器数组的一个元素中。那么新的问题就出现了。
1. 怎么样保证往定时器数组填数据的时候，不会填到数组以外的地址？
2. 以上例子声明了一个含有10元素的定时器数组。事实上，我使用到的定时器可能只有2个。那么有没有办法不要每次轮询定时器都要循环10次呢？我希望实际使用多少个定时器轮询时就循环多少次。

&emsp;&emsp;为了解决上述2个问题，需要增加2个变量作为控制。
```C
int current_num; // 当前定时器的数量
int max_num; // 允许的定时器的最大数量
```
&emsp;&emsp;我们把这2个变量和定时器的结构体封装在一起。
```C
struct {
    int current_num;
    int max_num;
    timer_t *timer;
}Timer;
```
&emsp;&emsp;事实上，这是一个线性表的数据结构。为了好看，我们把它写成：
```C
typedef struct {
    int length;     // 当前定时器的个数(当前线性表的长度)
    int listsize;   // 当前允许的定时器的最大个数(即数组的长度) 
    timer_t *timer; // 指向数组的基地址
}timerList_t;
```
&emsp;&emsp;那么对定时器的操作就变成对定时器链表的操作。
* **增加/删除定时器** 相当于 **向线性表中添加结点(node)**
* **启动/停止/复位定时器**相当于**查找并且访问线性表中的定时器**

#### **进一步完善定时器结构体和接口**
以上我们已经得到了用于表述定时器的结构体， 对定时器结构体操作的接口， 用于管理定时器的线性表。 但是还不完整，
- 我们需要在定时器结构体中添加一个变量`unsigned int allocated`用于记录定时器是否被分配到线性表的内存中。
- 我们需要在定时器结构体中添加一个变量`void *arg`用来向定时器传递用户数据。 当然也可以将用户数据定义为全局变量， 然后在回调函数中处理。 不过这样是不安全的， 因为， 很可能还有除了定时器以外的代码修改这些变量。
- 要让定时器运行起来， 还需要增加对定时器轮询的函数`timer_poll`， 并且在`main`函数中的`while`循环或者`SysTick_Handler`函数调用。

&emsp;&emsp;下面， 我们先给出笔者在写这篇文章时最终版本， 然后再讨论具体的实现过程
##### **头文件 timer.h**
```C
#ifndef __TIMER_LIST_H
#define __TIMER_LIST_H

#define config_TIMER_MAX_NUM  10

enum {
    timer_disable = 0,
    timer_enable  = 1,
};

typedef void (*timerCallBack_t)(void *arg);
typedef struct {
    unsigned int allocated;             // 记录线性表中是否已经分配这个定时器
    unsigned int state;                 // 记录定时器的状态
    unsigned int reload;                // 记录定时器的重装载值
    unsigned int count;                 // 记录定时器的当前计数
    void *arg;
    timerCallBack_t callBack;           // 定时器到期后的回调函数
}timer_t;

timer_t *timer_create(unsigned int reload, void *arg, timerCallBack_t callBack);
void timer_delete(timer_t * T);
void timer_start(timer_t *T);
void timer_stop(timer_t *T);
void timer_reset(timer_t *T);
void timer_poll(void);

void test_print_timerList(void);

#endif // __LIST_H
```
##### **定时器线性表在源文件中声明**

```C
typedef struct {
    unsigned int length;     // 当前定时器的个数(当前线性表的长度)
    unsigned int listsize;   // 当前允许的定时器的最大个数(即数组的长度)
    timer_t *timer; // 指向数组的基地址
}timerList_t;
```

#### **timer_create()函数实现**
&emsp;&emsp;创建定时器前， 需要先创建管理定时器的线性表。 我们以静态全局变量的方式声明这个线性表并初始化。

```
static timer_t timer[config_TIMER_MAX_NUM] = { 0 };
static timerList_t L = {0, config_TIMER_MAX_NUM, timer};
```
在头文件中， 定义了宏`#define config_TIMER_MAX_NUM  10`， 因此这个线性表最多只能容纳10定时器。

```C
timer_t *timer_create(unsigned int reload, void *arg, timerCallBack_t callBack)
{
    timer_t *new_timer;
    if(L.length >= L.listsize){
        LOG_E(("timer list is full"));
        return NULL;
    }
    new_timer = find_first_not_alloc_timer(&L);
    new_timer->allocated = 1;
    new_timer->reload = reload;
    new_timer->count = reload;
    new_timer->state = timer_disable;
    new_timer->arg = arg;
    new_timer->callBack = callBack;
    L.length++;
    return new_timer;
}
```
&emsp;&emsp;在分配空间之前， 首先判断线性表是否已经满了， 如果已满， 则返回`NULL`

&emsp;&emsp;在线性表中找到第一个没有被分配的空间， 返回它的首地址。 然后使用传入的参数`reload`， `arg`， `callBack`初始化定时器。 

&emsp;&emsp;`new_timer->allocated = 1;` 指示定时器已经被分配到线性表中。

&emsp;&emsp;`new_timer->count = reload;`表示`reload`值已经被装入到`count`中。

&emsp;&emsp;`L.length`计数加1， 表示当前定时器的数量。

##### **find_first_not_alloc_timer函数实现**
&emsp;&emsp;遍历线性表， 如果`allocated == 0`， 则表示这个空间没有被分配， 返回这个空间的首地址。 否则返回`NULL`
```C
static timer_t * find_first_not_alloc_timer( timerList_t *L)
{
    int i = 0;
    for(i=0; i<L->listsize; i++){
        if(L->timer[i].allocated == 0){
            return (&(L->timer[i]));
        }
    }
    LOG_E(("timer list is full"));
    return NULL;
}
```

#### **timer_delete()函数实现**

```C
void timer_delete(timer_t *T)
{
    int i;

    // 对线性表遍历, 确保T指向的地址在线性表中
    for(i=0; i<L.listsize; i++){
        if(&(L.timer[i]) == T){
            memset(&L.timer[i], 0, sizeof(timer_t));
            T = NULL;
            L.length--;
        }
    }
    if(i == L.listsize)
        LOG_E(("the is not in timer list"));
}
```
&emsp;&emsp;传入要从线性表中删除的定时器指针给`timer_delete`函数， 然后对线性表进行遍历， 通过地址匹配的方式找到待删除的定时器， 把定时器的所有内容设置为0。

&emsp;&emsp;笔者曾经考虑过另外的实现:
1. 按照常规的线性表删除结点的做法， 在删除结点的时候， 被删除结点后面的结点应该要往前移动。 笔者在定时器结构体`timer_t`中添加一个变量`id`， 一方面用来记录定时器的id， 同时也代表了定时器在线性表中的位置。 通过`id`直接找到要删除的结点并且将后面的结点往前移。 很明显， 这种方法是不行的， 因为一旦移动了结点， 那么应用部分的代码很可能就会访问到错误的定时器结点。

```C
//如果删除结点的时候, 同时移动结点, 那么可能会导致对其他定时器的访问错误. 因为可能我们需要访问的定时器地址已经发生变化.
void timer_delete(timer_t * T)
{
    int i;
    timer_t temp;

    // 如果删除的定时器在线性表的最后一个结点
    if(T->id == L.length-1){
        memset(T, 0, sizeof(timer_t));
        T = NULL;
        L.length--;
        return;
    }
    memcpy(&temp, T, sizeof(timer_t));
    for(i=temp.id; i<L.length - 1 ; i++){
        memcpy(&(L.timer[i]), &(L.timer[i+1]), sizeof(timer_t));
        memset(&(L.timer[i+1]), 0, sizeof(timer_t));
    }
    T = NULL;
    L.length--;

}
```

2. 在上述第1点的基础上修改， 在定时器结构体`timer_t`中添加一个变量`id`， 并且在删除定时器结点的时候不移动结点。 将`timer_t`中的`allocated`设置为0， `timer_t`中的其他内容也设置为0。 这似乎是一个非常高效的方法， 通过`id`直接找到待删除的结点， 同时也不会影响应用代码对定时器结点的访问。 但是我们并知道会传入什么样的 `timer_t * T`。 如果传入的指针的值不在线性表的地址范围内， 但是刚好满足`(T->id >= 0) && (T->id < L.listsize)`， 那么我们就会破坏了`timer_t * T`指向的内存。 甚至在执行删除操作前加入一个判断条件`(T >=&L.timer[0]) && (T < &L.timer[L.listsize-1])`都是不安全的。 因为我们不能够保证在正确的地址上修改内容。 

```C
void timer_delete(timer_t * T)
{
    int i;

    if((T->id >= 0) && (T->id < L.listsize) ){
        memset(&(L.timer[i]), 0, sizeof(timer_t));
        T = NULL;
        L.length--;
    }

}
```

#### **timer_start()、timer_stop()、timer_reset()函数实现**
&emsp;&emsp;这3个函数的实现会比较简单， 因为我们已经取得了定时器的地址， 直接通过定时器的地址访问`timer_t`的成员变量即可。

```C
void timer_start(timer_t *T)
{
    T->state = timer_enable;
}

void timer_stop(timer_t *T)
{
    T->state = timer_disable;
}

void timer_reset(timer_t *T)
{
    T->count = T->reload;
    T->state = timer_enable;
}
```

#### **timer_poll() 函数实现**
&emsp;&emsp;timer_poll函数需要放到`main`函数中的`while`循环或者`SysTick_Handler`函数中执行。 如果将timer_poll函数需要放到 `main`函数中执行， 那么则需要在`SysTick_Handler`函数中设置标志变量， 示例代码如下:

```
void SysTick_Handler(void)
{	
	if(SysTick_Handler_Flag == 0)
		SysTick_Handler_Flag = 1;

}
```

```C
void timer_poll(void)
{
    int i;
    
    if(SysTick_Handler_Flag == 1){
        for(i=0; i<L.listsize; i++){
            if(L.timer[i].allocated && L.timer[i].state){
                if(L.timer[i].count > 0){
                    L.timer[i].count--;
                }
                if(L.timer[i].count == 0){
                    L.timer[i].state = timer_disable;
                    L.timer[i].callBack(&L.timer[i]);
                }
            }
        }
        SysTick_Handler_Flag = 0;
    }
}
```
&emsp;&emsp;在`timer_poll()`中， 遍历线性表， 对满足`L.timer[i].allocated && L.timer[i].state`的定时器` L.timer[i].count--`， 当`L.timer[i].count == 0`， 则停止定时器并且执行回调函数， 并且将定时器自身的指针传入到回调函数。

##### **缺点**
&emsp;&emsp;到此， 我们已经实现了软件定时器的核心代码。 这种实现是有缺点的。
1. 只能允许少量的定时器， 否则仅对定时器线性表的遍历就会浪费大量的时间。
2. 在回调函数中不能够执行阻塞的代码或者需要等待太长时间的代码， 否者会导致其他定时器同样阻塞。

### **测试代码**
[利用刚刚实现的软件定时器让LED灯定时1秒翻转一次](https://github.com/YongChengLiu/stm32/tree/master/stm32f103vet6/1.soft_timer)

### **更多**
&emsp;&emsp;在对定时器线性表执行操作的时候， 我们只保证了不会访问到不对的地址。 在增加和删除定时器结点的时候， 还是不得不遍历定时器链表。 

&emsp;&emsp;我们知道， 线性表可以有两种实现， 一种是数组， 一种是链表。 而链表实现， 则会实现我们希望有多少定时器就访问多少定时器的想法。 我们会在后面的内容中讨论链表实现。

&emsp;&emsp;这个软件定时器是非常简单的。但“麻雀虽小，五脏俱全”。如果仔细阅读FreeRTOS的定时器实现，你会发现原理是类似的。FreeRTOS的定时器实现更加复杂。使用了一个Daemon任务运行定时器，使用2个链表和1个队列管理定时器。




