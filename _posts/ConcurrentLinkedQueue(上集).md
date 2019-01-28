# 	ConcurrentLinkedQueue(上集)

##  算法实现 CAS

- ### CAS的优点

  当一个线程执行任务失败不影响其他线程的进行 最大限度的利用CPU资源 能提高程序的伸缩性 伸缩性:不修改任何代码 升级硬件就能带来性能上的提高 升级硬件带来的性能提高明显 就是伸缩性良好

- ### CAS的缺点

  代码复杂 影响阅读性 刚开始看`ConcurrentLinkedQueue`的时候 没有正确的思路,理解起来会比较费劲 我推荐直接用多线程同时执行的方式去理解 这样会比较好

## 重要概念



- ### 不变性

  - 所有item不为null的节点都能从head头节点开始通过succ()方法访问到 

  - head!=null 只要队列有值 保证真实的head永不为null head哪怕会自引用 迟早也会解除这种假状态

    

- ### 可变性

  - heatd.item 可能为null也可能不为null 因为cas活锁操作 每一行代码执行都不影响其他线程的访问相同的代码块
  - tail尾节点的更新是滞后于head的 个人理解 在offer中 尾节点掉队后 通过head节点 (不变性1的保证) 成功访问最后一个p.next=null的节点

- ### 快照

  -  snapshot是我自己的理解 因为对于多线程操作来说 当前引用对象 如offer()中 t=tail中的t; p=t中的p; q=p.next中的q都是一个快照 他获得一个对象的快照版本 然后在后续的操作中 使(t!=(t=tail))这样操作有意义

## 重要方法

- `offer()`入队
- `poll()` 出队



## 源码

```java
public boolean offer(E e) {
        checkNotNull(e); //NullPointException检查   
        final Node<E> newNode = new Node<E>(e); //包装成一个Node对象

        for (Node<E> t = tail, p = t;;) {//获取当前尾节点 t=tail,p是真正的尾节点 p.next==null 
            Node<E> q = p.next;
            if (q == null) {
                // p is last node 
                if (p.casNext(null, newNode)) {//方法1 CAS更新 自己想3个线程同时进行这个操作
                    // Successful CAS is the linearization point
                    // for e to become an element of this queue,
                    // and for newNode to become "live".
                    if (p != t) // hop two nodes at a time //方法2 延迟更新尾节点 下面说为什么
                        casTail(t, newNode);  // Failure is OK.  成不成功无所谓 下面说
                    return true;
                }
                // Lost CAS race to another thread; re-read next
            }
            else if (p == q)// 方法4 学习offer方法时 可以暂时放弃这一步
                // We have fallen off list.  If tail is unchanged, it
                // will also be off-list, in which case we need to
                // jump to head, from which all live nodes are always
                // reachable.  Else the new tail is a better bet.
                p = (t != (t = tail)) ? t : head;
            else  //去找到真正的尾节点 此处和方法2 应是相互辉映的存在
                // Check for tail updates after two hops.
                p = (p != t && t != (t = tail)) ? t : q; //方法5
        }
    }
```

### 解读offer()方法

