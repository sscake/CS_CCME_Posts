# ADF 2012.01破解原理

最早的破解方法参照的是小木虫上的一篇文章。目前这篇文章已遭删除，备份于[此处](/adf/adfm.md#小木虫文章备份)。

实现破解的补丁程序源码位于[此仓库](https://github.com/Z-H-Sun/MRN-ADF_Patch)。

现在有了更好的反汇编工具：`IDA Pro`，无论是什么平台（32/64位）、什么操作系统（Windows / Mac OS X / Linux），都能解决。而且其生成伪代码的功能使得冷冰冰的汇编指令逻辑变得易读得多。

上面的破解方法抓住了一个很好的入口，但是还是有很大的问题。一是作者自己承认的，人工操作太耗费时间。所以我转而去寻找HEX指令中重复出现的“***模式***”，并用Ruby脚本自动化破解过程。毕竟判定过程的逻辑差不多，所以编译得到的HEX指令应该很相近才对。

第二点非常严重，我估计作者自己也没有意识到，这样的破解其实是不完全的……的确，他删除了相等时跳转的指令，但是其实也很容易想到，在不跳转的情况下，完全按顺序执行也是有可能执行到触发“过期”的那一步的。所以这样更改的净结果相当于是把之前“若`Year > a`***或***`Month > b`***或***`Month > c`，则触发过期”的逻辑转变为更弱的“若`Year > a`***且***`Month > b`***且***`Month > c`，则触发过期”。即一年中最后几个月中的最后几天还是没法用……很隐蔽的失误。

使用伪代码可以帮助我们更直观地了解这片指令附近的逻辑
<p align="center"><img src="/adf/2.png" width="80%" height="80%"></p>

原始指令为`mov ecx, 0`，对应右侧`v125=0`。后面有判断逻辑`if !(v125 && 1)`则触发过期。所以只要令`v125=-1`即可避免这一过程。程序一开始设置`v125=0`，如果后续判断中没有将其重设为`-1`，则触发过期。小木虫作者更改了上述“后续判断中”的两个逻辑条件，但并没有从源头上解决问题。解决办法很简单，最开始就令`v125=-1`即可（改为`mov ecx, FFFFFFFFh`）。

至于怎么找到需要打补丁的地方，小木虫作者已经讲得比较明白了。但是不得不说，他的方法比较慢。其实可以善用`IDA`的`Names Window`（`Ctrl+F4`）和`String Window`（`Shift+F12`）功能。跳转到相应字符串后，先`Alt+M`添加书签以备后查；再按`X`，跳出XREFS（交叉引用）列表，就能跳转到相应的函数/过程。

这里要吐槽写这个程序的作者：明明是同样的验证机制，写一个调用方法就好了呗；他偏不，搞了七八个功能完全一样的过程……所以还得一个个找出来挨个破解。其实我怀疑有几个过程压根就不可能被调用，但小心起见，还是一股脑儿全打补丁得了。比如说，直接运行dirac和加运行参数的dirac info所调用的就是其中两个不同的验证过程（*这里再吐槽一句，所有GUI程序进行自己的许可验证完了后还会再用dirac验证一遍……所以就算破解了GUI程序只要没破解dirac一样白搭，之前吃过这个亏，还是动态分析工具x64dbg帮忙找出来的*）。Windows下是能找到好几个“License for Module”字符串，每个字符串有且仅有一个交叉引用；Linux下则是仅有一个“License for Module”字符串，但有七八个交叉引用……

关键是每个过程编译完的汇编指令还不一样！这给自动化过程平添了很多麻烦。你必须找出所有可能的“***模式***”。

在另一种模式中，上述的`mov ecx, 0`变为了`mov eax, esi`，其中ESI寄存器一直没动过所以默认为0。麻烦之处在于，这个指令只占2个字节，而`mov eax, FFFFFFFFh`占5个字节，*真可惜，空白太小写不下了*。不过我们再回过头来看最终的判定条件，`if !(v125 && 1)`则触发过期，即EAX值和1进行`AND`运算后非零才能避免触发过期。所以只要最低位是1，不是0，即可。那么汇编指令可以这么写：`mov al, FFh`。这样做是把EAX寄存器低八位设为-1，即`LOBYTE(v125) = -1`（可以参考下图）。

还有更古怪的`xor ecx, ecx`也表示把ECX寄存器赋值为0（下图）。同样是占2个字节，所以照样改成`mov cl, FFh`。类似的还有`mov edx, edi`，`xor eax, eax`等。
<p align="center"><img src="/adf/3.png" width="80%" height="80%"></p>

总之找到所有可能的模式、知道怎么改就行。下图是[Ruby自动化破解程序](https://github.com/Z-H-Sun/MRN-ADF_Patch)运行示意图。至此，所有ADF模块都已经破解完成了。
<p align="center"><img src="/adf/1.png" width="80%" height="80%"></p>

但还有美中不足的地方。做了这么多，只是破解了“过期”这一点。但是我们的许可文件还有两处限制：

* **没有GENNBO（自然键轨道）的许可**
* **限制并行核数上限为8**

判断逻辑仍然位于这一个方法之内，但和判断过期的方式截然不同，分析难度陡增。似乎是先调用了另一个方法，总体上确定是否LICENSE INVALID，最后再细分是没有当前模块许可还是超过核数上限。真是奇葩的逻辑……（据说ADF 2014之后把过期判定也一并放到这个逻辑模式下了，所以这样的方法也就失效了）。不过破解到这个地步已经能满足绝大部分计算需求，而且为了这个破软件我已经花费了足够多的心思（包括使用时也有很多巨坑……暴力填实了）。所以我也不打算继续花心思研究下去了，如果读者还有兴趣不妨再挣扎一下。



## 小木虫文章备份

> <p align="center">ADF（32位windows版）过期后的破解方法</p>
> 
> <p align="center">作者 beefly	来源: 小木虫</p>
> 
> ADF的license管理系统十分复杂，把所有可能遇到的情况都测试一遍十分耗费精力（也许是我还没找到更简单的破解方法），因此这里不涉及完全破解（例如，制作license生成器，或取消license检查），只针对最简单的一种情况，即有效license（通常是试用license，或买的license）过期后的解决办法。当然修改系统时间也能解决问题，但是如果你没有windows系统管理员权限，或者觉得改系统时间太麻烦，就需要改应用程序了。
鉴于64位windows系统下还没有比较完善的分析工具，本文只涉及32位ADF，但是如果在其他系统（64位windows，linux，macos等）下能找到合适的工具，也可作为参考。破解工具用C32Asm。这是一个国产的静态反编译工具，比W32Dasm更好用：具有16进制编辑器，支持超大的文件，可以更准确地搜索字符串（即使是没有图形界面的应用程序），等等。对于1 MB以上大小的程序，C32Asm搜索字符串比较慢。所以除了C32Asm以外，最好再装一个UltraEdit，用于快速搜索字符串。
为了熟悉操作，从一个比较小的模块gennbo.3.exe开始。License过期后，运行gennbo会出现以下输出：
> 
> License for module        ADF version  *** has expired on  ***
> 
>（略）
> 
> License for module      Utils version  *** has expired on  ***
> 
> \---------------
> 
> LICENSE INVALID
> 
> \---------------
> 
> 这说明，判断license过期的代码应该在出错信息之前，在打印“License for module …”的代码附近。
> 
> 先用UltraEdit打开gennbo.3.exe，搜索字符串“License for module”，会发现有4处。
> 
> 再用C32Asm打开gennbo.3.exe，搜索字符串“License for module”，出现以下代码：
> 
> ::0047CAE0::  F685 889EFFFF 01         TEST BYTE PTR [EBP+FFFF9E88],1          \:BYJMP JmpBy:0047C9E9,
> 
> ::0047CAE7::  0F85 78E6FFFF            JNZ 0047B165                            \:JMPUP
> 
> （略）
> 
> ::0047CB0F::  C785 ECBCFFFF A4B15600   MOV DWORD PTR [EBP+FFFFBCEC],56B1A4         \->: License for module 
> 
> 立刻可以看出，这段代码是从0047C9E9跳进来的。0047C9E9和之前几行的代码如下：
> 
> ::0047C9D5::  83BD 849EFFFF 00         CMP DWORD PTR [EBP+FFFF9E84],0          \:BYJMP JmpBy:0047C992,0047C9B3,
> 
> ::0047C9DC::  0F84 83E7FFFF            JE 0047B165                             \:JMPUP
> 
> ::0047C9E2::  F685 889EFFFF 01         TEST BYTE PTR [EBP+FFFF9E88],1          
> 
> ::0047C9E9::  0F84 F1000000            JE 0047CAE0
> 
> 这段代码有两个入口：0047C992和0047C9B3。来到0047C992，代码如下：
> 
> ::0047C992::  75 41                    JNZ SHORT 0047C9D5                      \:JMPDOWN
> 
> （略）
> 
> ::0047C9B3::  75 20                    JNZ SHORT 0047C9D5                      \:JMPDOWN
> 
> 0047C992和0047C9B3这两行表示，若不等就打印license过期的信息，并退出计算。用C32Asm的HEX编辑器把这两行的75 41和75 20都改为90 90（不作任何操作）。
> 
> 搜索剩下的三处字符串“License for module”，并做类似修改。保存。
> 
> 现在再运行gennbo，可以正常计算了。
> 
> ADF的其他模块也可以作类似修改。例如，在adf.3.exe，dirac.3.exe，band.3.exe中，字符串“License for module”分别出现8次。由于这些程序很大，搜索+修改可能要花一两个小时，但是方法都一样。用此方法，我花了整整12小时，把ADF涉及license的30个非图形程序都破了，测试计算全部正常。
> 
> 上面的方法对ADF的图形界面程序一样有效，但是搜索过程极其耗时。