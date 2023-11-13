---
title: snappy
date created: 2023-11-10
date modified: 2023-11-10
---

## 代码实现

[snappy](https://github.com/xerial/snappy-java)

```
<dependency>
  <groupId>org.xerial.snappy</groupId>
  <artifactId>snappy-java</artifactId>
  <version>(version)</version>
  <type>jar</type>
  <scope>compile</scope>
</dependency>
```

```
public class SnappyCompress implements Compress {

    public byte[] compress(byte[] array) throws IOException {
        if (array == null) {
            return null;
        }
        return Snappy.compress(array);
    }

    public byte[] unCompress(byte[] array) throws IOException {
        if (array == null) {
            return null;
        }
        return Snappy.uncompress(array);
    }
}
```
