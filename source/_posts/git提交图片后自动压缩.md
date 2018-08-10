---
title: git提交图片后自动压缩
tags: [git,编译,linux]
categories: [git]
date: 2018-08-09 20:07:42
description: 前言、压缩工具、git hook、git hook 安装工具
---

# 前言

一般而言，当我们做需求需要在项目中加入图片时，都会去[tinyPng](https://tinypng.com/)上进行图片压缩
tinyPng使用的是有损压缩的方法，支持png与jpg格式，压缩完的图片看起来不会产生太大的差别，但每次需求的图片感觉都能压缩几百k的样子
虽然Android Studio编译出来的release版本apk本来就会对资源文件进行压缩，但我们把原图压缩一遍再去提交还是能节省不少大小
考虑到每次提交图片文件前都需要去打开网页进行压缩比较繁琐，因此博主考虑使用git hook来编写一套自动压缩工具

# 压缩工具

## 压缩图片网站api

最开始，博主考虑的是使用图片压缩网站提供的api来实现图片压缩，如[tinyPng的api](https://tinypng.com/developers)
然而，这些网站提供的api不仅有使用次数限制（tinyPng是每个月500张，多出的要收钱），而且每次压缩需要调用http请求也显得很繁琐
因此，博主对比后决定寻找本地的压缩工具
在对比过多个图片工具后，博主决定使用[pngquant](https://github.com/kornelski/pngquant)来压缩png图片，使用[jpegoptim](https://github.com/tjko/jpegoptim)来压缩jpg图片

## pngquant

这款png压缩工具做的比较友好，它支持png图片的有损压缩，我们可以通过下面的参数来更改压缩出来图片的最小与最大质量：
```
--quality=min_quality-max_quality
```
更多的命令请参考github或官网
同时，在不同的平台上运行我们并不需要自己去编译代码，在官网上进行下载即可：https://pngquant.org/

## jpegoptim

这款jpg压缩工具也支持有损压缩，同样也可以通过参数指定压缩图片的最大质量：
```
-mmax_quality
```
不过这款工具比较麻烦的地方在于它的并没有给出编译好的版本，我们只能在要使用到的平台上自行编译一次

## 在windows平台上编译jpegoptim

以下记录一下博主在windows平台上编译的辛酸史
不同于Linux以及Mac，在windos上配置环境、编译库等步骤比较麻烦，这里按照issues上倒数第二个回答的指示，一步步完成jpegoptim的编译工作：https://github.com/tjko/jpegoptim/issues/18

1. 首先，我们需要安装相应的编译工具：mingw（Minimalist GNU for Windows），针对Windows的迷你GNU工具集合：http://www.mingw.org/
使用MinGW Installer安装C++的编译环境，把Basic Setup里面的都装了：
![安装C++的编译环境](1.png)
2. 由于编译jpegoptim需要libjpeg库，我们需要去下载mozjpeg：https://github.com/mozilla/mozjpeg/releases
3. 使用mingw安装目录下的MinGW\msys\1.0\msys.bat启动命令行，执行mozjpeg下的configure文件生成makefile：
```
./configure --host=i686-w64-mingw32
```
	然后使用make命令进行编译
4. 下载jpegoptim源码，执行configure文件生成makefile：
```
CPPFLAGS=-I../mozjpeg LDFLAGS=-L../mozjpeg/.libs ./configure --host=i686-w64-mingw32
```
	然后使用make命令进行编译，不过这样编译出来的执行文件需要依赖动态链接库才能运行
5. 通过执行以下命令重新进行编译，使得编译出来的jpegoptim不再依赖动态链接库：
```
i686-w64-mingw32-gcc -g -O2 -I../mozjpeg -DHAVE_CONFIG_H -o jpegoptim jpegoptim.o jpegdest.o misc.o -L../mozjpeg/.libs -lm ../mozjpeg/.libs/libjpeg.a
```

至此，我们终于得到一个编译好的jpegoptim执行程序了

# git hook

git hook 的介绍网站：
https://git-scm.com/book/zh/v1/%E8%87%AA%E5%AE%9A%E4%B9%89-Git-Git%E6%8C%82%E9%92%A9

git hook 允许我们在一些特定的行为后执行自动的脚本
这里我们使用post-commit，在一次commit提交后找出该次commit提交包含的png以及jpg文件，本地进行压缩后新增一次commit进行提交
git hook 的代码如下：
```sh
#!/bin/sh

# version 1.3

hook_tools_dir="picCompressGitHook/"
hook_commit_message="[#0]this commit is to compress picture by post-commit hook"

# if true, that means recently commit is by post-commit hook, just exit
if [ "$(git log -1 HEAD --pretty="%s")" = "$hook_commit_message" ]
then
	exit 0
fi

# this is copy from sample
if git rev-parse --verify HEAD >/dev/null 2>&1
then
	against=HEAD~1
else
	# Initial commit: diff against an empty tree object
	against=4b825dc642cb6eb9a060e54bf8d69288fbee4904
fi

# check the switch
no_compress_pic=$(git config --bool hooks.nocompresspic)
if [ "$no_compress_pic" = "true" ]
then
	echo "no compress pic after commit"
	exit 0
fi

close_hint="if you want to disable pic compress, you can disable switch : \ngit config hooks.nocompresspic true"
max_quality=100
min_quality=80
png_suffix="-compress.png"

# get pic to compress
diff_item_array=($(git diff $against --name-only --diff-filter=AM HEAD))
pic_array=()
counter=0
for diff_item in ${diff_item_array[@]}
do
	if [ $(expr match "$diff_item" ".*\.[p|j][n|p]g$") -gt 0 ]
	then
		pic_array[counter]=$diff_item
		let counter++
	fi
done

# check have pic to compress and compress tools
if [ ! ${#pic_array[@]} -gt 0 ]
then
	echo "this commit don't have pic to compress, just exit"
	exit 0
elif [ ! -f $hook_tools_dir"pngquant-windows" ] || [ ! -f $hook_tools_dir"jpegoptim-windows" ] # pwd is root project dir
then
	echo "stop compress pic !"
	echo "you don't have pngquant-windows and jpegoptim-windows in the $(pwd)/$hook_tools_dir dir !"
	echo -e $close_hint
	exit 0
fi

# check whether need to stash old uncommit file
working_file_array=$(git diff --name-only)
cached_file_array=$(git diff --cached --name-only)
git_need_stash_pop=0
if [ ${#working_file_array[@]} -gt 0 ] || [ ${#cached_file_array[@]} -gt 0 ]
then
	echo "there are some files not added to commit, do git stash"
	git stash
	git_need_stash_pop=1
fi

# compress pic
echo "there is ${#pic_array[@]} pic going to compress"
echo -e "\n###################################################\n"

compress_success=0
for pic in ${pic_array[@]}
do
	if [ $(expr match "$pic" ".*\.png$") -gt 0 ]
	then
		# png
		png_compress=${pic%\.*}$png_suffix
		"./"$hook_tools_dir"pngquant-windows" --skip-if-larger --ext $png_suffix --quality $min_quality-$max_quality -f --strip $pic >/dev/null 2>&1
		if [ -f $png_compress ]
		then
			mv -f $png_compress $pic
			echo "compress png pic : ${pic##*/} success"
			compress_success=1
		else
			echo "compress png pic : ${pic##*/} fail"
		fi
	else
		# jpg
		"./"$hook_tools_dir"jpegoptim-windows" -m$max_quality -o --all-normal $pic >/dev/null 2>&1
		echo "compress jpg pic : ${pic##*/} finish"
		compress_success=1
	fi
done

echo -e "\n###################################################\n"

# do commit
if [ $compress_success == 1 ]
then
	echo "some compress success , do commit !"
	git add .
	git commit -m "$hook_commit_message"
else
	echo "no pic compress success, abort commit !"
fi

# stash pop old file
if [ $git_need_stash_pop == 1 ]
then
	echo "do git stash pop to get old files"
	git stash pop
fi
```

目前由于博主只编译了windows的pngquant以及jpegoptim压缩工具，因此在git hook 中只处理了pngquant-windows以及jpegoptim-windows两款工具，并没有处理不同平台的逻辑

# git hook 安装工具

由于git是默认不追踪本地的git hook的，为了存放压缩工具、方便组内的童鞋使用以及日后的git hook脚本更新，博主新建了一个picCompressGitHook文件夹，往其中放入了：
- jpegoptim-windows：windows环境编译的jpegoptim
- pngquant-windows：windows环境编译的pngquant
- post-commit：git hook 脚本
- install.sh：安装脚本，调用即可复制git hook

安装脚本的代码如下：
```
#!/bin/sh

chmod a+x pngquant-windows
chmod a+x jpegoptim-windows
cp -f post-commit ../.git/hooks/post-commit
```

使用了自动压缩图片的git hook之后，提交了图片会自动压缩，再也不用自己打开网页去压缩图片了！
![自动压缩图片](2.png)