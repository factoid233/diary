---
title: jacoco 增量代码覆盖率笔记 踩坑记录
---
## 1. gitlab 分支比较的api(Repositories API Compare) 返回的响应中 $.diffs[*].diff 为空
  - 现象： 由于gitlab Diff limits的限制，即Maximum diff patch size最大只能为500KB，现像为如果新增一个文件行数特别多，返回的响应结果中 diff就是空的
    > This endpoint can be accessed without authentication if the repository is publicly accessible. Note that diffs could have an empty diff string if diff limits are reached.  <br/> [gitlab Compare Api链接](https://docs.gitlab.com/ee/api/repositories.html#compare-branches-tags-or-commits)<br/>
    > 
      | Value                       | Definition                                    | Default value | Maximum value |
      | :-------------------------- | :-------------------------------------------- | :-----------: | :-----------: |
      | **Maximum diff patch size** | The total size, in bytes, of the entire diff. |    200 KB     |    500 KB     |
      | **Maximum diff files**      | The total number of files changed in a diff.  |     1000      |     3000      |
      | **Maximum diff lines**      | The total number of lines changed in a diff.  |    50,000     |    100,000    |
     > [gitlab diff limit链接](https://docs.gitlab.com/ee/user/admin_area/diff_limits.html)
  - 解决方案： 采用Jgit本地操作git仓库进行diff比较
## 2. java编译提示 Invalid signature file digest for Manifest main attributes 
  - 解决方案： 删除jar包中的 *.SF *.DSA  *.RSA文件  
