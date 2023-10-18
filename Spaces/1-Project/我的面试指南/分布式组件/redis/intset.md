---
title: intset
date created: 2023-10-18
date modified: 2023-10-18
---

# intset

> [!TIP] 尽可能高效地利用内存💡  
>  在redis的set数据类型，底层主要使用intset和hashtable.为尽可能利用内存，如果是整形，并且不超过指定长度，则int更省内容。  
>  **元素65535，如果用字符串存储的话需要5个字节，而按整型存储则只需2个字节即可，从这方面来看，将整型转化为字符串存储并不划算**，而 intset 存储整型数据优势明显.  
>  用hashtable的话，相当于key是 [[sds]],value是NULL

![[Extras/Draws/redis-数据类型.md#^frame=eJlwPA3-ds9YoMr9Y_bVE]]  
和sds很像，

+ encoding,编码类型，决定每个元素占用几个字节
	+ int16_t：每个元素占 2 个字节。
	+ int32_t：每个元素占 4 个字节。
	+ int64_t：每个元素占 8 个字节
+ length: 元素个数
+ contents: 存储具体元素。根据 encoding 字段决定多少个字节表示一个元素
