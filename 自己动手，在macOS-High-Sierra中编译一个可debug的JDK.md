---
title: 自己动手，在macOS High Sierra中编译一个可debug的JDK
date: 2018-01-28 15:08:20
tags:
---
## 背景
由于最近想分析几个`native`方法，所以需要手头有一个可以`debug`的`JDK`，因此，这两天折腾了折腾，踩了10+个坑，看了10+篇文章，尝试了10+次，最后总算把`JDK`给编出来了，当在自己编译出来的`JDK`上运行`javac -version`输出了那熟悉的文字后，感觉<del>已是老泪纵横</del>还是很有成就感的。

## 前期准备
- [了解一下OpenJDK的相关知识](https://zh.wikipedia.org/zh-hans/OpenJDK): 我们要编译的是`Open JDK 9u`
- 安装`Homebrew`(可选)
- 番羽土啬: 因为假如编译出错需要查询错误的解决方案时需要通过一个不存在的网站来进行
- 准备一个好用的源码编辑器: 因为运气好的话[滑稽]我们可能需要修改`JDK`源码中的八阿哥(我用的 mscode)
- 安装一个`Boot JDK`: 因为`JDK`中有不少代码是`Java`写的，所以想要编译`JDK`就需要用到另一个`JDK`来辅助(鸡生蛋、蛋生鸡/自己拽自己的头发把自己拽起来)，我安装的是Oracle官方的`JDK`
- 安装`freetype`:这是个依赖项`brew install freetype`
- 安装`Xcode`

## 环境描述
- 操作系统: `macOS High Sierra Version: 10.13.3 (17D47)`
- `Boot JDK`: `1.8.0_161`
- `freetype`: `2.9`
- `Xcode`: `Version 9.2 (9C40b)`
- `Open JDK`源码: `jdk9u-2e265b4b8622` `corba-2ef36e70f490` `hotspot-bb73b31e70e3` `jaxp-95a71f690b44` `jaxws-f4f878b5f01c` `jdk-a779673ab57d` `langtools-e2bf77b3f002` `nashorn-fb3f7ae74bf6`

## 前人蹚过的坑/我蹚过的坑
1. 不要编译`JDK 8`: 因为编译8需要`Xcode 4`现在`Xcode`版本已远高于4了(这个坑是前人蹚出来的，当然我不信邪，然后自己蹚了一遍，然后发现确实是个坑)
2. 尽量不要使用`Mercurial`来下`Open JDK`的源码，而用浏览器直接下载打包好的源码: 因为你要用`Mercurial`下载了`Open JDK`的顶层工程后，还需要执行其中的`get_source.sh`来下载其子工程的源码，这个过程漫长而且失败率高(关键还没有执行百分比提示，干等)，用浏览器自己下载所有子工程的压缩包要快很多，而且成功率`100%`(我自己蹚出来的，前人的几篇文章中全告诉我用`Mercurial`，结果坑了)
3. 编译`JDK 9u`而不是`JDK 9`因为我当时编译`9`的时候出了一堆error，所以我想带个`u`的是不是会好些，当然这个有很大程度是我臆测的，我后期就都拿`9u`搞了，没试`9`，如果各位有兴趣可以自己蹚蹚这个
4. `configure`的时候一定要带上`--disable-warnings-as-errors`这个参数，否则编译过程中的`warning`也会中断编译的进程，实际上这些`warning`并不影响编译后的目标`JDK`的运行

## 走流程
如果前面几点 <b><i>该准备的你都准备了</i></b>，<b><i>该注意的你也注意了</i></b>，那么开始走流程编译吧，如果报错（有很大几率会）不要摔键盘，用前面我们提到的不<del>可描述</del>存在的网站来找答案，我自己摸着石头过河，反复试了2天才搞定，鲁迅先生说的好：<i>“只要功夫深，铁杵磨成针”</i>，祝你好运！！

### step0
所有前文提到的准备工作

### step1
[下载源码压缩包](http://hg.openjdk.java.net/jdk-updates)

{% asset_img 1613b6b332a4c3d6.png %}
{% asset_img 1613b6787016fbb8.png %}
并将下载好的源码压缩包解压后按官方源码库的层级码放好
{% asset_img 1613b662d64838c2.png %}

### step2
在源码顶层目录上执行`sh configure --with-debug-level=slowdebug --disable-warnings-as-errors --with-freetype-include=/usr/local/Cellar/freetype/2.9/include/freetype2 --with-freetype-lib=/usr/local/Cellar/freetype/2.9/lib`
(带版本号的地方自己注意根据实际情况换)如果这一步没问题的话应当看到这样的输出：

{% asset_img 1613b75ceb064458.png %}

### step3
如果你的编译环境以及源码版本跟我的完全一样，那么先别`make all`，先来修一个[bug](https://bugs.openjdk.java.net/browse/JDK-8174050)：([patch在此](http://hg.openjdk.java.net/jdk10/jdk10/hotspot/rev/316854ef2fa2))
这个bug会导致`make`报这个:`error: ordered comparison between pointer and zero ('char *' and 'int') 。。。此处省略`


打开`hotspot`目录中的
- `src/share/vm/memory/virtualspace.cpp` 搜索其中`if (base() > 0) {`改为`if (base() != NULL) {` 
- `src/share/vm/opto/lcm.cpp` 搜索其中`if (Universe::narrow_oop_base() > 0) {` 改为 `if (Universe::narrow_oop_base() != NULL) {`
- `src/share/vm/opto/loopPredicate.cpp` 搜索其中`assert(rng->Opcode() == Op_LoadRange || _igvn.type(rng)->is_int() >= 0, "must be");` 改为 `assert(rng->Opcode() == Op_LoadRange || iff->is_RangeCheck() || _igvn.type(rng)->is_int()->_lo >= 0, "must be");`

### step4
`make all`
编译时间略长，我的机子是2017款的 MacBook Pro大概用了20分钟+(没仔细计)，而且编译过程中风扇狂转，笔记本的话注意剩余电量以及散热，编译若成功，指定`javac`全路径运行一下`javac -version`看看

{% asset_img 1613b8ae4a42b445.png %}

如果报了error，具体error具体分析，因为编译环境不同，可能error也不同，去不存在的网站找找解决方案，耐心+运气你就一定能成功的。

## 总结
原以为编一个`JDK`能有多难，事实证明：是不难，但是坑多，麻烦，前前后后试了10多次，一共花了两天时间，从尝试编译`JDK 8`再到`9`中间出了各样问题，之中一度想放弃，但是由于想用它来实现更远的目标，所以还是坚持弄下来了，最终在搜索引擎和前人的帮助下成功了，所以还是那句话：“只要功夫深铁杵磨成针”，按正确的方向，反复尝试，终能成功。

## 参考资料
[Building OpenJDK 8 on Mac OS X Mavericks](http://gvsmirnov.ru/blog/tech/2014/02/07/building-openjdk-8-on-osx-maverick.html)

[Building and Packaging OpenJDK9 for OSX](https://github.com/hgomez/obuildfactory/wiki/Building-and-Packaging-OpenJDK9-for-OSX)

[Compilation errors with clang-4.0](https://bugs.openjdk.java.net/browse/JDK-8174050)

[OpenJDK / jdk10 / jdk10 / hotspot
changeset 13502:316854ef2fa2](http://hg.openjdk.java.net/jdk10/jdk10/hotspot/rev/316854ef2fa2)

[Ordered comparison between pointer and zero ('char *' and 'int') error #5](https://github.com/tat/mimetic/issues/5)

[MAC编译OpenJDK8](http://blog.csdn.net/manageer/article/details/72812149)

[Compile&Debug openjdk](https://liuzhengyang.github.io/2017/04/28/buildopenjdk/)

[openjdk code compilation/ IDE setup
](https://stackoverflow.com/questions/44512922/openjdk-code-compilation-ide-setup)

[使用Clion(GDB)调试小型JVM源码](https://www.jianshu.com/p/ca46826073a0)

[解决GDB在Mac下不能调试的问题](https://segmentfault.com/q/1010000004136334)

[xcode-select active developer directory error
](https://stackoverflow.com/questions/17980759/xcode-select-active-developer-directory-error)

[Mac OsX Sierra - Stuck on binaryTreeDictionary.hpp compilation](http://mail.openjdk.java.net/pipermail/build-dev/2017-May/019179.html)