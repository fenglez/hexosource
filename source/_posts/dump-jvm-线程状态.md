---
title: dump jvm 线程状态
date: 2017-09-05 19:43:55
tags:
---
#### dump下来jvm状态
``` bash
/Library/Java/JavaVirtualMachines/jdk1.8.0_131.jdk/Contents/Home/bin/jstack 6366 > ./dump17
``` 
#### 查看现在状态
``` bash
grep java.lang.Thread.State dump17 | awk "{print $2$3$4$5}" | sort |uniq -c

  17    java.lang.Thread.State: RUNNABLE
   4    java.lang.Thread.State: TIMED_WAITING (on object monitor)
  35    java.lang.Thread.State: TIMED_WAITING (parking)
  10    java.lang.Thread.State: TIMED_WAITING (sleeping)
   2    java.lang.Thread.State: WAITING (on object monitor)
  11    java.lang.Thread.State: WAITING (parking)
  
``` 
####死锁
``` bash
 死锁的状态：java.lang.Thread.State:BLOCKED
 死锁避免：
    避免一个线程同时获取多个锁
    避免一个线程在锁内同时占用多个资源，尽量保证每个锁只占用一个资源
    尝试使用定时锁，使用lock.tryLock（timeout）来替代使用内部锁机制
    对于数据库锁，加解锁必须在一个数据库连接里，否则会出现解锁失败的情况
 ``` 
 ####volatile
 ``` bash
        有该变量修饰的共享变量进行写操作的时候会多出lock add1 $0...汇编代码，lock前缀的指令在多核处理
    器下回引发两件事情。
    1：将当前处理器缓存行的数据回写到系统内存；
    2：这个写回内存的操作会使在其他cpu缓存了该内存地址的数据无效。
        为了提高处理速度，处理器不直接和内存进行通信，而是将系统内存的数据读到内部缓存后再进行操作，但操
    做完了不知道何时会写到内存。如果对声明了volatile的变量进行写操作，jvm就会向处理器发送一条lock前缀的
    指令，将这个变量所在缓存行的数据写回到系统内存。在多处理器下，为了保证各个处理器的缓存是一致的，就会实
    现缓存一致性协议，每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是不是过期了，当处理器发现自己
    缓存行对应的内存地址被修改，就会将当前处理器的缓存行设置成无效状态，当处理器对这个数据进行修改操作的时
    候，会重新从系统内存中把数据读到处理器缓存。
 ``` 
 ####synchronized
  ``` bash
  1：对于普通方法，锁是当前实例对象
  2：对于静态方法，锁是当前类的Class对象
  3：对于同步方法块，锁是synchronized括号里配置的对象
  对于一个线程访问同步代码，必须先得到锁，退出或抛出异常时抛出锁。
  ``` 