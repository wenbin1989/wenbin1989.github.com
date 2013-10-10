---
layout: post
title: "gentoo下使用nginx+fastcgi部署trac"
date: 2013-10-07 23:14
comments: true
categories: linux nginx
keywords: gentoo nginx trac fastcgi tracd flup 中文
description:
---

首先，运行
    emerge --ask nginx trac
安装所需应用。

需注意确保trac的USES标记中含有fastcgi与vhosts。
如果没有vhosts，trac安装完成后会自动调用webapp-config在localhost虚拟目录下创建trac应用，而本人更倾向于根据需要手动安装。

运行
    webapp-config -I -h trac trac 1.0.1
将trac安装在名为trac的虚拟目录下。
再运行
    trac-admin /path/to/project initenv
按提示创建trac环境。
详细介绍及trac环境的目录结构见[wiki](http://trac.edgewall.org/wiki/TracEnvironment)。

发布静态资源，以便通过nginx直接提供，提高效率：
    trac-admin /path/to/project deploy /var/www/trac
如配置为多项目，则需将`/var/www/trac/htdocs`下`site`与`common`目录移至`/var/www/trac/htdocs/project`下。

以nginx+fastcgi方式部署trac时，前端nginx做转发，后端fastcgi运行方式有两种：

*  利用spawn-fcgi等服务，管理运行trac虚拟目录下cgi-bin/trac.fcgi脚本。
*  以fcgi协议运行[tracd](http://trac.edgewall.org/wiki/TracStandalone)服务。

由于使用tracd配置最为便捷，其他人提及的又相对较少，所以本文主要介绍后者。

<!-- more -->

运行
    tracd --protocol=fcgi /path/to/project
进行测试，出现以下错误：
    ImportError: No module named flup.server.fcgi
查看错误代码，发现是因为当tracd运行协议为fcgi、scgi等时，实现依赖flup，而emeger安装trac时并没有自动安装该依赖。
运行
    emerge --ask flup
安装flup后，tracd正常运行，无错误发生。

编辑tracd配置文件(`/etc/conf.d/tracd`)，修改`TRACD_OPTS`段落为:
    TRACD_OPTS="--protocol=fcgi --env-parent-dir /path/to" (多项目)
或
    TRACD_OPTS="--protocol=fcgi -s /path/to/project" (单项目)
更多tracd选项请运行`tracd --help`查看。

启动tracd服务：
    /etc/init.d/tracd start
设置tracd开机自动运行：
    rc-update add tracd default

最后，配置nginx虚拟目录：

```
server {
	listen   80; ## listen for ipv4; this line is default and implied
	server_name trac.example.com;

	access_log /var/log/nginx/trac.access_log main;
	error_log /var/log/nginx/trac.error_log notice;

	# 单项目静态资源配置，多项目环境只需下一段
	# it makes sense to serve static resources through Nginx
	location ~ /chrome/ {
		rewrite /chrome/(.*) /$1 break;
		root /var/www/trac/htdocs;
		access_log        off;
		expires           max;
	}

	# 多项目静态资源配置，单项目环境只需上一段
	# it makes sense to serve static resources through Nginx
	location ~ /(.*?)/chrome/ {
		rewrite /(.*?)/chrome/(.*) /$1/$2 break;
		root /var/www/trac/htdocs;
		access_log        off;
		expires           max;
	}

	location / {
		fastcgi_pass localhost:8000;
		include fastcgi_params;
		fastcgi_param  SCRIPT_NAME        "";
		fastcgi_param  PATH_INFO          $fastcgi_script_name;
	}
}
```

注意，trac官方wiki中的配置方法为：
```
server {
	...
	# (Or ``^/some/prefix/(.*)``.
	if ($uri ~ ^/(.*)) {
		set $path_info /$1;
	}
	...
	location / {
		fastcgi_pass localhost:8000;
		include fastcgi_params;
		fastcgi_param  SCRIPT_NAME        "";
		fastcgi_param  PATH_INFO          $path_info;
	}
}
```
正常情况下nginx在传递请求参数时会做url encode，但该段配置用正则表达式匹配请求参数，从而使变量$1、即$path\_info不会被encode。
这种情况下，请求中存在中文的资源会无法正确解析，PATH\_INFO设置为$fastcgi\_script\_name则表现正常。

重启nginx，部署完成。
