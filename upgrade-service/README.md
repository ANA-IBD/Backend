# ANA IBD Task 0 upgrade-service

## 故事背景（仅供娱乐，无任务相关内容）

**Disclaimer:** 本故事纯属虚构，与现实中一切人物和团体无关。

> zbc通过HitGub search和关键词提取，发现了Mark服务器的地址，因为A1pha在某个咖啡因超标的午夜不小心将MarkConfig.yml上传到了自己的HitGub开源仓库上。邪恶的zbc对Mark开始进行DOS攻击，但A1pha已经通过流量监控发现了zbc的攻击。为了应对zbc流量不大的DOS攻击，A1pha决定升级自己的服务器。A1pha购买了服务器CPU算力升级包，该升级包支持对一系列服务器进行升级，但是他不清楚选择升级哪些服务器能为Mark带来最大收益，于是他给刚拿到ANA研发部通行证的佐地圭发送了一条信息，请求佐地圭的帮助......
>
> （后面的文字消失了，只留下了几个锟斤铐）

## 设置

我们为每一个提供了`github username`的同学都创建了他们的student仓库，前往GitHub[查看](https://github.com/notifications)邀请。

首先clone你的student仓库：

```bash
git clone git@github.com:ANA-IBD/studentx.git
```

使用`cd studentx`进入你的student目录。

注意将`studentx`换成你自己的student仓库名。

在你的“student”目录中，使用以下命令(Command)获取该任务的文件：

```bash
git pull staff main
```

如果你遇到以下错误：

```
fatal: 'staff' does not appear to be a git repository
fatal: Could not read from remote repository.
```

请确保如下设置了staff remote：

```bash
git remote add staff https://github.com/ANA-IBD/student0.git
```

然后再次运行`git pull staff main`。

之后使用`ls`命令应该能看到一个upgrade-service文件夹。

如果你遇到了合并冲突，并且自己无法解决，请在交大门上创建带anaibd标签的话题或在github上提交**Issues**。

## 基础知识

### 前言

欢迎来到考核的第二部分，在这部分里面，我们需要完成一个算法设计与实现问题，以应对在开发过程中可能出现的各种算法问题。

**对于已经有一定基础的同学，我们建议直接开始[做题](#problem)**；对于小白，我们会教授一些实现算法常用的工具，以帮助你们应对这个考核。

在上一部分的考核中，我们学习了一些C语言基本知识。这里，我将给你们介绍一下C语言的另一个变体（variant）C++，通常也称为cpp，因为C++原文件以.cpp作为后缀。通常cpp是兼容c的各种语法的，仅仅在某些极为特殊的情况下，需要使用一些特殊语法来使cpp兼容c的语法，例如：

```cpp
#ifdef __cplusplus
extern "C" {
#endif

// C 风格的函数声明
void c_style_function();

#ifdef __cplusplus
}
#endif

```

至于在何种情况下需要使用，请自行了解。并且前面已经说过，通常不需要用到。

当然，cpp的语法相比c更为复杂，我们只介绍其中一部分。

### long long（c, cpp都可以使用）

在处理大数据时，用int可能没有办法正确存储数据。这是因为int类型是一个32位二进制数，最高位为符号位，所以int可存储的数据范围为-2<sup>31</sup>~2<sup>31</sup>-1。所以如果输入数据范围或者中间变量的可能取值超过了2147483647，我们就需要使用long long来声明变量。long long是一个64位有符号整数，它的取值范围非常大，至于能取到多少，请自己思考。

### 命名空间（namespace）

C++为了更好地兼容各个库（library），引入了命名空间的概念。命名空间就相当于一个黑盒子，盒子外面看不到里面，盒子里面也看不到外面，除了某些全局变量。所以，不同的两个命名空间互不影响。但是这样的话，黑盒子成为了一个闭包，不就没有意义了吗？所以命名空间也提供了外界访问内部元素的方式，就像在黑盒子上打开了一个口，设置一个门卫（即成员访问运算符"::"），允许外面的人通过门卫与里面的人交互。为了更好地理解，我们可以把命名空间当成一个不需要初始化的对象。命名空间之间通常没有冲突，也就是说，两个不同的命名空间中可以有相同的函数和相同的变量。

要使用命名空间，可以直接调用成员访问运算符"::"，或使用using关键字，以下给出两个C++示例：

```cpp
// prog1: not using 'using'
#include <iostream>

int main() {
    std::cout << "cout is in namespace std" << std::endl;
    return 0;
}



// prog2: using
#include <iostream>

using namespace std;       // std是标准命名空间，许多库函数都在这里面

int main() {
    cout << "cout is in namespace std" << endl;
    return 0;
}
```

至于什么情况下应该用什么方法，请自行思考。

也许你会疑问，为什么有这么多“思考题”？这是因为我们不想把所有事情都说明白，思考能力也是考核的一项内容，在以后的开发中你会遇到很多这样的权衡问题，所以多问问自己为什么，cs的世界从来没有什么理所当然，每一个设计都有它存在的理由。当然，如果你感到疑惑，可以在群里询问。

### C++ STL库

C++ standard library涵盖了诸多算法竞赛常用轮子（工业界术语，即基础设施）。一般来说，我们直接在.cpp文件开头写上`#include <bits/stdc++.h>`，并且引入命名空间std，就可以访问这里面的所有工具。

一些常用的函数有：sort, lower_bound, upper_bound, find, nth_element等

一些常用的容器（字面意思，即存储数据的对象，与docker中的容器是不同的概念）有：array, vector, list, stack, queue, priority_queue, deque, set, map等

这里我们着重介绍一下queue（队列）。

队列，顾名思义，就像日常生活中的排队，先来后到，先入队的先出队。C++队列可以用来存储各种各样的数据，因为C++队列是使用模板（template）实现的，template是C++的关键字，如果你对此感到好奇，可以自行学习。在这个任务里我们只需要了解队列如何使用。

队列具有以下接口（interface，面向对象概念之一，可以理解为外界与队列交互的方式，就像机器上的按钮）

- `push(element)`: 将元素添加到队列的末尾。
- `pop()`: 移除队列中的第一个元素。
- `front()`: 返回队列中的第一个元素，但不删除它。
- `back()`: 返回队列中的最后一个元素，但不删除它。
- `empty()`: 检查队列是否为空，如果为空则返回 true，否则返回 false。
- `size()`: 返回队列中元素的数量。
- `emplace()`: 调用传入模板T的构造函数构造一个新的对象，并放入队列。
- `swap()`: 交换两个队列的元素。

```cpp
#include <iostream>
#include <queue>

int main() {
    std::queue<int> myQueue;

    myQueue.push(1);  // 添加元素到队列的末尾
    myQueue.push(2);
    myQueue.push(3);

    std::cout << "Front: " << myQueue.front() << std::endl;  // 访问队列的第一个元素
    std::cout << "Back: " << myQueue.back() << std::endl;    // 访问队列的最后一个元素

    myQueue.pop();  // 移除队列中的第一个元素

    std::cout << "Size: " << myQueue.size() << std::endl;    // 返回队列中的元素数量

    while (!myQueue.empty()) {
        std::cout << myQueue.front() << " ";  // 遍历并打印队列中的元素
        myQueue.pop();
    }
    std::cout << std::endl;

    return 0;
}
```

输出结果如下：

```
Front: 1
Back: 3
Size: 2
2 3
```

## 题目描述<span id="problem"></span>

A1pha在Magic Cloud上有n台服务器，每台服务器有一个固定的算力。为了应对zbc的恶意攻击，A1pha购买了服务器CPU算力升级包。该升级包只能使用一次，但是也可以选择不使用。使用该升级包后，相邻的连续m个服务器的算力将被替换为指定值k。A1pha想知道，如何升级服务器能让收益最大化，即服务器总算力最大。由于可能出现使用升级包后服务器总算力降低或零提升的情况，这种情况下A1pha为了节省经费，会选择不使用升级包，找Magic Cloud退钱。

#### 输入数据格式：

> 第一行n, m, k
>
> 第二行n个正整数a<sub>i</sub>，1 &le; i &le; n，分别代表第i个服务器的初始算力值

#### 输出数据格式：

> 一行两个正整数i, j, 1 &le; i, j &le; n，代表选择升级从第i个服务器到第j个服务器，如果选择不升级，输出0 0
>
> 如果有多个满足条件的最优解，你需要输出i值最小的最优解。

输入样例1

> 5 3 6
>
> 1 2 3 2 5

输出样例1

> 1 3

输入样例2

> 6 2 1
>
> 1 4 7 8 9 6

输出样例2

> 0 0

样例1解释：显然，选择升级服务器1, 2, 3收益最大。

样例2解释：不论选择哪两个连续的服务器升级，总算力都不会提高，所以选择不使用升级包，输出0 0。

#### 数据范围

**对于20%的数据**

1 &le; n, a<sub>i</sub>, k, m &le; 10

**对于60%的数据**

1 &le; n, m &le; 1000

1 &le; a<sub>i</sub>, k &le; 1&times;10<sup>6</sup>

**对于100%的数据**

1 &le; n &le; 2&times;10<sup>7</sup>

1 &le; a<sub>i</sub>, k &le; 1&times;10<sup>11</sup>

1 &le; min(m, n - m) &le; 1&times;10<sup>6</sup>

**时间限制：3s，内存限制：128MiB**

## 模板

你可以使用如下模板：

```cpp
#include <bits/stdc++.h>

using namespace std;

// Uncomment the following line if you need it
// typedef long long ll;

int main() {
	long long n, m, k;
	cin.sync_with_stdio(false);
	cin >> n >> m >> k;
	for (long long i = 1; i <= n; i++) {
		long long ai;
		cin >> ai;
        // YOUR CODE HERE
        
	}

    cout << "Your answer here" << endl;
	return 0;
}
```

## 提交

保存你的全部工作，commit并push到你的git远程仓库中，然后将其提交到Gradescope上的**upgrade-service**作业。
