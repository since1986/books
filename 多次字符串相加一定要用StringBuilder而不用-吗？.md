---
title: 多次字符串相加一定要用StringBuilder而不用 + 吗？
date: 2017-11-06 15:06:37
tags:
---
今天在写一个读取Java class File并进行分析的Demo时，偶然发现了下面这个场景(基于oracle jdk 1.8.0_144)：

```
package test;

public class Test8 {

    String s1 = "111", s2 = "222", s3 = "333", s4 = "444";

    public String test() {
        return s1 + s2 + s3 + s4 + "5555" + "66666666666666666666666666" + "777" + new String("测试测试") + String.valueOf("test test") + "长字符串长字符串长字符串长字符串长字符串长字符串长字符串长字符串长字符串长字符串长字符串长字符串";
    }
}
```

这是一个很简单的类，只完成了字符串的 + 操作，我们查看对应生成的class文件的outline：

```
// class version 52.0 (52)
// access flags 0x21
public class test/Test8 {

  // compiled from: Test8.java

  // access flags 0x0
  Ljava/lang/String; s1

  // access flags 0x0
  Ljava/lang/String; s2

  // access flags 0x0
  Ljava/lang/String; s3

  // access flags 0x0
  Ljava/lang/String; s4

  // access flags 0x1
  public <init>()V
   L0
    LINENUMBER 3 L0
    ALOAD 0
    INVOKESPECIAL java/lang/Object.<init> ()V
   L1
    LINENUMBER 5 L1
    ALOAD 0
    LDC "111"
    PUTFIELD test/Test8.s1 : Ljava/lang/String;
    ALOAD 0
    LDC "222"
    PUTFIELD test/Test8.s2 : Ljava/lang/String;
    ALOAD 0
    LDC "333"
    PUTFIELD test/Test8.s3 : Ljava/lang/String;
    ALOAD 0
    LDC "444"
    PUTFIELD test/Test8.s4 : Ljava/lang/String;
    RETURN
   L2
    LOCALVARIABLE this Ltest/Test8; L0 L2 0
    MAXSTACK = 2
    MAXLOCALS = 1

  // access flags 0x1
  public test()Ljava/lang/String;
   L0
    LINENUMBER 8 L0
    NEW java/lang/StringBuilder
    DUP
    INVOKESPECIAL java/lang/StringBuilder.<init> ()V
    ALOAD 0
    GETFIELD test/Test8.s1 : Ljava/lang/String;
    INVOKEVIRTUAL java/lang/StringBuilder.append (Ljava/lang/String;)Ljava/lang/StringBuilder;
    ALOAD 0
    GETFIELD test/Test8.s2 : Ljava/lang/String;
    INVOKEVIRTUAL java/lang/StringBuilder.append (Ljava/lang/String;)Ljava/lang/StringBuilder;
    ALOAD 0
    GETFIELD test/Test8.s3 : Ljava/lang/String;
    INVOKEVIRTUAL java/lang/StringBuilder.append (Ljava/lang/String;)Ljava/lang/StringBuilder;
    ALOAD 0
    GETFIELD test/Test8.s4 : Ljava/lang/String;
    INVOKEVIRTUAL java/lang/StringBuilder.append (Ljava/lang/String;)Ljava/lang/StringBuilder;
    LDC "555566666666666666666666666666777"
    INVOKEVIRTUAL java/lang/StringBuilder.append (Ljava/lang/String;)Ljava/lang/StringBuilder;
    NEW java/lang/String
    DUP
    LDC "\u6d4b\u8bd5\u6d4b\u8bd5"
    INVOKESPECIAL java/lang/String.<init> (Ljava/lang/String;)V
    INVOKEVIRTUAL java/lang/StringBuilder.append (Ljava/lang/String;)Ljava/lang/StringBuilder;
    LDC "test test"
    INVOKESTATIC java/lang/String.valueOf (Ljava/lang/Object;)Ljava/lang/String;
    INVOKEVIRTUAL java/lang/StringBuilder.append (Ljava/lang/String;)Ljava/lang/StringBuilder;
    LDC "\u957f\u5b57\u7b26\u4e32\u957f\u5b57\u7b26\u4e32\u957f\u5b57\u7b26\u4e32\u957f\u5b57\u7b26\u4e32\u957f\u5b57\u7b26\u4e32\u957f\u5b57\u7b26\u4e32\u957f\u5b57\u7b26\u4e32\u957f\u5b57\u7b26\u4e32\u957f\u5b57\u7b26\u4e32\u957f\u5b57\u7b26\u4e32\u957f\u5b57\u7b26\u4e32\u957f\u5b57\u7b26\u4e32"
    INVOKEVIRTUAL java/lang/StringBuilder.append (Ljava/lang/String;)Ljava/lang/StringBuilder;
    INVOKEVIRTUAL java/lang/StringBuilder.toString ()Ljava/lang/String;
    ARETURN
   L1
    LOCALVARIABLE this Ltest/Test8; L0 L1 0
    MAXSTACK = 4
    MAXLOCALS = 1
}

```

请注意这段：
```
// access flags 0x1
  public test()Ljava/lang/String;
   L0
    LINENUMBER 8 L0
    NEW java/lang/StringBuilder
    DUP
    INVOKESPECIAL java/lang/StringBuilder.<init> ()V
    ALOAD 0
    GETFIELD test/Test8.s1 : Ljava/lang/String;
    INVOKEVIRTUAL java/lang/StringBuilder.append (Ljava/lang/String;)Ljava/lang/StringBuilder;
    ALOAD 0
```
我们可以看到，即便我们没有显式的使用`StringBuilder`，实际上编译器也会隐式的将我们的 + 运算符优化为`StringBuilder`的`append()`操作；另外，其中字符串常量的相加这里，也就是 `"5555" + "66666666666666666666666666" + "777"` 这里对应的操作是：
```
GETFIELD test/Test8.s4 : Ljava/lang/String;
    INVOKEVIRTUAL java/lang/StringBuilder.append (Ljava/lang/String;)Ljava/lang/StringBuilder;
    LDC "555566666666666666666666666666777"
```
直接被合并在了一次(具体这是什么操作我不是很明白)

这时候我就想起来，原来一直被教导的“字符串相加一定要用StringBuilder而不要用 + ”真的正确吗？这个值得深思。

---
2017-01-22更新：
```
GETFIELD test/Test8.s4 : Ljava/lang/String;
    INVOKEVIRTUAL java/lang/StringBuilder.append (Ljava/lang/String;)Ljava/lang/StringBuilder;
    LDC "555566666666666666666666666666777"
```
> 直接被合并在了一次(具体这是什么操作我不是很明白)

这里可能是编译器做了“公共子表达式消除”这个优化操作

---
2017-01-25更新：

在循环+=的情况下，编译器也会做优化工作的，但是IDE仍然会给出警告，不知道编译器的优化是否在所有情况下均会触发（有待继续学习）
```
public static void main(String[] args) {
    String s = "";
    for (int i = 0; i < 10000; i++) {
        int int_ = new Random().nextInt();
        s += int_;
    }
    System.out.println(s);
}
```

{% asset_img 1612b6b7a730a9e8.png %}

{% asset_img 1612b6b9c7da64cf.png %}