---
title: nginx 笔记
---
### 使用docker 方式部署nginx  docker映射端口不一致时无法访问
#### 现象 
  - 访问 http://xxx.com/home/ 带斜杠是正常的， 
  - 访问 http://xxx.com/home 不带斜杠访问时nginx时会进行重定向，导致无法访问

#### 原理分析:
  - 当访问的 uri 最后带斜杠时，例如 http://xxx.com/home/，查找 home 下的 index 页面，存在就返回；若不存在且未开启自动索引目录选项（autoindex on）则报 403 错误
  - 当访问的 uri 最后不带斜杠时，例如 http://xxx.com/home ，会先查找 home 文件，存在就返回；若存在 home 文件夹，会在末尾加上一个斜杠并产生 301 跳转
  
#### 解决方案
对不带斜杠的 $uri 精确匹配进行rewrite 到一个可以找到的location
```
listen 7000;
location /home {
            alias /usr/share/nginx/html;
            index index.html;
            if ($uri = /home){
                rewrite .* /index.html;
            }
       }
location / {
            root /usr/share/nginx/html;
            index index.html;
       }
```
