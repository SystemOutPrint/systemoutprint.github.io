---
layout:       post
title:        "Java中String的intern"
subtitle:     "Java中String的intern"
date:         2017-06-09 12:00:00
author:       "CaiJiahe"
header-img:   "img/tag-bg.jpg"
header-mask:  0.3
catalog:      true
multilingual: true
tags:
    - Java
---

## 0x01 复现问题
```
public class TestStrIntern {
	
	@Test
	public void test1() {
		String str1 = new String("1") + new String("11");
		str1.intern();
		String str2 = "111";
		Assert.assertTrue(str1 == str2);
	}
	
}
```
在JDK1.6和1.7上分别运行该段代码，发现在JDK1.6上输出false，在JDK1.7上输出true。

## 0x02 原因
__String.intern__：判断字符串常量池中是否存在该串，不存在则加入到字符串常量池中。<br>
JDK1.6中字符串常量池存储在永久代中，永久代是与java堆不同的内存区域。而JDK1.7中字符串常量池从永久代移到了java堆当中存储。<br>

