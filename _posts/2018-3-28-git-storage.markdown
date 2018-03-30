---
layout: default
title:  "git存储原理"
date:   2018-03-28
categories: file system
---

由于我们的文件系统核心支撑场景是git应用，理解git存储原理是非常关键，所以先分析下这块。

参考书籍，[Pro Git第二版](https://git-scm.com/book/zh/v2/Git-%E5%86%85%E9%83%A8%E5%8E%9F%E7%90%86-%E5%BA%95%E5%B1%82%E5%91%BD%E4%BB%A4%E5%92%8C%E9%AB%98%E5%B1%82%E5%91%BD%E4%BB%A4).

## git目录结构介绍

git的核心业务逻辑都在隐藏的.git目录中。

比如我们新建一个git仓库，git init，就会生成一个.git目录。其中的结构如下：

![]({{ site.baseurl }}/assets/gitstore-1.png)

其中相对静态的文件如下：

- config 文件包含项目特有的配置选项 —— 不用太关注
- info 目录包含一个全局性排除（global exclude）文件，用以放置那些不希望被记录在 .gitignore 文件中的忽略模式（ignored patterns）—— 也不是重点
- hooks 目录包含客户端或服务端的钩子脚本（hook scripts） —— 用来触发一些hook动作，比如触发CI，也不是重点

git中关键的几个文件如下：

- HEAD：指向目前被check出的分支
- index文件（初始化的时候没有）：保存暂存区的信息（add了，还没commit的信息）
- objects目录：存储所有的数据内容
- refs目录：指向分支的提交对象的指针

本文重点关注这4个文件和目录的操作流程。

## Git中的对象模型

- Git是基于文件系统之上构建的基于内容寻址的KVDB存储系统（或者称多版本文件系统更准）。我们可以通过git的命令行来操作这个KVDB，比如往启动写入一个Value，就返回一个hash值，也就是key。其中-w参数指示写入存储，否则仅仅只是计算一个hash值。

- 存储在git中的对象类型包括blob对象（类似file内容），tree对象（目录树），commit对象（类似快照指针）

### blob对象
```
$ echo 'test content' | git hash-object -w --stdin
d670460b4b4aece5915caf5c68d12f560a9fe3e4
```
- 扫描盘上文件，发现多了一个以这个hash值为文件名的文件

```
$ find .git/objects -type f
.git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4
$ cat .git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4
xK��OR04f(I-.QH��+I�+�K�	% (一堆乱码，因为git对内容作了压缩和编码，只能用git的命令行来查看)
$ git cat-file -p d670460b4b4aece5915caf5c68d12f560a9fe3e4 # -p参数是自动判断数据类型，以友好的方式显示
test content
```

- 当然对文件也可以用类似的操作方式，此时相当于git commit了一个test.txt到本地git仓库中

```
$ echo 'version 1' > test.txt
$ git hash-object -w test.txt
83baae61804e65cc73a7201a7252750c76066a30
```

- 我们改变内容再写入一次，相当于针对text.txt作了一次修改再提交

```
$ echo 'version 2' > test.txt
$ git hash-object -w test.txt
1f7a7a472abf3dd9643fd615f6da379c4acb3e3a
```

- 此时查看.git中的对象，可以发现2次提交直接对应了2个文件

```
$find .git/objects -type f
.git/objects/1f/7a7a472abf3dd9643fd615f6da379c4acb3e3a
.git/objects/83/baae61804e65cc73a7201a7252750c76066a30
.git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4
```

- 想读取不容版本的内容，直接用不同的key下去查询即可

```
$ git cat-file -p 83baae61804e65cc73a7201a7252750c76066a30 > test.txt
$ cat test.txt
version 1
$ git cat-file -p 1f7a7a472abf3dd9643fd615f6da379c4acb3e3a > test.txt
$ cat test.txt
version 2
```

### tree对象

- 类似文件系统目录的概念，tree与blob的关系如下图所示：

![](https://git-scm.com/book/en/v2/images/data-model-1.png)

- 一个tree对象包含一个或者多个指针（key），也是就是hash值，指向blob或者tree，随意打开一个现有的git仓库看看就清楚了

```
>git cat-file -p master^{tree}
100644 blob 98aa193be6888d0c7edbc72626bbcd0570a98202	.gitignore
100644 blob fde8f86ddbd1231c862cc4c2266002ac69e9347c	.mailmap
100644 blob c355e96fb72cb3b4f6a1ef92c8b6e8d4e6d69ab7	.travis.yml
100644 blob 23c050d8652c05bb0ec186da8929a7efa5b5b504	AUTHORS
100644 blob d511905c1647a1e311e8b20d5930a37a9c2531cd	COPYING
100644 blob fb3ba4074581b564c2922e5f4a9dabac8547345c	ChangeLog
100644 blob 7d1c323beae76333f523f6df31c47a87f5597edb	INSTALL
100644 blob 5be13796191ff081fc7bce7c6b90ccb984123004	Makefile.am
100644 blob 859ecdc34f76ba325f0d814ac60a1b6781383ec1	README-CN.md
100644 blob 82faea60cd2113b400832c770e3b0757dbcd3140	README.md
100755 blob ae16921b9430adb99f59ca3313d1ea3a0ce54639	autogen.sh
100644 blob b5c8676ccfd2d0f57b463692ab4b814767303986	configure.ac
040000 tree 4adcfd9081816aef6e72dcdef305e59d26e1a818	doc
040000 tree 42a9084cda14820530a7fb0d3d10bce9219de913	scripts
040000 tree 7fbee2acc2ebe1f28b9337979d7250af3b68b7b0	src
040000 tree ac15e14431b1ef04470816ef7477492b1815fb59	test

git cat-file -p 7fbee2acc2ebe1f28b9337979d7250af3b68b7b0
100644 blob ce8d33fed3372442235f6f97826e520b8069cae4	Makefile.am
100644 blob 6773fbf082c7d95808dc3b9ce0341a18714e1401	cache.cpp
100644 blob ee88310227920772893069b0095fa37acf980652	cache.h
100644 blob 4ff6061bbcb5951a9cd4c11449c208a5f940ea52	common.h
100644 blob 4d3b0b268368497abf3f781cf1486becb945f3f8	common_auth.cpp
100644 blob 85e1becb2e7a94d2b9909cb9b196dc29eaf3cd2d	curl.cpp
100644 blob 6450a1a457fb4daa784334ab7cf702e8e4ccce03	curl.h
100644 blob 3e8e9eafd20eaea7cf50c0c1474e478bc2c8071c	fdcache.cpp
100644 blob d2a6b74885d73f1e521d36411cfe41ba6ba5bb09	fdcache.h
100644 blob d0067924f4f678eaa5b3699c1c6250fdfcb2dc72	gnutls_auth.cpp
100644 blob 6187b306eeb50a8a44f390115d22dcf98f9ee07f	nss_auth.cpp
100644 blob cb1c4c13efc7255617a4029b12e4677c532f5789	openssl_auth.cpp
100644 blob 9ada605e77c6024d2fc4dc7cc0599796f4d446be	s3fs.cpp
100644 blob c9e82ed0ef0b1181a15735ab6f95248b4c00f97a	s3fs.h
100644 blob eaf09b18371032ff2bfa2be53bcd920015c2b913	s3fs_auth.h
100644 blob aeba6c2c5be308a35ccc9c9999bd699f1bfe3662	s3fs_util.cpp
100644 blob 799a16dd63ba804330b20e6c4a143c4f0136ff55	s3fs_util.h
100644 blob e33ecac1362e35f3dce57a5c124458bfca4a0e12	string_util.cpp
100644 blob 91f7b7a8c5fa5f343953a8f628350472f5e1751d	string_util.h
100644 blob c1782173b24aeb8944641a89be4d79c1af7be017	test_string_util.cpp
100644 blob 0e496ee29c917f984ce12421349b2f7f520fe23c	test_util.h
```

- 可见整个源码库源文件都在一颗树下，git clone的过程就是复制这里的东东放入本地文件系统的目录中的；每个分支都会对应这样的一棵树

- 当然我们也可以通过命令行创建一个tree对象:本例中，指定了文件模式为 100644，表明这是一个普通文件。其他可用的模式有：100755 表示可执行文件，120000 表示符号链接。文件模式是从常规的 UNIX 文件模式中参考来的，但是没有那么灵活 ── 上述三种模式仅对 Git 中的文件 (blobs)

```
$ git update-index --add --cacheinfo 100644 \
  83baae61804e65cc73a7201a7252750c76066a30 test.txt
  # 相当于调用add命令将test.txt加入暂存区并创建Index文件。
$ git write-tree
d8329fc1cc938780ffdd9f94e0d364e0ea74f579
$ git cat-file -p d8329fc1cc938780ffdd9f94e0d364e0ea74f579 # 这个tree下面存了一个blob对象test.txt 
100644 blob 83baae61804e65cc73a7201a7252750c76066a30	test.txt
```

- 当然我们也可用命令行，将tree串联起来，组成一个树；有点像用文件系统mkdir + mv的动作

```
$ git read-tree --prefix=bak d8329fc1cc938780ffdd9f94e0d364e0ea74f579 #将第一版本test.txt对应的父目录tree对象读出到暂存区，并命名为bak
$ git write-tree # 将读出来的结果写入当前的tree中，组成一个大树
3c4e9cd789d88d8d89c1073707c3585e41b0e614
$ git cat-file -p 3c4e9cd789d88d8d89c1073707c3585e41b0e614
040000 tree d8329fc1cc938780ffdd9f94e0d364e0ea74f579      bak
100644 blob fa49b077972391ad58037050f2a75f74e3671e92      new.txt
100644 blob 1f7a7a472abf3dd9643fd615f6da379c4acb3e3a      test.txt
#最终形成下面的图
```
![](http://git-scm.com/figures/18333fig0902-tn.png)

- 我们已经知道了blob对象就是对应本地目录中的一个文件，以内容的hash为文件名；实际上tree对象也是一样，每个tree对象在创建时会对应objects目录下的一个文件，只是存放的内容是一个个的目录项（有点dentry的概念）。

```
find .git/objects -type f
.git/objects/5f/da43a84182aa7329131e05a62cd6bb21b2feef
.git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4
.git/objects/d8/329fc1cc938780ffdd9f94e0d364e0ea74f579 # 这是个tree对象
.git/objects/1f/7a7a472abf3dd9643fd615f6da379c4acb3e3a
.git/objects/01/55eb4229851634a0f03eb265b69f5a2d56f341
.git/objects/e6/9de29bb2d1d6434b8b29ae775ad8c2e48c5391
.git/objects/fa/49b077972391ad58037050f2a75f74e3671e92
.git/objects/83/baae61804e65cc73a7201a7252750c76066a30

$ git cat-file -p d8329fc1cc938780ffdd9f94e0d364e0ea74f579
100644 blob 83baae61804e65cc73a7201a7252750c76066a30	test.txt # tree对象的内容
```

- 从blob对象和tree对象来看，在一个镜像仓库的早期，在本地文件系统中都是以小文件的方式存储（几十字节到KB级别），文件名为hash值

### commit对象

- 上面可以看到，不容的git文件系统树可以用不容的根来描述，但是由于tree对象都是用hash值来标记，人工太难记住和操作。commit对象就是用来保存不容快照目录树的根的。

![](http://git-scm.com/figures/18333fig0903-tn.png)

```
$ echo 'first commit' | git commit-tree d8329fc1cc938780ffdd9f94e0d364e0ea74f579 #创建一个commit对象
fdf4fc3344e67ab068f836878b6c4951e3b15f3d

>git cat-file -p 3c7898e2f1bef5b372b58db6440d2dcd7d9ba20b #查看这个提交对象
tree d8329fc1cc938780ffdd9f94e0d364e0ea74f579
author mingliang <sandpiper126@126.com> 1522422312 +0800
committer mingliang <sandpiper126@126.com> 1522422312 +0800

first commit
```

- commit对象也是对应盘上一个小文件，近路commit对应的message，commit对应的tree的key等信息

至此，.git目录下objects目录中的所有内容分析完成，在项目初期，这个里面的内容都是一个个很小的文件，key为文件内容的hash，value根据对象的不容有不同，blob存放实际的代码内容，tree对象存放下一级tree或者blob的指针（目录项），commit以可读的文本记录提交信息，并指向对应的tree。

### 对象的编码方式

- 我们可以看到，直接cat文件只能看到乱码，是因为所有的内容都经过了git编码，需要使用git的命令才能读取出来。

- 编码方式如下：

```
>> content = "what is up, doc?" #假设的内容
>> header = "blob #{content.length}\0"
=> "blob 16\000" # 得到content的头部

>> store = header + content # 存储的内容就是头部 + 内容
=> "blob 16\000what is up, doc?"

>> sha1 = Digest::SHA1.hexdigest(store) # 计算存储内容的key
=> "bd9dbf5aae1a3862dd1526723246b20206e5fc37"

>> zlib_content = Zlib::Deflate.deflate(store) #将存储的内容通过zlib压缩
=> "x\234K\312\311OR04c(\317H,Q\310,V(-\320QH\311O\266\a\000_\034\a\235"

>> path = '.git/objects/' + sha1[0,2] + '/' + sha1[2,38] # 用hash key的前2位来做目录（防止单个目录太大），后36位用来做文件名
=> ".git/objects/bd/9dbf5aae1a3862dd1526723246b20206e5fc37" # 也就是.git中objects路径下的东东了

将zlib_content存储到这个文件即可，读取的时候通过key找到文件，读取出来解压即可。
```