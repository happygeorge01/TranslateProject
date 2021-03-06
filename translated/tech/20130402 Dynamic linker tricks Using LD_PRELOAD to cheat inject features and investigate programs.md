# 动态连接的诀窍：使用 LD_PRELOAD 去欺骗、注入特性和研究程序

**本文假设你具备基本的 C 技能**

Linux 完全在你的控制之中。从每个人的角度来看似乎并不总是这样，但是一个高级用户喜欢去控制它。我将向你展示一个基本的诀窍，在很大程度上你可以去影响大多数程序的行为，它并不仅是好玩，在有时候也很有用。

#### 一个让我们产生兴趣的示例

让我们以一个简单的示例开始。先乐趣，后科学。


random_num.c:
```
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
 
int main(){
  srand(time(NULL));
  int i = 10;
  while(i--) printf("%d\n",rand()%100);
  return 0;
}
```

我相信，它足够简单吧。我不使用任何参数来编译它，如下所示：

> ```
> gcc random_num.c -o random_num
> ```

我希望它输出的结果是明确的 – 从  0-99 中选择的十个随机数字，希望每次你运行这个程序时它的输出都不相同。

现在，让我们假装真的不知道这个可执行程序的来源。也将它的源文件删除，或者把它移动到别的地方 – 我们已不再需要它了。我们将对这个程序的行为进行重大的修改，而你不需要接触到它的源代码也不需要重新编译它。

因此，让我们来创建另外一个简单的 C 文件：


unrandom.c:
```
int rand(){
    return 42; //the most random number in the universe
}
```

我们将编译它进入一个共享库中。

> ```
> gcc -shared -fPIC unrandom.c -o unrandom.so
> ```

因此，现在我们已经有了一个可以输出一些随机数的应用程序，和一个定制的库，它使用一个常数值 42 实现一个 rand() 函数。现在 … 就像运行 `random_num` 一样，然后再观察结果：

> ```
> LD_PRELOAD=$PWD/unrandom.so ./random_nums
> ```

如果你想偷懒或者不想自动亲自动手（或者不知什么原因猜不出发生了什么），我来告诉你 – 它输出了十次常数 42。

它让你感到非常惊讶吧。

> ```
> export LD_PRELOAD=$PWD/unrandom.so
> ```

然后再以正常方式运行这个程序。一个未被改变的应用程序在一个正常的运行方式中，看上去受到了我们做的一个极小的库的影响 …

##### **等等，什么？刚刚发生了什么？**

是的，你说对了，我们的程序生成随机数失败了，因为它并没有使用 “真正的” rand()，而是使用了我们提供的 – 它每次都返回 42。

##### **但是，我们 *告诉* 它去使用真实的那个。我们设置它去使用真实的那个。另外，在创建那个程序的时候，假冒的 rand() 甚至并不存在！**

这并不完全正确。我们只能告诉它去使用 rand()，但是我们不能去选择哪个 rand() 是我们希望我们的程序去使用的。

当我们的程序启动后，（为程序提供需要的函数的）某些库被加载。我们可以使用 _ldd_ 去学习它是怎么工作的：

> ```
> $ ldd random_nums
> linux-vdso.so.1 => (0x00007fff4bdfe000)
> libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f48c03ec000)
> /lib64/ld-linux-x86-64.so.2 (0x00007f48c07e3000)
> ```

正如你看到的输出那样，它列出了被程序 `random_nums` 所需要的库的列表。这个列表是构建进可执行程序中的，并且它是在编译时决定的。在你的机器上的精确的输出可能与示例有所不同，但是，一个 **libc.so** 肯定是有的 – 这个文件提供了核心的 C 函数。它包含了 “真正的” rand()。

我使用下列的命令可以得到一个全部的函数列表，我们看一看 libc 提供了哪些函数：

> ```
> nm -D /lib/libc.so.6
> ```

这个 _nm_  命令列出了在一个二进制文件中找到的符号。-D 标志告诉它去查找动态符号，因此 libc.so.6 是一个动态库。这个输出是很长的，但它确实在很多的其它标准函数中列出了 rand()。

现在，在我们设置了环境变量 LD_PRELOAD 后发生了什么？这个变量 **为一个程序强制加载一些库**。在我们的案例中，它为 `random_num` 加载了 _unrandom.so_，尽管程序本身并没有这样去要求它。下列的命令可以看得出来：

> ```
> $ LD_PRELOAD=$PWD/unrandom.so ldd random_nums
> linux-vdso.so.1 =>  (0x00007fff369dc000)
> /some/path/to/unrandom.so (0x00007f262b439000)
> libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f262b044000)
> /lib64/ld-linux-x86-64.so.2 (0x00007f262b63d000)
> ```

