---
layout:     post
title:      "深入理解Unity WebGL内存"
date:       2017-03-29 09:50:00
author:     "Lizarder"
header-img: "img/post-bg-2015.jpg"
catalog: false
tags:
    - unity3d
    - webgl
    - 内存分析
---

> #参考说明：此文参考翻译Unity官方社区文章，原文地址:[https://blogs.unity3d.com/cn/2016/09/20/understanding-memory-in-unity-webgl](https://blogs.unity3d.com/cn/2016/09/20/understanding-memory-in-unity-webgl/)


### unity webgl平台特别之处

<br>如果开发者从一些本来就要控制内存的平台像是PC或是WebPlayer转过来的话或许已经有个基础，应该都不会造成问题。
<br>
<br>如果目标是游戏机(Console)的话，内存管理相对比其他平台容易，因为可以准确知道内存是如何运用的。可以很好的管理内存确保游戏完美运作。手机平台内存管理就有些复杂，因为设备种类繁多，但至少可以选择最低标准的设备，并根据市场情况略过那些标准更低的设备。
<br>
<br>网页就没那么轻松了，理想状况下所有玩家(Client)都有64位浏览器和海量存储器，但事实总是残酷的。首先无法知道它们计算机的硬件规格。然后除了他们的操作系统和浏览器之外，并无法取得其它信息。最后玩家可能像执行其它网页一样的方法开启WebGL内容。因此这是一个非常复杂的问题。 

#### 概览
下面这张图显示了浏览器中unity webgl内存配置情况:
![post_unity_webgl_memory](https://lizarder.github.io/img/2017-03-29-unity_webGL_heap-2017/post_unity_webgl_memory.png)

<p id = "build"></p>
---

---

这张图显示在Unity Heap区域上，Unity WebGL的内容会需要分配额外的内存，这里需要理解清楚才能进一步优化来降低玩家流失率。
从图里能看到有几个内存分配，DOM, Unity Heap, Asset Data以及程序代码，程序代码一旦加载网页就会永远存在内存里，其他像是Asset Bundles, WebAudio, Memory File System将会依据内容不同来分配(Asset Bundle下载大小不同，音乐的播放不同等等)
在载入的时候，asm.js解析和编译期间也有几个浏览器内存临时分配，这里有时候会导致32位浏览器出现内存不足。

#### Unity Heap
一般来说，Unity Heap是包含所有Unity特定的对象(Game Objects)，组件(Components)，材质(Textures)和着色器(Shaders)等等。

在WebGL，Unity Heap的大小需要先算好让浏览器分配内存空间，一旦分配好之后Buffer就不能重设大小了。

负责分配Unity Heap的程序如下:
```C#
buffer = new ArrayBuffer(TOTAL_MEMORY);
```
 
这段程序可以在产生的build.js里找到，并会交由浏览器的JS VM执行。

TOTAL_MEMORY可以从Player Settings里的WebGL Memory Size来指定，预设情况是256MB，实际上一个空项目只有16MB。

然而，真的游戏项目可能需要更多，在大多数情况下会需要用到256MB或386MB，记住，设定越多内存表示越多玩家能正常执行。 

#### 源代码/编译码 的内存

程序可以执行之前需要:

1.被下载到Client

2.复制到一个Text blob区块

3.编译

 
顾虑到上述每一个步骤都需要一块内存

1）下载暂存区是暂时的，但源代码和编译码在内存里会留着直到关闭页面。

2）下载暂存区和源代码区分配的内存大小都是由Unity所产生的无压缩js空间。估计会需要多少空间:

	1、制作一个可发布版本
	2、将jsgz及datagz改名为*.gz，然后用解压缩的工具解开它们。
	3、解开后的大小会是需要的浏览器内存大小

3）编译码的大小取决于不同浏览器

一个简单的优化方法是启用引擎剥离功能(Strip Engine Code)，那样的话发布的包就不会包含不需要的原生的引擎码，(例如:不需要2d物理模块将会被剥除)。

例外支持(Exceptions support)和第三方的套件会影响程序大小，话虽如此，想要在发布时做空值检查(null checks)和数组检查(Array bounds checks)时内存不要销超出了例外支持的范围，为此，可以送-emit-null-checks和-enable-array-bounds-check给il2cpp，像这样:
```C#
PlayerSettings.SetPropertyString("additionalIl2CppArgs", "--emit-null-checks --enable-array-bounds-check");
```

最后，发布development build会产生更多程序代码的包因为它不是minified，因为最后会发布给玩家的是最终版本(release build)，对吧? ;-)。



