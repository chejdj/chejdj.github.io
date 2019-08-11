---
title: Andoid中的internal Strorage和 External Storage
date: 2018-11-25 19:00:00
categories: 
- Android
tags:
- blog
--- 

我们在操作File的时候，总会涉及到文件路径，而Android主要分为两种文件文件Internal Storage和External Storage，之前说实话一直使用的是External Storage而对于这两种之间的区别还是不太了解，直到有一天，我在测试服务器下发的图片的时候(之前因为服务器尺寸不对，需要更换尺寸)，使用链接下载，是正确的尺寸，可是在代码中打印Log却是不对，当然这种情况，是由于图片本地缓存的原因，于是我卸载再重新装，然后再测试，发现任然不对，难道本地缓存没有删除？Android对于App的文件又是一个怎么管理的过程呢？这个说实话，我之前确实没有了解，带着这些疑问我去Google。  

<!--more-->
#### 总概
我们所有使用的Android设备的文件存储区域分为 "Internal" 和 "Extrenal"，一个本机大小不可变存储空间，一个挂载的存储部件(理解为之前使用的"SD"卡，只不过现在好像这部分也不可以卸载了)。  
1. Internal storage  
* 总是存在，可用  
* 这个部分的文件默认只能被我们自己的APP访问  
* 当用户卸载app的时候，系统会把internale内和我们app相关的文件清除  
* Internal storage可以确保不被用户和其他APP访问  
2. External storage    
* 因为是可挂载的，并不总是可用，使用的判断一下可用性  
* 大家可以访问到，也可能被其他程序访问  
* 当用户卸载的时候，只是删除external根目录（getExternalFilesDir()）下的文件  
* 该区域，可以被其他程序和用户访问到  
3. app是默认安装到 internal storage的，但是我们也是可以更改他的安装目录的，  
在manifest使用andorid:installLocation属性,有三个属性值  
* auto 首选安装在internal，internal不够，安装在External  
* internalOnly: 默认值，只安装在internal部分，不足安装失败  
* preferExternal: 和auto相反  

#### 详细介绍  
1. internal storage
内部存储的路径为 `data/data/< package name >/files/ ` ,我们可以在AndroidStudio的Device File explore中查看，数据库文件，preference都存储在这里  
2. external storage  
存储路径为 `mnt/sdcard/Android/data/< package name >/files/…`  
3. 相关API介绍  

**对于Internal路径的API如下**：  
* getFilesDir() 返回一个File,代表我们APP的internal目录，卸载时自动删除  
(` /data/data/cn.xxx.xxx(当前包)/files `) 
* getCacheDir() 返回一个File,代表我们app的internal的缓存目录  
(` /data/data/cn.xxx.xxx(当前包)/cache `),这个文件目录并不会被在被卸载的时候删除，所以我们需要开发人员需要自己保证在不需要的时候删除，并且对大小进行合理限制，毕竟internal的存储大小有限。  

**对于External路径的API介绍**  
对于外部存储，我们有区分为两种
* Public file： 这个是用户和其他APP是可以访问的，而且在卸载的时候不删除的  
* Private file： 这个针对我们自己APP私有化的一个部分，这些文件理论上应该是不被其他APP访问的，但是存储在外部存储上，所以不是很好管理，是可以被其他App访问的，在用户卸载的时候，我们APP的private目录会被删除，所以我们的一些文件缓存应该放在这个地方  
* `getExternalStoragePublicDirectory(Environment.DIRECTORY_PICTURES)` 这个就是获取我们APP外部Public类型的文件，需要传入一个参数说明我们这些文件的用途  
* `getExternalFilesDir(Environment.DIRECTORY_PICTURES)`这个获取外部Private类型的数据，卸载的时候会被删除  
对于这个指定文件的类型，我个人认为是我们在使用MediaStore扫描本机媒体文件的时候，就去这些文件夹去查看  
(tips： 就比如我们在使用Glide图片缓存的时候，其实我个人更加倾向于使用getExternalFilesDir，在卸载的时候应该去掉缓存，我也越来越理解，打开手机个人文件，里面很多文件都是乱七八糟，很多都是没有自己在用户卸载的时候就直接留在那里，我觉得我们还是有必要自己清除的)  

#### 其他获取文件API介绍  
![code](https://github.com/chejdj/chejdj.github.io/raw/master/assets/blog_image/internal_and_external_storage/1.png)  
![log](https://github.com/chejdj/chejdj.github.io/raw/master/assets/blog_image/internal_and_external_storage/2.png)  
`Environment.getRootDirectory()` 获取系统目录  
`/storage/emulated/0/....`  外部存储  
`/data/...`  内部存储

 