注意，它列出了我们当前的库。实际上这就是代码为什么被运行的原因：`random_num` 调用了 rand()，但是，如果 `unrandom.so` 被加载，它调用的是我们提供的实现了 rand() 的库。很清楚吧，不是吗？

#### 更清楚地了解

这还不够。我可以用相似的方式注入一些代码到一个应用程序中，并且用这种方式它能够使用函数正常工作。如果我们使用一个简单的 “return 0” 去实现 open() 你就明白了。我们看到这个应用程序就像发生了故障一样。这是 **显而易见的**, 真实地去调用原始的 open：

inspect_open.c:
```
int open(const char *pathname, int flags){
  /* Some evil injected code goes here. */
  return open(pathname,flags); // Here we call the "real" open function, that is provided to us by libc.so
}
```

嗯，不完全是。这将去调用 “原始的” open(…)。显然，这是一个无休止的回归调用。

怎么去访问这个 “真正的” open 函数呢？它需要去使用程序接口进行动态链接。它听起来很简单。我们来看一个完整的示例，然后，我将详细解释到底发生了什么：

inspect_open.c:

```
#define _GNU_SOURCE
#include <dlfcn.h>
 
typedef int (*orig_open_f_type)(const char *pathname, int flags);
 
int open(const char *pathname, int flags, ...)
{
    /* Some evil injected code goes here. */
 
    orig_open_f_type orig_open;
    orig_open = (orig_open_f_type)dlsym(RTLD_NEXT,"open");
    return orig_open(pathname,flags);
}
```

_dlfcn.h_ 是被 _dlsym_ 函数所需要，我们在后面会用到它。那个奇怪的 _#define_ 是命令编译器去允许一些非标准的东西，我们需要它去启用 _dlfcn.h_ 中的 `RTLD_NEXT`。那个 typedef 只是创建了一个函数指针类型的别名，它的参数是原始的 open – 别名是 `orig_open_f_type`，我们将在后面用到它。

我们定制的 open(…) 的主体是由一些代码构成。它的最后部分创建了一个新的函数指针 `orig_open`，它指向原始的 open(…) 函数。为了得到那个函数的地址，我们请求 _dlsym_ 去为我们查找，接下来的 “open” 函数在动态库栈上。最后，我们调用了那个函数（传递了与我们的假冒 ”open" 一样的参数），并且返回它的返回值。

我使用下面的内容作为我的 “邪恶的注入代码”：

inspect_open.c (fragment):

```
printf("The victim used open(...) to access '%s'!!!\n",pathname); //remember to include stdio.h!
```

去完成它，我需要稍微调整一下编译参数：

> ```
> gcc -shared -fPIC  inspect_open.c -o inspect_open.so -ldl
> ```

我增加了 _-ldl_ ，因此，它将共享库连接 _libdl_ ，它提供了 _dlsym_ 函数。（不，我还没有创建一个假冒版的 _dlsym_ ，不过这样更有趣）

因此，结果是什么呢？一个共享库，它实现了 open(…) 函数，除了它 _输出_ 文件路径以外，其它的表现和真正的  open(…) 函数 **一模一样**。:-)

如果这个强大的诀窍还没有说服你，是时候去尝试下面的这个示例了：

> ```
> LD_PRELOAD=$PWD/inspect_open.so gnome-calculator
> ```

我鼓励你去看自己实验的结果，但是基本上，它实时列出了这个应用程序可以访问到的每个文件。

我相信它并不难去想像，为什么这可以用于去调试或者研究未知的应用程序。请注意，那只是部分的诀窍，并不是全部，因此 _open()_ 不仅是一个打开文件的函数 … 例如，在标准库中也有一个 _open64()_ ，并且为了完整地研究，你也需要为它去创建一个假冒的。

#### **可能的用法**

如果你一直跟着我享受上面的过程，让我推荐一个使用这个诀窍能做什么的一大堆创意。记住，你可以在不损害原始应用程序的同时做任何你想做的事情！

1.  ~~获得 root 权限~~ 你想多了！你不会通过这种方法绕过安全机制的。（一个专业的解释是：如果 ruid != euid，库不会通过这种方法预加载的。）

