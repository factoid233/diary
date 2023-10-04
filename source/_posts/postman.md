postman 旧版本 禁止更新
----------
## 使用旧版本
postman v10的版本以上部分导入受到了限制，必须要登录才行。登录后打开经常会卡死，还是本地使用比较好避免了关键信息泄露的风险
使用v9.31.28版本，本地Scratch Pad模式支持所有的导入
```text
https://dl.pstmn.io/download/version/9.31.28/win64
https://dl.pstmn.io/download/version/9.31.28/osx64
https://dl.pstmn.io/download/version/9.31.28/osx_arm64
https://dl.pstmn.io/download/version/9.31.28/linux_64
https://dl.pstmn.io/download/version/9.31.28/linux_arm64
```
## 禁止更新配置
1. 本地需要安装nodejs 
2. 新建一个存放postman源码的文件夹
3. windows系统，win+R 输入appdata 打开appdata目录，切换到`C:\Users\<your_account>\AppData\Local\Postman\app-9.31.28\resources`
在当前路径下打开cmd
4. 解压源码压缩包到目标文件夹
执行命令 `npx asar extract app.asar <destfolder>`
```cmd
# 示例
PS C:\Users\<your_account>\AppData\Local\Postman\app-9.31.28\resources> npx asar extract app.asar D:\Projects\nodejs\postman
```
5. 使用vscode 打开源码文件夹，找到services/AutoUpdaterService.js 文件，按照下方编辑后保存
```javascript
// 找到这个函数
isAppUpdateEnabled = function () {
  // App updates are not enabled for enterprise application
  if (enterpriseUtils.isEnterpriseApplication()) {
    return false;
  }
  // 增加这个返回
  return false;

  return _getInstallationType() !== LINUX_SNAP;
};
```
6. 打包修改后的源码，还在当前源码根目录执行
`npx asar pack . app.asar`
7. 替换原来的app.asar
```javascript
// 找到原始文件位置
PS C:\Users\<your_account>\AppData\Local\Postman\app-9.31.28\resources>
// 备份原来的app.asar 为app.asar.old
// 放入刚才打包好的 app.asar

```


