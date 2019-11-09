---
title: 多个android_support包？
date: 2019-9-22
tags:
  - 模块化
categories: Android
---

## 问题描述

`Android Studio`工程下多个`support`包的问题，查看的IDE左边`External Libraries`,现在我工程下`annotations`包有4个不同版本，特别是`25.0.1`版本，我压根没引用过，估计是第三方引用的。请问这样多个support包怎么管理？有什么影响？或者有什么统一方案么

这个问题在不同角度下不一样，比如至少有2个角度：

<!-- more -->

1. IDE debug 跟踪时会进入的 android support 包
    - 对于第一个角度，这个我认为和这个`IDE debug`的设计有关系，不过一般情况下都是这个库依赖于哪个版本那么调试时候它默认打开的文件会是哪个版本，没有的话可能会反编译什么的(这我不确定。。) 但是总之会找最接近的给你， 对于 `android studio` 好像是当前的module下依赖的那个包(因为这个点不是很重要，所以也不做探究，所以不一定正确) 
2. gradle编译打包时最后解依赖后打包进入apk的 lib(android support 或者其他的库) 的版本
    - 对于第二个角度，实际上是gradle自己的是怎么处理包之间的依赖和这些依赖相关的包的版本问题。详细的内容你肯定可以在gradle的文档里发现。原则上肯定是把最高版本的打包进去而忽略所有低版本的。这你不用担心，需要担心的反而是写库的人够不够规范，比如要是A库依赖于B库，你同时使用了A和B库，你用的B库版本高，但是A库依赖的B库版本低，那么但A库使用了一个B库已经在高版本废弃的api的话那就gg了，所以写库的人一定要保证自己的接口一般不会轻易修改

## 具体分析

### 假设
假设有一个项目，下面含有3个module A B C, A 是构建apk的主module 然后 A依赖于B和C， 但是 B和C没有任何依赖关系 然后有一个远程库叫做 lib.aar 它有2个版本 lib-1.0.aar 和 lib-1.5.aar 其中 这两个版本差距很大 然后还有一个远程库叫做 testLib.aar 这个库依赖于 lib-1.0.aar

### 接下来开始假设编译环境： 

1. 情况1： 只在 A 中(意思就是只在一个module中操作)引用了 lib-1.5.aar 和 testLib.aar 这两个库 结果：在 Project 页的 External Libraries 下只会出现lib库的最高版本 lib-1.5 和 testLib，最后打包进apk的是 lib-1.5 和 testLib 这2个库。
2. 情况2：在A中引用了 lib-1.0.aar 而在 B中引用了lib-1.5.aar (在不同的module中引用了 相同库的不同版本 ) 结果：结果：在 Project 页的 External Libraries 下只会出现最高版本的 lib-1.5 ，最后打包进apk的也是 lib-1.5, 但是在分别的module下按 ctrl 或 command 追源的时候只会到相应的版本的lib下
3. 情况3：在 A 中引用了 lib-1.5.aar 而在B中引用了 testLib.aar (在不同的module中引用了 不同的库， 但是库之间存在依赖关系且版本号不一致) 结果：在 Project 页的 External Libraries 下会同时出现lib-1.5 lib-1.0 及 testLib，最后打包进apk的 只会是 lib-1.5 和 testLib (注：这也就是为什么在主项目的 External Libraries 下面会出现同一个库多个版本的现象)
4. 情况4： 在B中引用了 lib-1.0.aar 而在 C中引用了lib-1.5.aar (没有依赖关系的module使用了相同库的不同版本) 结果： 在Project 页的 External Libraries 下只会出现最高版本的 lib-1.5 ，最后打包进apk的也只有是 lib-1.5, 但是在分别的module按 ctrl 或 command 追源的时候只会到相应的版本的lib下
5. 情况5： 在 B 中引用了 lib-1.0.aar 在 C 中引用了 testLib.aar (没有依赖关系的module使用了不同库，但是库之间存在依赖且版本号不一致) 结果： 在Project 页的 External Libraries 下只会出现最低版本的 lib-1.0 和 testLib ，最后打包进apk的却是 lib-1.5, 注： 实际上上面看 Project 页下的 External Libraries 是不准确的 最后打包进apk 或者是 通过 ctrl & command 追源 的时候 实际使用的应该是 当前的module 下的 build 目录下的 intermediates 目录下的exploded-aar 也就是 <module name>/build/intermediates/exploded-aar/<lib package name>/<版本>/<src> 所以我一开始提到的为什么我就算删除了本地的缓存 <usr/.gradle/cache/<相应的缓存>>，仍然会出现老的错误，原因是因为我没有删除 build 目录下的东西(但是为什么 Android studio 不用引用目录的方式要用拷贝的方式到build目录下啊？？？·····)

### 结论

最后结论当然就是只要记住：不管gradle怎么依赖，只要编译通过了，那么最后打包进apk的只会是最高版本的库 不过这句话的前提是编译通过了 

还要附加一点： 为什么会强调 module 的关系： 显然：因为打包进最后的只会是最高版本的，但是在单独的这个module下又会只使用自己module下指定的(build目录下存在的)， 所以在下断点调试的时候，如果跟到了库中的函数，若出现了行数对不上的情况，一点也不奇怪，因为很可能实际使用的是高版本的包而你当前调试时跟进入的只是个低版本的包而已