- 自顶向下 思考CAS中可能出现的情况 CAS是活锁 所谓活锁即是每一行代码运行时 允许其他线程访问相同的代码块 成功与失败并存 衍生了更多的条件判断 本人觉得CAS方法都应该从这个方法去理解 再自己画画时序图 (注意:理解`offer()时,先把方法4排除`，因为4方法出现自引用的情况 只有`offer()`和`poll()`交替执行时会出现)

  - 多线程操作
    1. 第一种情况: 只有 `offer()`
    2. 第二种情况: `offer()`和 `poll()`方法交替执行

  - 同时执行`offer()`(假设我们现在有3个线程)

    - 不变性:永远只有一个线程CAS成功 并且总会成功一个  
    - 循环次数分析:Thread1 成功 循环一次退出 Thread2失败 再循环一次成功 Thread3失败 再循环两次成功 如果有n个线程同时执行 `offer()` 执行次数 最大为n次 最少为1次
    - 方法5中三目表达式解析: p=condition?result1:result2  我先说一下这里的意义 满足result1的场景为 :获取尾节点tail的快照已经过时了(其他线程更新了新的尾节点tail) 直接跳转到当前获得的最新尾节点的地方 满足result2的场景为:多线程同时操作`offer()` 执行1方法CAS成功后 未更新尾节点(未执行3方法:两种原因 1是未满足前置条件if判断 2是CAS更新失败) 直接找next节点 
    - 方法2与方法5 是整个`offer() ` 操作的点睛之笔 下面解释

       

    

    1. 只有`offer()` 操作时

       #### 假设:

       Thread 1执行完1方法成功 还未执行2方法 Thread2和Thread3进入5方法 ,也就是说Thread2和Thread3执行5方法发生在Thread1执行2方法之前 `Thread2 and Thread3 invoke method5() before Thread1 invoke method2()` 

       此时 Thread2.p =q,Thread3.p=q, 因为p==t成立 时序图如下,然后Thread1执行方法2 p==t 不执行tail尾节点的更新操作 由此可知 尾节点是延迟更新 一切为了更高效～～～

       ![image-20190125115202133](/Users/sunboyoung/Library/Application Support/typora-user-images/image-20190125115202133.png)

       ​									图 1

       Thread 2 与 Thread3 此时再次执行 1 方法 见图1 他们此时的q.next==null 我们规定Thread2 CAS成功 Thread3失败了 成功后的时序图如下 我们假设  ` Thread3 invoke method5() after Thread2 invoke method2()`  Thread2执行方法2 在 Thread3执行方法5之前

       ![image-20190125143253655](/Users/sunboyoung/Library/Application Support/typora-user-images/image-20190125143253655.png)

       ​								       图2

       对于Thread2 进入2方法 p!=t 满足  执行 casTail(t, newNode) 更新尾节点的快照 如下图

        ![image-20190125152955871](/Users/sunboyoung/Library/Application Support/typora-user-images/image-20190125152955871.png)

       ​								      图3

       Thread2 工作完成 退出循环 

       对于Thread3 因为执行1方法失败 进入5方法  此时Thread3的tail快照t3  

       p = (p != t && t != (t = tail)) ? t : q; 

       按图3来翻译

       p=(p!=t3&&t3!=(t3=t2))?t2:q;

       p=t2;//直接去当前能获取到的尾节点！！！

       到这里 `offer()` 方法解决完成

    ### ConcurrentLinkedQueue核心总结 

    - tail和head都是 延迟更新的 但是tail更新在head更新后面 因为方法4中 需要依赖head节点 去找每一个存活的节点
    - 前面的叙述中 可以看到 `offer()` 方法内 核心操作 就是 p=condition?result1:result2 
    - 偶数次`offer()` 操作更新一次tail 单线程的环境下

    ### 与Michael-Scott 队列比较

    - Michael-Scott队列 每次操作 都需要判断是否需要推动尾节点  采取CAS的操作  优点也是缺点 

    - Doug Lead老神仙的CAS 我这个菜鸟猜测 能不用CAS 就尽量不用 因为CAS存在竞争 提供以最少次数的更新达到最终正确的效果

    - 我们把`offer()`中的整个行为想象为跳台阶 result1的形式就像是 武侠小说中的越阶战斗！！！result2的形式就是一步一个脚印 每次平稳地去下一个台阶  

    - 我们想象一下 `offer()`最优的情况  10个线程同时`offer()` 

      每一个执行1方法成功的线程都没有(执行2方法或则执行3方法失败) 没关系 尾节点的更新终会成功

      每一个失败的线程都是去当前节点的next节点 p.next进行插入操作 在第9个线程(相当于我们上文中的线程2) 

      当第10个线程操作时 虽然它很可怜 一直排到最后 但是尾节点更新一下就越过了99阶!!!(不太恰当的地方请大佬们指点) 

    - ConcurrrntLinkedQueue 优点
      1. 能跃过一整段因为多线程在极短时间内`offer()`插入的节点 直接去尾节点 直接跨过去 
      2. 能抵达每一个相对于当前快照来说最新的next节点
      3. 高并发时 tail 和 p 相互配合 尽力去离当前尾节点 最近的地方

    -  ConcurrentLinkedQueue 缺点
      1.  CAS操作 虽然总会成功 但是竞争效率如果很低 不如用同步锁 采用CAS编写并发代码 都是大佬级别 难度高 不接地气(嘿嘿)
      2. 循环可能会带来额外的资源开销 

    

    