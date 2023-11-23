---
title: redis-如何排查慢查询
date created: 2023-11-23
date modified: 2023-11-23
---

类似mysql，大于`slowlog-log-slower-than`,进行慢查询记录。最多存`slowlog-max-len`条记录

## 资料

[22 第11～21讲课后思考题答案及常见问题答疑](https://learn.lianglianglee.com/%e4%b8%93%e6%a0%8f/Redis%20%e6%a0%b8%e5%bf%83%e6%8a%80%e6%9c%af%e4%b8%8e%e5%ae%9e%e6%88%98/22%20%20%e7%ac%ac11%ef%bd%9e21%e8%ae%b2%e8%af%be%e5%90%8e%e6%80%9d%e8%80%83%e9%a2%98%e7%ad%94%e6%a1%88%e5%8f%8a%e5%b8%b8%e8%a7%81%e9%97%ae%e9%a2%98%e7%ad%94%e7%96%91.md)
