---
title: 从 “剑指 Offer 20. 表示数值的字符串” 引发的三色标记法构建状态机思考
date: 2020-10-14 22:06:12
category: 沉淀
tags: ["技术杂谈", "沉淀"]
---

偶然间在 LeetCode 上看到一个令人惊叹的题解，并从中引发了对状态机的学习和思考。从网上了解到的资料来看，状态机好像是编译原理里面的知识，但是大学课程没有这个知识的学习，所以还是按照自己理解的范畴来写吧。
题目地址 [剑指 Offer 20. 表示数值的字符串][biao-shi-shu-zhi-de-zi-fu-chuan-lcof]

[biao-shi-shu-zhi-de-zi-fu-chuan-lcof]:https://leetcode-cn.com/problems/biao-shi-shu-zhi-de-zi-fu-chuan-lcof/
<!--more-->

### 题目描述

请实现一个函数用来判断字符串是否表示数值（包括整数和小数）。例如，字符串"+100"、"5e2"、"-123"、"3.1416"、"-1E-16"、"0123"都表示数值，但"12e"、"1a3.14"、"1.2.3"、"+-5"及"12e+5.4"都不是。

### 解法 [摘自作者的解法][jiefa]

[jiefa]:https://leetcode-cn.com/problems/biao-shi-shu-zhi-de-zi-fu-chuan-lcof/solution/mian-shi-ti-20-biao-shi-shu-zhi-de-zi-fu-chuan-y-2/
使用有限状态自动机。根据字符类型和合法数值的特点，先定义状态，再画出状态转移图，最后编写代码即可。（能想出这种解法的人可以用神仙来称呼，太牛逼了！！！）

字符类型：```空格 「 」、数字「 0—90—9 」 、正负号 「 +-+− 」 、小数点 「 .. 」 、幂符号 「 eE*e**E* 」 。```

状态定义：

```
按照字符串从左到右的顺序，定义以下 9 种状态。

0.开始的空格
1.幂符号前的正负号
2.小数点前的数字
3.小数点、小数点后的数字
4.当小数点前为空格时，小数点、小数点后的数字
5.幂符号
6.幂符号后的正负号
7.幂符号后的数字
8.结尾的空格
```

结束状态：```合法的结束状态有 2, 3, 7, 8 ```

