---
title: 进制转换
author: zhangzangqian
date: 2017-06-17 19:00:00 +0800
categories: [技术]
tags: [radix]
math: true
mermaid: true
---

## 前言

- 数制是用一组固定的符号和统一的规则来表示数值的方法
- 计算机底层是用的是数制是二进制
- 用 Java 编程使用的是十进制。Java 底层仍使用二进制
- 计算机常用的数制还有八进制和十六进制

## 进制

### 十进制

> 十进制的基本数字是0～9，逢10进位。10称作“基数”，10^n （10的n次幂）被称作“权”。
{: .prompt-tip}

```
10000=1*10^4
1000=1*10^3
100=1*10^2
10=1*10^1
1=1*10^0
```

### 二进制

> 二进制的基本数字数0～1，逢2进位。基数是2，权为 2^n。
{: .prompt-tip}

```
10000=1*2^4
1000=1*2^3
100=1*2^2
10=1*2^1
1=1*2^0
```

### 十六进制

>基本数字:0～9，A~F,A表示10，以此类推，逢16进位，基数16，权是 16^n。十六进制是二进制的简写，方便专业人员书写二进制数据。在 Java 代码中十六进制数用0X或0x做前缀。
{: .prompt-tip}

```
1000=1*16^3
100=1*16^2
10=1*16^1
1=1*16^0
```

## 进制转换

### 十六进制

#### 转换为十进制

```
5E(16)
= 5*16^1 + 14*16^0
= 94
```

#### 转换为二进制

> 十六进制中的一位代表二进制中的四位，忽略前面的0。
{: .prompt-tip}

```
5E(16) = 1011110(2)
5 = 1*2^2 + 1*2^0 = 0101
E = 1*2^3 + 1*2^2 + 1*2^1
= 1110
```

### 十进制

#### 转为十六进制

> 十进制转为二进制便不断的除以二，保留余数，直到商为0，将所有余数倒序排列。
{: .prompt-tip}

```
138=8A(16)
138/16=8.......A
8/16=0........8
```

#### 转为二进制

> 十进制转为二进制便不断的除以二，保留余数，直到商为0，将所有余数倒序排列。
{: .prompt-tip}

```
13 = 1101(2)
13/2＝6........1
6/2=3........0
3/2=1........1
1/2=0........1
```

### 二进制

#### 转为十进制

```
100010(2)
=1*2^5 + 1^2^1
=34
```

#### 转为十六进制

> 可将二进制转为十进制，然后由十进制转为十六进制。
{: .prompt-tip}

```
100010(2)
=1*2^5 + 1^2^1
=34
34／16=2......2
2/16=0......2
```