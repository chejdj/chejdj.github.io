---
title: ThreadPool用法解析
date: 2018-04-22 18:40:00
categories:
- Java
tags:
- Java
- 线程池
---

线程池好像每一次参加面试的时候，都会问道，自己之前确实没有使用过什么，今天大致结合看网上博客，发现了一篇写的很好的博客，略作修改，写下  
[原博客地址](https://www.jianshu.com/p/0e4a5e70bf0e)强烈推荐这个作者写的博客，思路很清晰！！！

<!--more-->

#### 什么是线程池  
线程池当中缓存了一定数量的线程，线程池实现了对于线程的管理和复用(最重要的就是实现对线程的复用，避免频繁创建线程和销毁线程带来的开销)，当然管理的可以实现线程执行的统一分配以及调优  
#### 必须知道的线程池核心参数  
1. **corePoolsize**  
核心线程数 ：  默认的情况下会一直存活下去(空闲的时候也存活)
2. **maximumPoolSize**  
线程池所能容纳的最大线程数 ：  当活动线程数到达该数值之后，后续的新任务将会阻塞 
3. **keepAliveTime**  
非核心线程 闲置超时时长 ：   超过该时长，非核心线程会被回收(当执行alllowThreadTimeout为true的时候，keepAliveTime同时用于核心线程)
4. **unit**
指定keepAliveTiem参数的时间单位 ： TimeUnit.MILLSECONDS(毫秒),TimeUnit.SECONDS(秒)，TimeUnit.MINUTES(分)等等
5. **workQeue**  
任务队列 :  通过线程池的execute()方法提交Runnable对象，存储在该参数中
6. **threadFactory**  
线程工厂 ： 为线程创建新线程  

这6个参数的配置决定了线程池的作用，在创建线程池类的时候传入  
ThreadPoolExecutor 是线程类的实现类，我们可以通过它实现自定义线程池  
```
Executor executor=new ThreadPoolExecutor(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue)
}
```  

#### 内部逻辑原理  
![图片](https://upload-images.jianshu.io/upload_images/944365-90cfd4951a587ebd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)  

#### ThreadPool使用流程  
1. 创建线程池  
创建时，通过配置线程池的参数，从而实现自己所需的线程池
```Executor threadPool = new ThreadPoolExecutor(
                                              CORE_POOL_SIZE,
                                              MAXIMUM_POOL_SIZE,
                                              KEEP_ALIVE,
                                              TimeUnit.SECONDS,
                                              sPoolWorkQueue,
                                              sThreadFactory
                                              );
```
2. 向线程池提交任务：execute（）
```// 说明：传入 Runnable对象
       threadPool.execute(new Runnable() {
            @Override
            public void run() {
                ... // 线程执行任务
            }
        });```

3. 关闭线程池shutdown() 
  `threadPool.shutdown();`  
关闭线程的原理  
a. 遍历线程池中的所有工作线程  
b. 逐个调用线程的interrupt（）中断线程（注：无法响应中断的任务可能永远无法终止）
也可调用shutdownNow（）关闭线程：threadPool.shutdownNow（） 
二者区别：  
shutdown：设置 线程池的状态 为 SHUTDOWN，然后中断所有没有正在执行任务的线程  
shutdownNow：设置线程池的状态 为 STOP，然后尝试停止所有的正在执行或暂停任务的线程，并返回等待执行任务的列表  
使用建议：一般调用shutdown（）关闭线程池；若任务不一定要执行完，则调用shutdownNow（）  

#### Java内部实现好了的4类功能线程池  
根据参数的不同配置，Java最常见的线程池有4类，其实这四个线程池都是通过`ThreadPoolExecutor`调整6个参数而创建而成    
1. 定长线程池（FixedThreadPool）  
2. 定时线程池（ScheduledThreadPool)  
3. 可缓存线程池（CachedThreadPool）  
4. 单线程化线程池（SingleThreadExecutor）  

##### 定长线程池(FixedThreadPool)  
1. 特点：只有核心线程 & 不会被回收、线程数量固定、任务队列无大小限制（超出的线程任务会在队列中等待）
2. 应用场景：控制线程最大并发数
3. 具体使用：通过 Executors.newFixedThreadPool() 创建  
示例：  
```
ExecutorService fixThreadPool=Executors.newFixedThreadPool(3);//最大并发量为3
	for(int i=0;i<6;i++){
		final int index=i;
		fixThreadPool.execute(new Runnable() {
			
			@Override
			public void run() {
				// TODO Auto-generated method stub
				try {
				System.out.println("当前线程"+Thread.currentThread()+"  执行任务"+index);
					Thread.sleep(2000);
				} catch (InterruptedException e) {
					// TODO Auto-generated catch block
					e.printStackTrace();
				}
			}
		});
	}
//这个程序最多只有三个线程在执行
```  

##### 定时线程池（ScheduledThreadPool ）  
1. 特点：核心线程数量固定、非核心线程数量无限制（闲置时马上回收）
2. 应用场景：执行定时 / 周期性 任务
3. 使用：通过Executors.newScheduledThreadPool()创建  
示例：  
```
ScheduledExecutorService scheduleThreadpool=Executors.newScheduledThreadPool(3);
	Runnable task=new Runnable() {
		
		@Override
		public void run() {
			// TODO Auto-generated method stub
			try {
				System.out.println("当前线程"+Thread.currentThread());
				Thread.sleep(5000);
			} catch (InterruptedException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			}
		}
	};
	scheduleThreadpool.schedule(task, 1,TimeUnit.SECONDS);//延迟一秒执行任务
	scheduleThreadpool.scheduleAtFixedRate(task, 10, 2000, TimeUnit.MILLISECONDS);//延迟 10ms 周期的执行
```   

##### 可缓存线程池（CachedThreadPool）  
1. 特点：只有非核心线程、线程数量不固定（可无限大）、灵活回收空闲线程（具备超时机制，全部回收时几乎不占系统资源）、新建线程（无线程可用时）源码中显示是通过60s回收线程
2. 任何线程任务到来都会立刻执行，不需要等待
3. 应用场景：执行大量、耗时少的线程任务
使用：通过Executors.newCachedThreadPool()创建  
示例：
```  
ExecutorService cachedThreadPool=Executors.newCachedThreadPool();
	 Runnable task=new Runnable() {
			
			@Override
			public void run() {
				// TODO Auto-generated method stub
				try {
					System.out.println("当前线程"+Thread.currentThread());
					Thread.sleep(5000);
				} catch (InterruptedException e) {
					// TODO Auto-generated catch block
					e.printStackTrace();
				}
			}
		};
		for(int i=0;i<10;i++)//直接创建10个线程执行	
			cachedThreadPool.execute(task);```

##### 单线程化线程池（SingleThreadExecutor）  
1. 特点：只有一个核心线程（保证所有任务按照指定顺序在一个线程中执行，不需要处理线程同步的问题）
2. 应用场景：不适合并发但可能引起IO阻塞性及影响UI线程响应的操作，如数据库操作，文件操作等
3. 使用：通过Executors.newSingleThreadExecutor()创建  
示例:  
```
ExecutorService singleThradPool=Executors.newSingleThreadExecutor();
        Runnable task=new Runnable() {
			
			@Override
			public void run() {
				// TODO Auto-generated method stub
				try {
					System.out.println("当前线程"+Thread.currentThread());
					Thread.sleep(5000);
				} catch (InterruptedException e) {
					// TODO Auto-generated catch block
					e.printStackTrace();
				}
			}
		};
		for(int i=0;i<10;i++)
			singleThradPool.execute(task);//只有一个线程执行，执行完之后才执行下一个任务
```  

#### 常见线程池 总结 & 对比  
![图片](https://upload-images.jianshu.io/upload_images/944365-5d6a2497f809d62a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)  

 