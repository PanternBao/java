参考链接：http://www.rdrop.com/users/paulmck/scalability/paper/whymb.2010.06.07c.pdf



问题：

1. 多cpu中，对一个共享变量，有且只会有一个线程访问，这个变量需要用volatile保证可见性嘛？
2. 缓存一致性协议已经可以保证某个内存的可见性，为什么还需要volatile关键字？



https://stackoverflow.com/questions/38447226/atomicity-on-x86/38465341#38465341

https://stackoverflow.com/questions/39393850/can-num-be-atomic-for-int-num



并发导致的问题：单核和多核

可见性？

原子性？

数据冗余 -》一致性的问题



解决方法：锁

硬件层面

锁总线

锁缓存 -》缓存一致性算法

## 什么是重排序

```
public class PossibleReordering { static int x = 0, y = 0; static int a = 0, b = 0; 	public static void main(String[] args) throws InterruptedException {    	Thread one = new Thread(new Runnable() {        	public void run() {            	a = 1;            	x = b;        	}    	});        	Thread other = new Thread(new Runnable() {        	public void run() {            	b = 1;            	y = a;        	}    	});    	one.start();other.start();    	one.join();other.join();    	System.out.println(“(” + x + “,” + y + “)”); } }
```

代码的运行结果?



处理器重排：大多数现代微处理器都会采用将指令乱序执行（out-of-order execution，简称OoOE或OOE）的方法，在条件允许的情况下，直接运行当前有能力立即执行的后续指令，避开获取下一条指令所需数据时造成的等待。通过乱序执行的技术，处理器可以大大提高执行效率。

编译器重排：常见的Java运行时环境的JIT编译器也会做指令重排序操作，即生成的机器指令与字节码指令顺序不一致。

编译期重排序的典型就是通过调整指令顺序，在不改变程序语义的前提下，尽可能减少寄存器的读取、存储次数，充分复用寄存器的存储值。

假设第一条指令计算一个值赋给变量A并存放在寄存器中，第二条指令与A无关但需要占用寄存器（假设它将占用A所在的那个寄存器），第三条指令使用A的值且与第二条指令无关。那么如果按照顺序一致性模型，A在第一条指令执行过后被放入寄存器，在第二条指令执行时A不再存在，第三条指令执行时A重新被读入寄存器，而这个过程中，A的值没有发生变化。通常编译器都会交换第二和第三条指令的位置，这样第一条指令结束时A存在于寄存器中，接下来可以直接从寄存器中读取A的值，降低了重复读取的开销。



**编译期重排序**和**运行期重排序**

## as-if-serial语义

As-if-serial语义的意思是，所有的动作(Action)5都可以为了优化而被重排序，但是必须保证它们重排序后的结果和程序代码本身的应有结果是一致的。Java编译器、运行时和处理器都会保证单线程下的as-if-serial语义。