#### 资源(Asset Data)

在其他平台上，应用程序可以永久的存取硬盘上的内容，但在网页是没有实际的文件系统所以是不可能的，因此，一旦Unity WebGL的数据(.data档案)被下载完成后，它就会存在内存里。缺点是和其他平台相比它需要额外的内存(从5.3开始.data的档案会压缩成lz4放在内存)。例如这个Profiler说明，256mb的Unity heap会产生约40mb的.data档案。
![unity_memory_p1](https://lizarder.github.io/img/2017-03-29-unity_webGL_heap-2017/post_unity_memory_p1.png)
 
甚么是.data档案?他是Unity产生的文件组合:data.unity3d(全部的场景，有依赖关系的资源和Resources目录底下的所有东西)，unity_default_resources和引擎所需要的一些小档案。

要了解资源的确切大小，可以在WebGL打包完后看一下 Temp\StagingArea\Data里的data.unity3d(记住，当关闭Unity时，temp文件夹会被删除)。也可查看传给UnityLoader.js里DataRequest的偏移植。
```
new DataRequest(0, 39065934, 0, 0).open('GET', '/data.unity3d');
```
(这段程序代码可能会依照不同的Unity版本而不同 - 这段从Unity5.4节录)

 
#### 内存文件系统(Memory File System)

上面提到Unity WebGL虽然没有真正的文件系统，但还是可以存取数据，和其他平台相比主要的差异是在所有的I/O行为都在内存里面完成。重要的是这个文件系统并不在Unity heap里面，因此会需要额外的内存开销，比如当我们写一个数组到档案时:
```
var buffer = new byte [10*1014*1024];

File.WriteAllBytes(Application.temporaryCachePath + "/buffer.bytes", buffer);
```

这个档案会写到内存里，也可以在浏览器的profiler找到:
![unity_memory_p2](https://lizarder.github.io/img/2017-03-29-unity_webGL_heap-2017/post_unity_memory_p2.png)

这段Unity heap的内存是256mb

同理，由于Unity的快取系统(caching system)依赖着文件系统，整个快取也是放在这样的文件系统，这代表像PlayerPrefs和快取的Asset Bundle会同样永远放在内存里直到关闭，而且是在Unity heap区的规画之外。


### Asset Bundles

降低WebGL内存消耗最好的方法之一就是采用Asset Bundle(如果对这个不熟可以参考文件)。然而，不同的Asset Bundle使用方式可能会对内存(Unity heap内或外)产生重大的影响，有可能会造成无法再32位浏览器无法正常执行。

现在需要用到Asset Bundle，然后该怎么做?将所有的资源打包成一包Asset Bundle?

千万不要!就算这样能降低网页的加载时间，还是需要下载(可能超大)这个Asset Bundle，导致内存飙高。来看看下载AB之前的内存用量:
![unity_memory_p3](https://lizarder.github.io/img/2017-03-29-unity_webGL_heap-2017/post_unity_memory_p3.png)

如看到的，256mb被Unity heap定义了。这是下载了AB之后还没有放入暂存。
![unity_memory_p4](https://lizarder.github.io/img/2017-03-29-unity_webGL_heap-2017/post_unity_memory_p4.png)

现在看到的是一个额外的缓冲区，大约是同等于硬盘上的65mb，由XHR分配。这只是一个临时的缓冲区但是他可能造成内存几祯的尖峰，直到GC(garbagecollect)回收它。

那么如何做可以减低这些内存尖峰?需要帮每个资源建立一个AssetBundle?很有趣的想法但不太实用。

打包Asset Bundle是没有一个规则，需要根据项目的需求来让包装下载有意义。(例如:单机游戏可选男女主角，且游戏周期只会用到一种，那选完角色之后可以只加载该性别的包)

最后，记住完成之后要用AssetBundle.Unload来卸除。
 
### Asset Bundle Caching
WebGL的Asset Bundle外取和在其他平台一样用WWW.LoadFromCacheOrDownload，虽然有一个明显的不同是这是放在内存的。在Unity WebGL里AB的快取依赖IndexedDB，被放在内存文件系统里的emscripten编译程序支持。

来看看用LoadFromCacheOrDownload下载AssetBundle之前的内存抓图:
![unity_memory_p5](https://lizarder.github.io/img/2017-03-29-unity_webGL_heap-2017/post_unity_memory_p5.png)

如所见，512mb被Unity heap用掉了，4mb左右其他分配。包被载入之后的图:
![unity_memory_p6](https://lizarder.github.io/img/2017-03-29-unity_webGL_heap-2017/post_unity_memory_p6.png)

额外的内存需求上升至167mb左右，这是这个AssetBundle额外需要的内存(原本是64mb压缩包)。然后下图是js vm做完GC之后的图:
![unity_memory_p7](https://lizarder.github.io/img/2017-03-29-unity_webGL_heap-2017/post_unity_memory_p7.png)

结果好多了，但仍需要85mb左右，大多数的内存用来存放AssetBundle，这些内存就算用unload去释放到结束都无法取回内存。还有，当玩家第二次用浏览器打开内容时，不管之前是否有分配过区块，这些内存又会再次被分配。

这是来自Chrome的内存快照参考:
![unity_memory_p8](https://lizarder.github.io/img/2017-03-29-unity_webGL_heap-2017/post_unity_memory_p8.png)

在Unity heap之外还有一个Asset bundle系统所需要快取相关的临时分配。坏消息是最近发现它比预期的大很多，预期会在Unity 5.5 beta 4, 5.3.6p6和5.4.1p2修复这个问题。

如果用更旧的Unity版本不想升级或是WebGL内容已经接近发布了，可以透过编辑器脚本来设定一些属性:
```
PlayerSettings.SetPropertyString("emscriptenArgs", " -s MEMFS_APPEND_TO_TYPED_ARRAYS=1", BuildTargetGroup.WebGL);
```
长远来看要最小化Asset Bundle高速缓存用量最好的解决方案是用[WWW构造函数](https://docs.unity3d.com/ScriptReference/WWW-ctor.html)而不是用[LoadFromCacheOrDownload()](https://docs.unity3d.com/ScriptReference/WWW.LoadFromCacheOrDownload.html)或是没有hash/version参数的[UnityWebRequest.GetAssetBundle()](https://docs.unity3d.com/ScriptReference/Networking.UnityWebRequest.GetAssetBundle.html)，如果使用新的[UnityWebRequest API](https://docs.unity3d.com/Manual/UnityWebRequest.html)的话。

然后在XMLHttpRequest等级使用备用的快取机制，将下载的档案直接存到indexedDB里避开内存文件系统。这是我们最近开发放在[Asset Store的方案](https://www.assetstore.unity3d.com/en/#!/content/71538)，需要的话可随意取用修改。


### AssetBundle 压缩
在Unity 5.3和5.4，都是支持LZMA和LZ4压缩的，虽然使用LZMA(默认值)结果会比未压缩的LZ4来的小，但用在WebGL上有几个缺点:有明显的效能问题、需要更多内存分配。所以我们比较推荐用[LZ4]()或[不要压缩]()(实际上，LZMA压缩法在Unity5.5会被移除)，为了弥补这个缺口，可能会希望能用gzip/brotli来压缩资源。
 
可以查看更多关于[打包压缩](https://docs.unity3d.com/Manual/AssetBundleCompression.html)的相关资料。
 
### WebAudio
UnityWebGL的音效是有别于其他平台的，这和内存会有什么关联?

Unity将在JavaScript上支持建立特定的AudioBuffer对象，用来更方便播放网页音效。
由于WebAudio缓冲区存在Unity heap外面，因此无法用Unity profiler追踪，需要用特定的浏览器工具来检查内存查看有多少用在音效上。这个例子用Firefox来查询音效内存用量:
![unity_memory_p9](https://lizarder.github.io/img/2017-03-29-unity_webGL_heap-2017/post_unity_memory_p9.png)

考虑到这些音效缓冲存放未压缩的数据，不太适合放超大型的声音文件(例如:很长的背景音乐)。所以现在可能需要写些自己的js套件来使用<audio>标签，这样声音文件压缩用较少的内存。
 
### FAQ
减少内存使用的最佳方法是什么？简单概括如下：
1、减少Unity Heap的大小
尽可能保持“WebGL Memory Size”够小
2、减少包里程序代码量
启用Strip Engine Code
关闭异常检测(Disable Exceptions)
避免使用第三方插件
3、减少数据大小
使用Asset Bundle
压缩材质

##### 是否有能够决定最小WebGL Memory Size的方法？
有，最佳方案是使用内存分析器(memory profiler)，分析内容实际所需的内存大小，然后依据结果改变WebGLMemory Size。

以空项目为例，内存分析器告诉我们总使用量仅为16MB(这个值可能在不同Unity版本上有所不同)，这代表只须要设定WebGL Memory Size大于16MB即可。当然，内存的总使用量将会依据内容而有所不同。

然而，如果因为某些原因无法使用分析器，可以简单地通过不断减少WebGL Memory Size 值，直到发现内容真正所需要的最小内存使用量为止。

另外值得注意的是任何不是16的倍数的值都将被自动四舍五入(执行期间)为下一个16的倍数，这是Emscripten编译程序所要求的。

WebGL Memory Size(mb)设定将决定产生html中TOTAL_MEMORY(bytes)的值。
![unity_memory_p10](https://lizarder.github.io/img/2017-03-29-unity_webGL_heap-2017/post_unity_memory_p10.png)

在不重新打包项目的前提下要测试内存堆栈的值，建议使用编辑html的方式。一旦找到适合的值，只需在Unity项目设定中更改WebGL Memory Size即可。

最后，记住Unity分析器将占用一些来自Unity heap的内存，所以在使用分析器时可能需要加一些WebGL内存大小。



##### 执行时发生内存溢位，如何修复？
这要看是Unity还是浏览器的内存溢位。错误信息会指出问题所在和解决办法，“如果是开发者，可以透过在WebGL设定中为项目分配更多(或更少)的内存来解决。”可以依据此来调整WebGL内存大小。然而还有很多可以解决内存溢位的方法。如果出现以下错误信息：
![unity_memory_p11](https://lizarder.github.io/img/2017-03-29-unity_webGL_heap-2017/post_unity_memory_p11.png)
 
除了讯息所提之外，还可以尝试减少程序和数据的大小。因为当浏览器加载网页时，它会尝试为一些内容寻找空余的内存，其中最重要的是：代码，数据，Unity heap和被编译的asm.js。它们可能相当大，尤其是数据和Unity heap内存，这对32位浏览器来说可能是问题。
 
在一些例子中，尽管有足够的内存，浏览器仍加载失败，因为内存是碎片化的。这就是为什么有时候内容可能在重新启动浏览器之后，可以成功加载的原因。

另一种情况是，当Unity 内存溢位时提示以下讯息：
![unity_memory_p12](https://lizarder.github.io/img/2017-03-29-unity_webGL_heap-2017/post_unity_memory_p12.png)
 
这种情况下就需要优化Unity项目。
 
 
##### 如何衡量内存消耗？
为了分析内容所使用的浏览器内存，可以使用Firefox浏览器的内存工具或Chrome的Heap snapshot。但它们无法显示WebAudio内存使用情况，因此还可以透过about:memory里的方法在Firefox里拍张快照然后搜寻“webaudio”找到。如果需要透过JavaScript分析内存，请尝试使用window.performance.memory(只支援Chrome)。

使用Unity Profiler测量Unity heap内存使用。需注意您可能需要增加WebGL的内存大小，以便能够使用Profiler。

此外，我们一直致力于开发新的工具，以便分析发布版本：使用时先包成WebGL版本，访问http://files.unity3d.com/build-report/就能使用该工具。虽然这个工具在Unity5.4中可用，但请注意这还是开发中的功能，可能随时会更改或删除。但至少现在可以使用它达到测试的目的。

##### WebGLMemory Size的最小值与最大值是多少？
16MB是最小的，最大是2032MB，然而我们通常建议保持在512MB以下。

##### 是否可能因为开发目的需要分配超过2032MB的内存？
这是一个技术上的限制：2048MB（或更多）将会超出TypeArray所用的32位整数型态的最大值，而TypeArray被用于在JavaScript中实现Unity heap。

##### 为何Unity Heap大小不可改变？
我们一直在考虑使用Emscripten编译程序标志ALLOW_MEMORY_GROWTH，来允许调整Heap的大小，但目前是没这么设定，因为它会禁用一些Chrome中的优化。我们还未对这个影响做一些真正的基准检验。预计使用这个flag会导致内存问题更严重。如果您遇到Unity heap太小，以至于无法满足所需内存的情况，这时就需要更多内存，那么浏览器就必须分配一个更大的Heap，从旧的里面中复制一切，然后再释放旧的Heap。这么做需要同时维持新Heap和旧Heap两份内存(直到完成复制)，这样需要更多的总内存。因此反而会比使用预定固定内存的方式占用更大。

##### 为什么32位浏览器在64位操作系统上会内存溢位？
32位浏览器执行时的内存限制是一样的，无论操作系统是64或32位。


### 结论

最后建议使用浏览器专用的分析工具，来分析Unity WebGL内容，因为Unity profiler无法追踪超出Unity heap之外的内存分配。

—— Lizarder at 2017.3.25 -> GuangZhou.  ——


