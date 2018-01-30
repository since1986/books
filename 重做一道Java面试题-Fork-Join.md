---
title: 重做一道Java面试题(Fork/Join)
date: 2017-09-17 15:05:06
tags:
---
前几天参加了一场面试，当时有这莫一道题：
```
如何充分利用多核CPU，计算很大List中所有整数的和？
```
老实说，我当时并没有想出来具体该如何实现，只是有个大致的方向，肯定是`分治法`的思想；这两天我一直在尝试将这些当时没做出来的题想办法做出来，查了一些资料，看了若干文章，现在反过头来再来尝试解决一下这个题吧。

经过这两天的学习，我基本上搜集到了两种解这道题的思路：
1.用CyclicBarrier
这种方法，[有网友给出了详尽的解释](http://flysnow.iteye.com/blog/711162)，在此不再复述。

2.用Fork/Join
这个方法我是受到了这几篇文章的启发：

* [JDK 7 中的 Fork/Join 模式](https://www.ibm.com/developerworks/cn/java/j-lo-forkjoin/index.html)

* [我的Java开发学习之旅------>Java使用Fork/Join框架来并行执行任务](http://blog.csdn.net/ouyang_peng/article/details/46491217?utm_source=tuicool&utm_medium=referral)

* [Java 多线程（5）：Fork/Join 型线程池与 Work-Stealing 算法](https://segmentfault.com/a/1190000008140126?utm_source=tuicool&utm_medium=referral)

具体的看代码吧：
```
package com.github.since1986.test;

import java.util.concurrent.ExecutionException;
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.Future;
import java.util.concurrent.RecursiveTask;
import java.util.stream.IntStream;

/**
 * Created by since1986 on 2017/9/17.
 */
public class ForkJoinTest {

    public static void main(String... args) throws InterruptedException, ExecutionException {
        int[] array = IntStream.rangeClosed(0, 1_00_000_000).toArray(); //模拟一个“很大”的List，这里用直接用数组代替了（题目里其实也没说明白“很大”到底是什么概念，实际上太大了会OOM，但我觉得这道题主要考查的是对并发编程的基本思路吧，应该不会考察太深的，所以不用在意了，要是确实是要把怎么样避免OOM也考虑到的话暂时还没想到应该怎样解决）

        //简单粗暴的做法
        int sum = 0;
        for (int i = 0; i < array.length; i++) {
            sum += array[i];
        }
        System.out.println(sum);

        //Fork/Join的做法
        ForkJoinPool forkJoinPool = new ForkJoinPool(); //起一个数量等于可用CPU核数的池子（对应题目中“充分利用多核”）
        Task task = new Task(0, array.length, 10_000, array);

        Future<Integer> future = forkJoinPool.submit(task); //提交Task
        System.out.println(future.get()); //获得返回值

        forkJoinPool.shutdown(); //关闭池子
    }

    static class Task extends RecursiveTask<Integer> {

        public static final int DEFAULT_THRESHOLD = 1000;
        private int high, low;
        private int threshold;
        private int[] array;

        Task(int low, int high, int threshold, int[] array) {
            this.array = array;
            this.low = low;
            this.high = high;
            this.threshold = threshold; //任务划分的最小值（若为1000则含义是Fork到1000大小时就不再继续Fork了）
        }

        @Override
        protected Integer compute() {
            //System.out.println("low: " + low + "  high: " + high);
            if (high - low <= threshold) { //到了不能再Fork的阈值后直接循环累加返回
                int sum = 0;
                for (int i = low; i < high; i++) {
                    sum += array[i];
                }
                //System.out.println("sum: " + sum);
                return sum;
            } else { //没有到阈值的话，继续递归拆分任务为左任务和右任务（分治法的思想）
                int middle = (high - low) / 2 + low;
                //System.out.println("middle: " + middle);
                Task leftHandTask = new Task(low, middle, threshold, array); //左任务
                Task rightHandTask = new Task(middle, high, threshold, array); //右任务
                leftHandTask.fork(); //左任务还要继续拆，直到满足上边if里的阈值条件
                rightHandTask.fork(); //右任务也要继续拆，直到满足上边if里的阈值条件
                return leftHandTask.join() + rightHandTask.join(); //最后Join得到结果
            }
        }
    }
}
```