---
layout: post
title: Configure Xdebug + Laravel Homestead + VS Code
---

在 Homestead 中配置 xdebug 中其实蛮简单的，不过有很多地方容易遗漏或者搞错，这里记录下以后用到不用找来找去了。

## Configure Laravel Homestead

1. 找到虚拟机的网关，稍后用到
```
netstat -rn | grep "^0.0.0.0 " | cut -d " " -f10
```
2. 找到要配置的项目在 nginx 中使用的 fpm 版本
```
grep "php.*-fpm" /etc/nginx/sites-available/*
```
3. 设置 xdebug.ini，下面版本号需要替换为上一步找到的 fpm 版本号
```
sudo vim /etc/php/版本号/fpm/conf.d/20-xdebug.ini
```
4. 在文件中添加以下配置, 有的话不用重复加
```
;下面这项一般会有，没有的话需要补充
;xdebug.remote_enable=1
xdebug.remote_autostart=1
;这里的 IP 地址就是 #1 中得到的 IP 地址
xdebug.remote_host=10.0.2.2
```
5. 重启 fpm
```
sudo service php版本-fpm
```

## Configure VS Code

1. 安装插件 [PHP Debug plugin](https://marketplace.visualstudio.com/items?itemName=felixfbecker.php-debug)
2. 打开 launch.json，没有的话在配置中可以新建
3. 添加下面内容在 `configurations` 中，并配置 `pathMappings`
```
{
    "name": "Listen for XDebug on Homestead",
    "type": "php",
    "request": "launch",
    "pathMappings": {
        "homestead 中代码根目录完整路径": "真机代码的完整路径"
    },
    "port": 9000
}
```

## Done