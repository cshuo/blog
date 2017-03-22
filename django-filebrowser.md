title: django1.8 服务器目录管理 -- grappelli + filebrowser
date: 2015-06-23
tags: django
---

FileBrowser是对Django 管理员界面的一个扩展，目的是：
+ 浏览服务器上的目录，包括上传、删除、编辑和重命名文件。
+ 使用FileBrowseField将图像/文档添加到django的models或database中。
+ 使用TinyMCE选择图像或文档。  

<!--more-->

### 安装要求：
* [Django1.8](http://www.djangoproject.com)
* Grappelli
> pip install django-grappelli
* Pillow
> Pillow是[PIL](http://www.oschina.net/p/pil)的替代版本，，如：改变图像大小，旋转图像，图像格式转换，色场空间转换，图像增强，直方图处理，插值和滤波等等。   
首先安装一些图像处理包：  
apt-get install libjpeg-dev  
apt-get install libfreetype6-dev  
apt-get install zlib1g-dev  
apt-get install libpng12-dev
然后安装Pillow  
#pip install pillow

### FileBrowser安装与配置
* 安装命令   
{% codeblock %}
# pip install django-filebrowser
{% endcodeblock %}
* 相关配置  
在Django项目setttings.py中INSTALLED_APPS模块添加相关应用(在django.contrib.admin之前添加):    
{% codeblock %}
INSTALLED_APPS = (
'grappelli',
'filebrowser',
'django.contrib.admin',
)
{% endcodeblock %}
在url模式匹配中添加filebrowser的url:     
{% codeblock %}
from filebrowser.sites import site
urlpatterns = patterns[
   url(r'^admin/filebrowser/', include(site.urls)),
   url(r'^grappelli/', include('grappelli.urls')),
   url(r'^admin/', include(admin.site.urls)),
]   
{% endcodeblock %}
一般情况下，默认media文件夹下的uploads为上传文件夹(替换的是uploads文件夹，替换后的文件夹还在media目录下)，如果需要自定义该目录，则需要添加如下配置项：          
{% codeblock %}
FILEBROWSER_DIRECTORY = 'yourdirectory/'        
{% endcodeblock %}
* 测试    
在项目根目录下运行命令：
{% codeblock %}
# python manage.py runserver 0.0.0.0:8000           
{% endcodeblock %}
访问/admin/filebrowser/browse/，可以管理上传文件（夹）。
