### 1.开发起始分支

从master分支clone出开发分支

### 2.开发分支命名

功能分支f_feature_xxx

修复分支f_hotfix_xxx

### 3.开发完成，联调 (暂无联调环境)

将本地开发分支合并到f_common_dev

f_common_dev包版本固定：1.0.0-DEV-SNAPSHOT

如有修改，修改本地分支代码并提交，再合并到公共分支。勿修改公共分支。

### 4.联调完成，提测

将本地开发分支合并到f_common_test

发起code review(将需求wiki，git MR发送给相关review人)

f_common_test包版本固定：1.0.0-TEST-SNAPSHOT

如有修改，修改本地分支代码并提交，再合并到公共分支。勿修改公共分支。

### 5.测试完成，发布线上RELEASE包

拉取master最新代码，将master合并到本地开发分支，更新包版本为RELEASE版本，deploy

发起code review(将需求wiki，git MR发送给相关review人)

### 6.上线

将本地开发分支合并到master，打tag，线上发布新tag

![](http://image.clickear.top/20220908182829.png)
