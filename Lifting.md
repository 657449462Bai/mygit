# Lifting介绍
## 示例代码test.c
这代码有两个明显的问题：
1. y变量是没用的，会被优化掉。
2. for循环是固定的3次(不算多)，可以使用“循环展开”策略进行优化。
```
#include <stdio.h>

int test(int x)
{
    if (x % 2) {
        x += 100;
    } else {
        x += 99;
    }

    return x;
}

int main()
{
    int x;
    int i;

    int y = 100;
    y += 1;

    srand(100);
    for (i = 0; i < 3; i++) {
        x = test(rand());
        printf("result:%d\n", x);
    }    

    return 0;
}
```
可以使用clang的O0和O3来编译，观察各自编译后的代码，利用IDA比较分析其相似性。
```
clang -S -O1 -mllvm --x86-asm-syntax=intel test.cpp
clang -S -mllvm --x86-asm-syntax=intel test.cpp
```
![1650280969104.png](./img/1650280969104.png)
![1650280969104](https://github.com/Baihaibo09/image/blob/main/1650280969104.png)
实际上两种编译策略得到的代码差异很大
![1650280991257.png](./img/1650280991257.png)
![1650280991257](https://github.com/Baihaibo09/image/blob/main/1650280991257.png)
这种 利用不同的编译方式 编译相同的代码，但是得到的编译结果不同 的情况，叫做编译器噪声。
# Lift：Binary -> IR 
## lifting
* 常规编译过程
![1650281247960.png](./img/1650281247960.png)
![1650281247960](https://github.com/Baihaibo09/image/blob/main/1650281247960.png)
这是一个由高级形式到低级形式的过程，叫做lowering。
* 反编译过程
![1650281322146.png](./img/1650281322146.png)
![1650281322146.png](https://github.com/Baihaibo09/image/blob/main/1650281322146.png)
与编译过程完全相反，叫做lifting。
注：Binary到ASM这一步是中规中矩的反汇编，一般编译器都自带这个功能。这一步是一个纯粹的byte code到汇编代码的过程。
## IR
首选的IR形式当然是LLVM IR，因为LLVM下可用的东西很多。
总结我们的流程：
![1650281572009.png](./img/1650281572009.png)
![1650281572009.png](https://github.com/Baihaibo09/image/blob/main/1650281572009.png)
到了LLVM IR之后我们可做的事情就很多了，毕竟这个时候整个LLVM的后端技术都可以应用起来。
opt是llvm的优化器，我们可以把llvm的一大堆pass都给它用上。如果有必要，我们还可以自行开发pass进行有针对性的处理。
主要涉及的问题有两个：
1. 把二进制文件转成LLVM IR。
2. 使用固定版本的OPT和完整的优化策略对IR进行二次加工，消除差异。（编译噪声）
# 实验验证
## 常规编译过程
1. 编译测试源码，生成LLVM IR

        clang -emit-llvm -O0 -c test.c -o 1.bc 

![1650281980440.png](./img/1650281980440.png)
![1650281980440.png](https://github.com/Baihaibo09/image/blob/main/1650281980440.png)
在没有任何优化的情况下，编译生成的IR跟O0情况下生成的汇编在逻辑结构上是完全一样的。
2. 使用OPT对IR进行优化，消除死代码。

        ./opt --mem2reg --adce --bdce /data/test/1.bc -o /data/test/2.bc 

![1650282127577.png](./img/1650282127577.png)
![1650282127577.png](https://github.com/Baihaibo09/image/blob/main/1650282127577.png)
通过对比看出，对y变量的操作已经没有了。
3. 接下来使用OPT继续优化，增加“循环展开”策略优化。

        ./opt --mem2reg --adce --bdce --loop-unroll /data/test/2.bc -o /data/test/3.bc

![1650282290011.png](./img/1650282290011.png)
![1650282290011.png](https://github.com/Baihaibo09/image/blob/main/1650282290011.png)
可以看到，for循环已经没有了，变成了连续3次的调用。
## 反编译过程：Binary -> llvm IR 
### 开源项目：（仓库https://github.com/lifting-bits）
> [Mcsema:](https://github.com/lifting-bits/mcsema)定义了一个模板库，把X86指令映射成C函数，然后再使用clang来生成IR。个人认为这个效率很低。主要原因是引入了clang。
参考：[使用 McSema 将二进制文件提升到 LLVM](https://layle.me/using-mcsema/)
> [RetDec:](https://github.com/avast/retdec):RetDec相对来说要高效很多，通过一个叫Capstone的引擎，完成X86指令到LLVM IR的映射。
> [mctoll:](https://github.com/microsoft/llvm-mctoll)
> //revng：[使用 revng 和 LLVM 检测二进制文件:](https://layle.me/instrumentation-with-revng/)
参考：<https://github.com/revng/revng>
> Ghidra:by NSA in 2019免费开源
>* 我尝试了RetDec，编译这个项目后会生成一个retdec-bin2llvmir工具用来做这种转换。
>* 使用retdec-bin2llvmir 将二进制代码转换成LLVM IR 后，和clang O3生成的IR来做对比，发现还是有一些差异。
RetDec这个工具从X86 code转换出来的LLVM IR跟clang生成的IR是存在差异的，而且这个差异不能通过优化进行消除。

[bingary lifting 相关资料](https://alastairreid.github.io/RelatedWork/notes/binary-lifter/)
mctoll的输出似乎更接近前向编译生成的LLVM，尤其是基于每个LLVM模块中的函数数量。retdec和mcsema似乎更接近于特定的逆向工程工具，因为它们都试图从二进制文件中提取所有内容，例如编译器生成的函数。
[比较三个二进制到LLVM转换器生成的LLVM IR]<https://adalogics.com/blog/binary-to-llvm-comparison>

参考链接：
1. <https://zhuanlan.zhihu.com/p/341448835>
2. <https://github.com/avast/retdec>
3. <https://github.com/lifting-bits/mcsema>
4. <https://adalogics.com/blog/binary-to-llvm-comparison>