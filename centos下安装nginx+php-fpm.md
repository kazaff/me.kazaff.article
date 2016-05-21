好久没装过PHP环境了，好久没有手动配置LNMP环境了，今天就让我头疼了一把！

不过随着时间的推移，yum的源里越来越多的库可以直接使用了，现在自己在配置nginx和php环境就不再需要源码编译，也不再需要往yum中添加啥源了，直接就可以通过下面的命令完成安装：

```
yum install -y nginx php php-fpm
```

若系统之前yum安装过php，可以先卸载了：

```
yum remove httpd* php*
```

安装完毕后，需要稍微修改一下配置文件来完成最后的工作，php-fpm需要修改一下权限：

```
vi /etc/php-fpm.d/www.conf
```
将`Unix user/group of processes`改成你os对应的设置，例如：

```
user = www 
group = www  
```

然后需要开启nginx对应的php配置项：

```
vi /etc/nginx/conf.d/default.conf
```

开启下面这部分配置：

```
location ~ \.php$ {  
    include /etc/nginx/fastcgi_params;  
    fastcgi_pass  127.0.0.1:9000;  
    fastcgi_index index.php;  
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;  
}  
```

一切就绪，就可以分别开启对应服务了：

```
/etc/init.d/php-fpm restart 
/etc/init.d/nginx restart  
```

### nginx File not found 错误

这个时候如果你访问本地的nginx服务，如果看到了"File not found"错误提醒，原因多半是：**php-fpm进程找不到SCRIPT_FILENAME配置的要执行的.php文件**。

由于默认nginx将`root`参数放在了`location`内部，所以你得注意一下对应设置的文件目录是否正确，或者推荐你将`root`参数从`location`中移到`server`中，这样所有的子`location`将使用统一的web根目录。

此外，为了避免一些php cms系统的默认行为，你还是最好将`index`参数里增加`index.php`，来适配系统的默认首页匹配规范，避免不必要的麻烦。