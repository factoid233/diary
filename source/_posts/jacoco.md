---
title: jacoco 增量代码覆盖率笔记 踩坑记录
---
## gitlab 分支比较的api(Repositories API Compare) 返回的响应中 $.diffs[*].diff 为空
  - 现象： 由于gitlab Diff limits的限制，即Maximum diff patch size最大只能为500KB，现像为如果新增一个文件行数特别多，返回的响应结果中 diff就是空的
  - 解决方案： 采用Jgit本地操作git仓库进行diff比较
