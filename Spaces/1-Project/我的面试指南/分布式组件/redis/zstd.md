---
title: zstd
date created: 2023-11-10
date modified: 2023-11-10
---

Zstd 全称叫 Zstandard，是一个提供高压缩比的快速压缩算法，主要实现的编程语言为 C，是 Facebook 的 Yann Collet 于2016年发布的。它还为小数据提供了一种特殊的模式，称为字典压缩。参考库提供了非常广泛的速度/压缩权衡，并由一个非常快的解码器支持。Zstandard库是使用BSD许可证作为开源软件提供的。它的格式是稳定的，并发布为IETF RFC 8878。

参考资料：

官网：[https://facebook.github.io/zstd/#other-languages](https://facebook.github.io/zstd/#other-languages)

美团：[https://tech.meituan.com/2021/01/07/pack-gzip-zstd-lz4.html](https://tech.meituan.com/2021/01/07/pack-gzip-zstd-lz4.html)

注意事项：

1、压缩的数据尽量大于256字节，太小了压缩无意义  
2、Zstandard 的默认压缩级别为 3，但可以设置 1-22 之间的任何值。较高的设置将产生较小的压缩存档，但代价是压缩速度变慢

## 代码实现

[GitHub - luben/zstd-jni: JNI binding for Zstd](https://github.com/luben/zstd-jni)

``` xml
<dependency>
    <groupId>com.github.luben</groupId>
    <artifactId>zstd-jni</artifactId>
    <version>1.9.6</version>
</dependency>
```

```java
public class ZstdCompressor implements Compressor {  
  
    public final static Compressor DEFAULT = new ZstdCompressor();  
    /**  
     *压缩级别，默认3  
     */    private int level = 3;  
  
    private ZstdCompressor() {}  
  
    public ZstdCompressor(int level) {  
        this.level = level;  
    }  
  
    @Override  
    public byte[] compress(byte[] src) {  
        if(src == null || src.length == 0) {  
            return src;  
        }  
        return Zstd.compress(src, level);  
    }  
  
    @Override  
    public byte[] deCompress(byte[] src) {  
        if(src == null || src.length == 0) {  
            return src;  
        }  
        return Zstd.decompress(src, (int)Zstd.decompressedSize(src));  
    }  
}
```
