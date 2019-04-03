---
layout: post
title: java位移运算符
tags:
- foundation
categories: java
description: java位移运算符
---
java位移运算符包括<<（左移）、>>（带符号右移）和>>>（无符号右移）    

<!-- more -->

## <<（左移）  
**通用格式**：  
value << num  
**左移规则：**  
1,按二进制形式把所有的数字向左移动对应的位数，高位移出(舍弃)，低位的空位补零。  
2,如果移动的位数超过了该类型的最大位数，那么编译器会对移动的位数取模。如对int型移动33位，实际上只移动了332=1位。  
3,当左移的运算数是int 类型时，每移动1位它的第31位就要被移出并且丢弃；  
4,当左移的运算数是long 类型时，每移动1位它的第63位就要被移出并且丢弃。  
5,当左移的运算数是byte 和short类型时，将自动把这些类型扩大为 int 型。  
**数学意义**：  
在数字没有溢出的前提下，对于正数和负数，左移一位都相当于乘以2的1次方，左移n位就相当于乘以2的n次方  
**示例**：  
int 3的二进制: 0000 0000 0000 0000 0000 0000 0000 0011  
int 3左移2位的二进制：0000 0000 0000 0000 0000 0000 0000 1100  
## >>（带符号右移）  
**通用格式**：  
value >> num
**带符号右移规则：**  
1,按二进制形式把所有的数字向右移动对应的位数，低位移出(舍弃)，高位的空位补符号位，即正数补零，负数补1  
2,当右移的运算数是byte 和short类型时，将自动把这些类型扩大为 int 型。  
**数学意义：**  
右移一位相当于除2，右移n位相当于除以2的n次方。  
**示例：**  
int -11的二进制：11111111111111111111111111110101  
int -11带符号右移2位的二进制：11111111111111111111111111111101  
## >>>（无符号右移）  
**通用格式：**  
value >>> num  
**无符号右移规则：**  
同带符号右移相同。忽略了符号位扩展，0补最高位  
## 代码  
```
package com.tantao;

/**
 * Created by Administrator on 2019/4/3.
 */
public class Shift {
    public static void main(String[] args) {
        /**
         * 左移
         */
        int left = 3 << 2;
        System.out.println(left);//12

        int left_1 = 3 << 32;//溢出,编译器会对移动的位数取模，实际上没有移动
        System.out.println(left_1);//3

        byte b = 3;//byte只有8位
        int left_2 = b << 21;//当左移的运算数是byte 和short类型时，将自动把这些类型扩大为 int 型
        System.out.println(left_2);

        /**
         * 带符号右移
         * 向下取整
         */
        int right = -11 >> 2;
        System.out.println(right);//-3
        int right1 = 11 >> 2;
        System.out.println(right1);//2


        /**
         * 不带符号右移
         */
        int right_1 = -11  >>>  2;
        int right_2 = -11  >>  2;
        System.out.println(Integer.toBinaryString(right_1));//00111111111111111111111111111101(1073741821)
        System.out.println(Integer.toBinaryString(right_2));//11111111111111111111111111111101(-3)


    }
}
```