2.  欺骗游戏：**取消随机化** 这是我演示的第一个示例。对于一个完整的工作案例，你将需要去实现一个定制的 `random()` 、`rand_r()`、`random_r()`，也有一些应用程序是从`/dev/urandom` 中读取，或者，因此你可以通过使用一个修改的文件路径去运行原始的 `open()` 重定向它们到 `/dev/null`。而且，一些应用程序可能有它们自己的随机数生成算法，这种情况下你似乎是没有办法的（除非，按下面的第 10 点去操作）。但是对于一个新手来说，它看起来并不容易上手。

3.  欺骗游戏：**子弹时间** 实现所有的与标准时间有关的函数，让假冒的时间变慢两倍，或者十倍。如果你为时间测量正确地计算了新值，与时间相关的 `sleep` 函数、和其它的、受影响的应用程序将相信这个时间，（根据你的愿望）运行的更慢（或者更快），并且，你可以体验可怕的 “子弹时间” 的动作。或者 **甚至更进一步**，让你的共享库也可以成为一个 DBus 客户端，因此你可以使用它进行实时的通讯。绑定一些快捷方式到定制的命令，并且在你的假冒的时间函数上使用一些额外的计算，让你可以有能力按你的意愿去启用&禁用慢进或者快进任何时间。

4.  研究应用程序：**列出访问的文件** 它是我演示的第二个示例，但是这也可以进一步去深化，通过记录和监视所有应用程序的文件 I/O。

5.  研究应用程序：**监视因特网访问** 你可以使用 Wireshark 或者类似软件达到这一目的，但是，使用这个诀窍你可以真实地获得控制应用程序基于 web 发送了什么，而不仅是看看，但是也会影响到数据的交换。这里有很多的可能性，从检测间谍软件到欺骗多用户游戏，或者分析&& 逆向工程使用闭源协议的应用程序。

6.  研究应用程序：**检查 GTK 结构** 为什么只局限于标准库？让我们在所有的 GTK 调用中注入一些代码，因此我们可以学习到一个应用程序使用的那些我们并不知道的玩意儿，并且，知道它们的构成。然后这可以渲染出一个图像或者甚至是一个 gtkbuilder 文件！如果你想去学习怎么去做一些应用程序的接口管理，这个方法超级有用！

7.  **在沙盒中运行不安全的应用程序** 如果你不信任一些应用程序，并且你可能担心它会做一些如 `rm -rf /`或者一些其它的不希望的文件激活，你可以通过修改它传递到所有文件相关的函数（不仅是 _open_ ，它也可以删除目录），去重定向它所有的文件 I/O 到诸如 `/tmp` 这里。还有更难的如 chroot 的诀窍，但是它也给你提供更多的控制。它会和完全 “封装” 一样安全，并且除了你真正知道做了什么以外，这种方法不会真实的运行任何恶意软件。

8.  **实现特性** [zlibc][1] 是明确以这种方法运行的一个真实的库；它可以在访问时解压文件，因此，任何应用程序都可以在无需实现解压功能的情况下访问压缩数据。

9.  **修复 bugs** 另一个现实中的示例是：不久前（我不确定现在是否仍然如此）Skype – 它是闭源的软件 – 从某些网络摄像头中捕获视频有问题。因为 Skype 并不是自由软件，源文件不能被修改，就可以通过使用预加载一个解决了这个问题的库的方式来修复这个 bug。

10.  手工方式 **访问应用程序拥有的内存**。请注意，你可以通过这种方式去访问所有应用程序的数据。如果你有类似的软件，如 CheatEngine/scanmem/GameConqueror 这可能并不会让人惊讶，但是，它们都要求 root 权限才能工作。LD_PRELOAD 不需要。事实上，通过一些巧妙的诀窍，你注入的代码可以访问任何应用程序的内存，从本质上看，是因为它是通过应用程序自身来得以运行的。你可以在应用程序可以达到的范围之内通过修改它做任何的事情。你可以想像一下，它允许你做许多的低级别的侵入 … ，但是，关于这个主题，我将在某个时候写一篇关于它的文章。

这里仅是一些我想到的创意。我希望你能找到更多，如果你做到了 – 通过下面的评论区共享出来吧！

--------------------------------------------------------------------------------

via: https://rafalcieslak.wordpress.com/2013/04/02/dynamic-linker-tricks-using-ld_preload-to-cheat-inject-features-and-investigate-programs/

作者：[Rafał Cieślak][a]
译者：[qhwdw](https://github.com/qhwdw)
校对：[校对者ID](https://github.com/校对者ID)

本文由 [LCTT](https://github.com/LCTT/TranslateProject) 原创编译，[Linux中国](https://linux.cn/) 荣誉推出

[a]:https://rafalcieslak.wordpress.com/
[1]:http://www.zlibc.linux.lu/index.html


