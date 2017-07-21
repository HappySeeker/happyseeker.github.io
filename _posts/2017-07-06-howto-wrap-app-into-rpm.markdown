---
layout: post
title:  "如何将Window应用封装成rpm包"
date:   2017-07-06 10:22:43
author: JiangBiao
categories: Develop
---

#  场景

使用wine适配Windows应用程序后，需要将相应的Windows应用封装成rpm包的形式，便于使用包管理工具安装，安装后需要在桌面创建图标，用户点击后即可使用wine启动应用程序。

本文描述制作相应rpm包的基本方法。
  
# RPM包内容

RPM包中主要包含两部分内容：

1. Source
2. Spec文件

## Source

Source中需要提供3部分内容：

1. 包含Windows应用程序的所有文件的压缩包。
2. Desktop文件。
3. 应用程序对应的Icon文件。

已EDiary为例：SOURCES目录内容为：

		[root@localhost SOURCES]# ls
		EDiary.desktop  EDiary.png  wine-ediary-1.tar.gz

其中wine-ediary-1.tar.gz解压后内容为：

		[root@localhost SOURCES]# ls wine-ediary-1
		EDiary.desktop  EDiary.png  EDiary.tar.gz

其中，EDiary.tar.gz即为应用程序压缩包。

EDiary.desktop和EDiary.png与上级目录中的两个文件内容一样，只是因为rpm包装方便需要。


### 应用程序压缩包

通常将应用程序在Window系统中的安装目录打包即可，以EDiary为例，通常安装于C:\Program Files\Ediary<Ver>\目录，将此目录打包即可。已EDiary为例，其内容如下：

		[root@localhost wine-ediary-1]# ls EDiary
		Background  EDiary.exe  FileConv.exe  Sample.edf  unins000.dat
		EDiary.chm  EDiary.url  Readme.txt    Skins       unins000.exe

### Desktop文件

 用于显示桌面图标，并启动应用程序。已EDiary为例，其内容如下：
 
		 [Desktop Entry]
		Categories=System;
		Name=EDiary
		Exec=env WINEPREFIX="/root/.wine" /usr/bin/wine C:\\\\Program\\ Files\\\\EDiary\\\\EDiary.exe 
		Type=Application
		StartupNotify=true
		Comment=wine ediary
		Icon=EDiary
		Name[zh_CN]=EDiary
		
内容属自解释型，就不罗嗦了。

### icon文件

通常应用程序都会自带相应的icon文件，用于在桌面和其他场景(比如任务栏)中显示，最简单的情况，提供一个48x48大小的即可，通常在应用程序安装包中可以找到相应的文件。

## Spec文件

Spec文件中定义rpm包的内容，对于Windows程序的封装，更重要的是post脚本，在其中执行真正的安装和配置操作。以EDiary为例，其内容如下：

		[root@localhost SOURCES]# cat ../SPECS/wine-ediary.spec 
		Summary:        EDiary tool running with wine. 
		Name:           wine-ediary
		Version:        1
		Release:        0.1
		License:        GPL
		Group:          System Environment/Base
		URL:            http://www.gd-linux.com/
		Source0:         wine-ediary-%{version}.tar.gz
		Source1:         EDiary.desktop
		Source2:         EDiary.png
		Requires:	wine
		Requires:       winbase
		BuildArch:      noarch

		%description
		wine-ediary is EDiary tool running with wine.


		%prep
		%setup 

		%build

		%install
		mkdir -p $RPM_BUILD_ROOT/tmp/.winApp/
		mkdir -p $RPM_BUILD_ROOT/usr/share/applications/
		mkdir -p $RPM_BUILD_ROOT/usr/share/icons/hicolor/64x64/apps/

		mkdir -p %{buildroot}/root/桌面/
		mkdir -p %{buildroot}/root/.local/share/icons/hicolor/48x48/apps/
		cp -f EDiary.tar.gz $RPM_BUILD_ROOT/tmp/.winApp/
		cp -f EDiary.desktop %{buildroot}/root/桌面/
		cp -f EDiary.png %{buildroot}/root/.local/share/icons/hicolor/48x48/apps
		cp -rf %{SOURCE1}  $RPM_BUILD_ROOT/usr/share/applications/
		cp -rf %{SOURCE2}  $RPM_BUILD_ROOT/usr/share/icons/hicolor/64x64/apps/

		%files
		/tmp/.winApp/EDiary.tar.gz
		/root/桌面/EDiary.desktop
		/root/.local/share/icons/hicolor/48x48/apps/EDiary.png
		/usr/share/applications/EDiary.desktop
		/usr/share/icons/hicolor/64x64/apps/EDiary.png
		%post
		cd /tmp/.winApp/
		if [ -d /root/.wine/drive_c/Program\ Files ];then
			tar xvf EDiary.tar.gz -C /root/.wine/drive_c/Program\ Files/ >/dev/null
		fi

		gtk-update-icon-cache --force /usr/share/icons/hicolor/  1>/dev/null 2>&1

		%preun

		%clean
		[ "$RPM_BUILD_ROOT" != "/" ] && rm -rf "$RPM_BUILD_ROOT"

		%postun
		rm -rf /root/桌面/EDiary.desktop 
		if [ -d /root/.wine/drive_c/Program\ Files/EDiary ];then
			rm -rf /root/.wine/drive_c/Program\ Files/EDiary
		fi

		%changelog
		* Wed Jul 5 2017 Jiang Biao <jiang.biao@zte.com.cn> - 1-0
		- Initial wine EDiary

直接都能看懂，不过多解释，注意几个关键点：

1. 安装包只是将Windows程序压缩包安装到/tmp目录中，临时存放。真正的安装操作在%post脚本中，其中，会将将其解压到wine的工作目录中。
2. 图标仅用了48x48的，如果有其它size，可以参考做类似处理。

# Easyeast way

看起来步骤并不简单，其实，如果要封装一个新的app，最简单的方法，就是copy已有的封装过的rpm包源码目录，然后做相应的修改即可，简单方便，但需要细心。

