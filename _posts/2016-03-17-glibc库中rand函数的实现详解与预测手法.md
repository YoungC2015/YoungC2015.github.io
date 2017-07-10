---
layout: post
date:   2016-03-17 00:00:00
tags:   crypto ctf
---

<img src="{{ site.baseurl }}/images/glibc.jpg">
感谢0CTF，上交大这次举办了一个挺有深度的CTF比赛。

这次比赛有一道web题，题目很简单，大致如下：
```
<?php
include('config.php');
session_start();
	if($_SESSION['time'] && time() - $_SESSION['time'] > 60) {
	session_destroy();
	die('timeout');
} else {
	$_SESSION['time'] = time();
}
echo rand();
if (isset($_GET['go'])) {
	$_SESSION['rand'] = array();
	$i = 5;
	$d = '';
	while($i--){
		$r = (string)rand();
		$_SESSION['rand'][] = $r;
		$d .= $r;
	}
	echo md5($d);
} else if (isset($_GET['check'])) {
	if ($_GET['check'] === $_SESSION['rand']) {
		echo $flag;
	} else {
		echo 'die';
		session_destroy();
	}
} else {
	show_source(__FILE__);
}
```
完全没有别的考点，就是让你预测rand()的输出。看起来的确非常简单，名为rand()的函数实际上是伪随机函数生成器，是能够进行预测的。而且题目不限制提交次数，限制两次提交的时间间隔，连敌手攻击模型都固定了：不应该用暴力拆解（然而我用了），而是通过若干次请求之后能够收集数据预测之后的rand()函数输出。

国内查了很多资料都没找到关于rand()函数的实现细节。本人也拙，没有直接去找源码，而是听信某篇文章，进行爆破。结果运气不好，只能gg。

现在看完国外的博文之后才知道正确解法。原文详细阐述了glibc中rand函数的用法，英文好的可以直接读原文 [_http://www.mscs.dal.ca/~selinger/random/_](http://www.mscs.dal.ca/~selinger/random/) 。为了日后方便我将大致内容翻译整理如下：

 rand()由一个种子（singned int seed）进行初始化，生成的过程是非线性的。但是linux中man有些小误导，在初始化完成之后，随机数的生成就是线性的，实际上是以个线性移位反馈寄存器。给没学过密码学的安利一下最简单的情况，线性移位反馈寄存器实际就是把之前某几个特定位的输出取出来，进行一个操作（多半异或），然后作为现在的输出。常见于一些流密码的生成过程。这东西大家可以类比斐波那契序列进行理解。

如此，rand()可破，拿到足够的旧输出就行。那么rand()是如何工作的呢？rand()种子范围是0~2147483647（2**31-1），初始化对前34个内部向量r0~r33，操作如下：

	(1) r0 = s
	(2) ri = (16807 * (signed int) r(i-1)) mod 2147483647 (for i = 1...30)
	(3) ri = r(i-31) (for i = 31...33)
	
注意到乘16807这一步是在足够大的signed Int中进行的，所以在模运算之前不会发生溢出。而且在乘法之前，ri-1 也被转化为了 signed 32-bit int 。但是这个值唯一可能的负值出现在i=1的时候，即当s>=2**31时。由上能看出模运算就是0~2147483646之间的一个值，即便前一个数是负的。

所以，r34就是按如下的反馈循环进行的：

	(4) ri = (ri-3 + ri-31) mod 4294967296 (for i ≥ 34)
r0…r343 会被丢弃，第一个输出Oi实际上是：

(5) oi = r(i+344) >> 1
注意这个右移一位操作，丢弃了最后一bit，实际有效比特数为31位。

线性性质分析：

尽管最后一位被丢弃了，但是影响并不大，我们依然能够得到线性性质很好的输出序列。结合(4)、(5)式可得：

	(6.1) oi = o(i-31) + o(i-3) mod 2**31, for all i ≥ 31
	or  
	(6.2) oi = o(i-31) + o(i-3) + 1 mod 2**31, for all i ≥ 31
原文也给出一个C语言实现的简单版本random函数，附之于下：

	#include <stdio.h>

	#define MAX 1000
	#define seed 1

	main() {
	  int r[MAX];
	  int i;

	  r[0] = seed;
	  for (i=1; i<31; i++) {
		r[i] = (16807LL * r[i-1]) % 2147483647;
		if (r[i] < 0) {
		  r[i] += 2147483647;
		}
	  }
	  for (i=31; i<34; i++) {
		r[i] = r[i-31];
	  }
	  for (i=34; i<344; i++) {
		r[i] = r[i-31] + r[i-3];
	  }
	  for (i=344; i<MAX; i++) {
		r[i] = r[i-31] + r[i-3];
		printf("%d\n", ((unsigned int)r[i]) >> 1);
	  }
	}

<b>本文在[_free_](http://www.freebuf.com/articles/web/99093.html)已经发表，转载请留意