![07cVI0.png](https://s1.ax1x.com/2020/10/16/07cVI0.png)

```java
class Solution {
    public boolean isNumber(String s) {
        Map[] states = {
            new HashMap<>() {{ put(' ', 0); put('s', 1); put('d', 2); put('.', 4); }}, // 0.
            new HashMap<>() {{ put('d', 2); put('.', 4); }},                           // 1.
            new HashMap<>() {{ put('d', 2); put('.', 3); put('e', 5); put(' ', 8); }}, // 2.
            new HashMap<>() {{ put('d', 3); put('e', 5); put(' ', 8); }},              // 3.
            new HashMap<>() {{ put('d', 3); }},                                        // 4.
            new HashMap<>() {{ put('s', 6); put('d', 7); }},                           // 5.
            new HashMap<>() {{ put('d', 7); }},                                        // 6.
            new HashMap<>() {{ put('d', 7); put(' ', 8); }},                           // 7.
            new HashMap<>() {{ put(' ', 8); }}                                         // 8.
        };
        int p = 0;
        char t;
        for(char c : s.toCharArray()) {
            if(c >= '0' && c <= '9') t = 'd';
            else if(c == '+' || c == '-') t = 's';
            else if(c == 'e' || c == 'E') t = 'e';
            else if(c == '.' || c == ' ') t = c;
            else t = '?';
            if(!states[p].containsKey(t)) return false;
            p = (int)states[p].get(t);
        }
        return p == 2 || p == 3 || p == 7 || p == 8;
    }
}
```

### 状态机的定义

有限状态机（英语：finite-state machine，缩写：FSM）又称有限状态自动机（英语：finite-state automation，缩写：FSA），简称状态机，是表示有限个状态以及在这些状态之间的转移和动作等行为的数学计算模型。

### 状态机的本质

与我而言，状态机的本质其实是高维度的抽象模型。对我们的业务模型来说，数据的流向必定是携带状态的。

比如现在在做的账单系统和支付系统，每一条账单或每一笔支付流水在每个时刻必定是携带状态的，账单的状态和支付的状态必定是一个有限的状态，在服务拆分的模式中，是否有一种以状态机为粒度的架构模式，面向状态机编程。

### 从 GC 可达性分析的三色标记法联想状态机模型

##### 三色标记法

三色标记法是为了解决增量式 GC 的并发场景下，能够解决对象之间依赖关系的不稳定的有效方案。

1. 白色 -- 表示对象尚未被垃圾回收器访问过，且对象中的所有引用都已经扫描过。
2. 灰色 -- 表示对象已经被垃圾回收器访问过，但这个对象至少存在一个引用还没有被扫描过。
3. 黑色 -- 表示对象已经被垃圾回收器访问过，且这个对象的所有引用都已经扫描过。

![three color.png](https://i.loli.net/2020/10/19/t9w8hLidKOZv3Mb.png)

在这里解释一下，其实这张图是经历了两次gc，进行第二次gc中的图片。
第一次gc的时候，只有一个A对象，所以经历完gc后，A被标记为黑色。
第二次gc的时候，A已经被标记为黑色，但是这个时候A的引用链上又新增了B、C两个新对象，此时垃圾收集器正好扫描过B，但是还没有扫描到C，所以才会呈现图上的场景。

在经历完标记过程后，只会剩下黑色和白色的对象，白色对象就是需要被回收的，所以灰色对象可以看作是黑白两色对象的中间态。

##### 三色算法的抽象步骤

现代拥有可达性分析的gc一般都是用了三色算法的思想，虽然实现细节可能不同，但是其抽象步骤都是大致相同的。
观察了lua和go的三色标记步骤，发现大致是相同的。（为什么不观察 jvm，因为 jvm 的源码就你妈离谱，可读性太差了，我也不会cpp，看不懂）
并发标记下，单纯的三色标记会存在漏标，可以通过读写屏障解决，通过屏障来实现强/弱三色不变性。
早期的 cms 和 g1 是通过写屏障更新的，zgc 是通过读屏障更新的，读写屏障对于gc的性能差异留在以后再讨论吧。

1. 初始状态都是处于白色标记。
2. 将 gc roots 直接引用的对象标记为灰色。
3. 将本对象（灰色对象）标记为黑色，将本对象（灰色对象）直接引用到的其他对象都变成灰色。
4. 重复上一步，直到不存在灰色对象。
5. 回收白色对象。

##### 增量式 gc 三色标记的三色不变性

强三色不变性：```保证永远不会存在黑色对象到白色对象的引用```
弱三色不变性：```所有被黑色对象引用的白色对象，都可以直接或间接从灰色对象到达```

##### 三色算法的简单状态机实现（从强三色不变性前提下出发的简单模型）

[![BP441I.png](https://s1.ax1x.com/2020/10/21/BP441I.png)](https://imgchr.com/i/BP441I)

```java
public enum Event {
    INIT("初始化", Event::init),
    DIRECTLY_QUOTED_BY_GC_ROOTS("被gc roots直接引用", Event::directlyQuotedByGcRoots),
    DIRECTLY_QUOTED_BY_GREY_OB("被灰色对象直接引用", Event::directlyQuotedByGreyObj),
    SCAN_TO_THE_CURRENT_GREY_OB("当前灰色对象被扫描到", Event::scanToTheCurrentGreyObj),
    DIRECT_REFERENCE_OBJ_TURNS_WHITE("直接引用对象变为白色", Event::directReferenceObjTurnsWhite)
    ;

    String desc;
    Runnable action;

    Event(String desc, Runnable action) {
        this.desc = desc;
        this.action = action;
    }

    private static void init() {
        System.out.println("初始化，所有对象都被标记为白色");
    }

    private static void directlyQuotedByGcRoots() {
        System.out.println("被gc roots直接引用，当前对象从白色变为灰色");
    }

    private static void directlyQuotedByGreyObj() {
        System.out.println("被灰色对象直接引用，当前对象从白色变为灰色");
    }

    private static void scanToTheCurrentGreyObj() {
        System.out.println("当前灰色对象被扫描到，当前对象从灰色变为黑色");
    }

    private static void directReferenceObjTurnsWhite() {
        System.out.println("直接引用对象变为白色，当前对象从黑色变为灰色");
    }
}

public enum Color {
    WHITE(0, "white"),
    GREY(1, "grey"),
    BLACK(2, "black");

    int code;
    String colorDesc;

    Color(int code, String colorDesc) {
        this.code = code;
        this.colorDesc = colorDesc;
    }
}

public class TransferTable {
    private static final List<Transfer> transferTable = new ArrayList<Transfer>() {
        {
            add(Transfer.of( Color.WHITE,   Color.GREY,     Event.DIRECTLY_QUOTED_BY_GC_ROOTS));
            add(Transfer.of( Color.WHITE,   Color.GREY,     Event.DIRECTLY_QUOTED_BY_GREY_OB));
            add(Transfer.of( Color.WHITE,   Color.WHITE,    Event.INIT));
            add(Transfer.of( Color.GREY,    Color.BLACK,    Event.SCAN_TO_THE_CURRENT_GREY_OB));
            add(Transfer.of( Color.BLACK,   Color.GREY,     Event.DIRECT_REFERENCE_OBJ_TURNS_WHITE));
            add(Transfer.of( Color.BLACK,   Color.WHITE,    Event.INIT));
        }
    };

    static class Transfer {
        Color startColor;
        Color nextColor;
        Event event;

        static Transfer of(Color startColor, Color nextColor, Event event) {
            Transfer transfer = new Transfer();
            transfer.startColor = startColor;
            transfer.nextColor = nextColor;
            transfer.event = event;
            return transfer;
        }
    }

    public static void dispatch(JVMObj obj, Event event) throws Exception {
        for (Transfer transfer : transferTable) {
            if(transfer.startColor == obj.color && transfer.event == event){
                transfer.event.action.run();
                obj.color = transfer.nextColor;
                return;
            }
        }
        throw new Exception("状态流转不合法");
    }
}

public class JVMObj {
    Color color = Color.WHITE;
}

public class Main {
    public static void main(String[] args) throws Exception {
        JVMObj obj = new JVMObj();
        TransferTable.dispatch(obj, Event.INIT);
        TransferTable.dispatch(obj, Event.DIRECTLY_QUOTED_BY_GC_ROOTS);
        TransferTable.dispatch(obj, Event.SCAN_TO_THE_CURRENT_GREY_OB);
        TransferTable.dispatch(obj, Event.DIRECT_REFERENCE_OBJ_TURNS_WHITE);
        TransferTable.dispatch(obj, Event.SCAN_TO_THE_CURRENT_GREY_OB);
        TransferTable.dispatch(obj, Event.INIT);
        System.out.println(obj.color.colorDesc);
    }
}
```

在一般的三色标记算法中，用了 white_list、black_list、grey_list 三个集合来存储对象的三色状态，跟上述针对对象自己颜色转变的例子的维度不同。其实理解起来也不难，上述的case是为了从状态机模型出发，去构思了一个对象三色转变的生命周期。而一般 gc 算法中通过集合去存储对象的原因是因为便于垃圾回收，直接清空集合就可以，不需要再根据引用去查找白色对象（我猜的）。

毕竟第一次自己梳理状态机模型，难免有点问题，如有不正，欢迎指出。