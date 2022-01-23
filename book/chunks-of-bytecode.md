> If you find that you're spending almost all your time on theory, start turning
> some attention to practical things; it will improve your theories. If you find
> that you're spending almost all your time on practice, start turning some
> attention to theoretical things; it will improve your practice.
>
> <cite>Donald Knuth</cite>

We already have ourselves a complete implementation of Lox with jlox, so why
isn't the book over yet? Part of this is because jlox relies on the <span
name="metal">JVM</span> to do lots of things for us. If we want to understand
how an interpreter works all the way down to the metal, we need to build those
bits and pieces ourselves.

我们已经用 jlox 完成了 Lox 的实现，为什么这本书还没有完结呢？
部分原因是 jlox 依赖JVM为我们做很多事情。
如果我们想了解解释器是如何工作的，我们需要自己构建这些点点滴滴。

<aside name="metal">

Of course, our second interpreter relies on the C standard library for basics
like memory allocation, and the C compiler frees us from details of the
underlying machine code we're running it on. Heck, that machine code is probably
implemented in terms of microcode on the chip. And the C runtime relies on the
operating system to hand out pages of memory. But we have to stop *somewhere* if
this book is going to fit on your bookshelf.

当然，我们的第二个解释器依赖于 C 标准库来实现内存分配等基础知识，
而 C 编译器将我们从运行它的底层机器代码的细节中解放出来。
哎呀，该机器代码可能是根据芯片上的微代码实现的。 C 运行时依赖于操作系统来分发内存页面。
但是，如果这本书要放在你的书架上，我们就必须在某个地方停下来。

</aside>

An even more fundamental reason that jlox isn't sufficient is that it's too damn
slow. A tree-walk interpreter is fine for some kinds of high-level, declarative
languages. But for a general-purpose, imperative language -- even a "scripting"
language like Lox -- it won't fly. Take this little script:

jlox 不够用的一个更根本的原因是它太慢了。 Tree-walk 解释器适用于某些高级的声明性语言。
但是对于通用的命令式语言——即使是像 Lox 这样的“脚本”语言——它也行不通。拿这个小脚本:

```lox
fun fib(n) {
  if (n < 2) return n;
  return fib(n - 1) + fib(n - 2); // [fib]
}

var before = clock();
print fib(40);
var after = clock();
print after - before;
```

<aside name="fib">

This is a comically inefficient way to actually calculate Fibonacci numbers.
Our goal is to see how fast the *interpreter* runs, not to see how fast of a
program we can write. A slow program that does a lot of work -- pointless or not
-- is a good test case for that.

这是实际计算斐波那契数的一种非常低效的方法。
我们的目标是查看解释器的运行速度，而不是查看我们可以编写多快的程序。
一个做很多工作的慢程序——不管是否毫无意义——是一个很好的测试用例。

</aside>

On my laptop, that takes jlox about 72 seconds to execute. An equivalent C
program finishes in half a second. Our dynamically typed scripting language is
never going to be as fast as a statically typed language with manual memory
management, but we don't need to settle for more than *two orders of magnitude*
slower.

在我的笔记本电脑上，jlox 大约需要 72 秒才能执行。一个等效的 C 程序在半秒内完成。
我们的动态类型脚本语言永远不会像具有手动内存管理的静态类型语言那样快，
但我们不需要满足于慢两个数量级以上。

We could take jlox and run it in a profiler and start tuning and tweaking
hotspots, but that will only get us so far. The execution model -- walking the
AST -- is fundamentally the wrong design. We can't micro-optimize that to the
performance we want any more than you can polish an AMC Gremlin into an SR-71
Blackbird.

我们可以使用 jlox 并在分析器中运行它，然后开始调整和调整热点，但这只会让我们到目前为止。
执行模型——走 AST——从根本上是错误的设计。
我们无法对其进行微优化以达到我们想要的性能，就像您无法将 AMC Gremlin 打磨成 SR-71 Blackbird 一样。

We need to rethink the core model. This chapter introduces that model, bytecode,
and begins our new interpreter, clox.

我们需要重新思考核心模型。本章介绍该模型、字节码，并开始我们的新解释器 clox。

## Bytecode?

In engineering, few choices are without trade-offs. To best understand why we're
going with bytecode, let's stack it up against a couple of alternatives.

在工程中，很少有选择是没有权衡的。
为了更好地理解我们为什么要使用字节码，让我们将它与几个替代方案进行比较。

### Why not walk the AST?

Our existing interpreter has a couple of things going for it:

*   Well, first, we already wrote it. It's done. And the main reason it's done
    is because this style of interpreter is *really simple to implement*. The
    runtime representation of the code directly maps to the syntax. It's
    virtually effortless to get from the parser to the data structures we need
    at runtime.

*   It's *portable*. Our current interpreter is written in Java and runs on any
    platform Java supports. We could write a new implementation in C using the
    same approach and compile and run our language on basically every platform
    under the sun.

我们现有的解释器有几件事要做：

*   好吧，首先，我们已经写好了。完成。这样做的主要原因是因为这种风格的解释器实现起来非常简单。
    代码的运行时表示直接映射到语法。从解析器到我们在运行时需要的数据结构几乎毫不费力。

*   它是便携式的。我们当前的解释器是用 Java 编写的，可以在 Java 支持的任何平台上运行。
    我们可以使用相同的方法在 C 中编写一个新的实现，并在几乎所有平台上编译和运行我们的语言。

Those are real advantages. But, on the other hand, it's *not memory-efficient*.
Each piece of syntax becomes an AST node. A tiny Lox expression like `1 + 2`
turns into a slew of objects with lots of pointers between them, something like:

这些都是真正的优势。但是，另一方面，它的内存效率不高。每一段语法都成为一个 AST 节点。
像 `1 + 2` 这样的小型 Lox 表达式会变成一堆对象，它们之间有很多指针，例如：

<span name="header"></span>

<aside name="header">

The "(header)" parts are the bookkeeping information the Java virtual machine
uses to support memory management and store the object's type. Those take up
space too!

“(header)”部分是 Java 虚拟机用来支持内存管理和存储对象类型的簿记信息。那些也占用空间！

</aside>

<img src="image/chunks-of-bytecode/ast.png" alt="The tree of Java objects created to represent '1 + 2'." />

Each of those pointers adds an extra 32 or 64 bits of overhead to the object.
Worse, sprinkling our data across the heap in a loosely connected web of objects
does bad things for <span name="locality">*spatial locality*</span>.

这些指针中的每一个都为对象增加了额外的 32 或 64 位开销。
更糟糕的是，将我们的数据散布在松散连接的对象网络中的堆中对空间局部性不利。

<aside name="locality">

I wrote [an entire chapter][gpp locality] about this exact problem in my first
book, *Game Programming Patterns*, if you want to really dig in.

如果您想真正深入研究，我在我的第一本书《游戏编程模式》中就这个确切的问题写了[一整章][gpp locality]。

[gpp locality]: http://gameprogrammingpatterns.com/data-locality.html

</aside>

Modern CPUs process data way faster than they can pull it from RAM. To
compensate for that, chips have multiple layers of caching. If a piece of memory
it needs is already in the cache, it can be loaded more quickly. We're talking
upwards of 100 *times* faster.

现代 CPU 处理数据的速度比从 RAM 中提取数据的速度要快。
为了弥补这一点，芯片具有多层缓存。如果它需要的一块内存已经在缓存中，它可以更快地加载。
我们说话的速度提高了 100 倍以上。

How does data get into that cache? The machine speculatively stuffs things in
there for you. Its heuristic is pretty simple. Whenever the CPU reads a bit of
data from RAM, it pulls in a whole little bundle of adjacent bytes and stuffs
them in the cache.

数据如何进入该缓存？机器投机地为你塞东西。它的启发式非常简单。
每当 CPU 从 RAM 中读取一点数据时，它都会拉入一大堆相邻的字节并将它们填充到缓存中。

If our program next requests some data close enough to be inside that cache
line, our CPU runs like a well-oiled conveyor belt in a factory. We *really*
want to take advantage of this. To use the cache effectively, the way we
represent code in memory should be dense and ordered like it's read.

如果我们的程序接下来请求一些足够接近该缓存行的数据，我们的 CPU 就像工厂中润滑良好的传送带一样运行。
我们真的很想利用这一点。为了有效地使用缓存，
我们在内存中表示代码的方式应该是密集和有序的，就像它被读取一样。

Now look up at that tree. Those sub-objects could be <span
name="anywhere">*anywhere*</span>. Every step the tree-walker takes where it
follows a reference to a child node may step outside the bounds of the cache and
force the CPU to stall until a new lump of data can be slurped in from RAM. Just
the *overhead* of those tree nodes with all of their pointer fields and object
headers tends to push objects away from each other and out of the cache.

现在抬头看看那棵树。这些子对象可以是anywhere。 
tree-walker 在引用子节点时所采取的每一步都可能超出缓存的范围并迫使 CPU 停止，
直到可以从 RAM 中获取新的数据块。
只是这些树节点及其所有指针字段和对象标头的开销往往会将对象推离彼此并移出缓存。

<aside name="anywhere">

Even if the objects happened to be allocated in sequential memory when the
parser first produced them, after a couple of rounds of garbage collection --
which may move objects around in memory -- there's no telling where they'll be.

即使在解析器第一次生成对象时这些对象碰巧被分配在顺序内存中，
经过几轮垃圾收集（可能会在内存中移动对象）之后，也不知道它们会在哪里。

</aside>

Our AST walker has other overhead too around interface dispatch and the Visitor
pattern, but the locality issues alone are enough to justify a better code
representation.

我们的 AST walker 在接口调度和访问者模式方面也有其他开销，
但仅局部性问题就足以证明更好的代码表示是合理的。

### Why not compile to native code?

If you want to go *real* fast, you want to get all of those layers of
indirection out of the way. Right down to the metal. Machine code. It even
*sounds* fast. *Machine code.*

如果您想真正快速地进行，您希望将所有这些间接层排除在外。
一直到金属。机器码。它甚至听起来很快。机器码。

Compiling directly to the native instruction set the chip supports is what the
fastest languages do. Targeting native code has been the most efficient option
since way back in the early days when engineers actually <span
name="hand">handwrote</span> programs in machine code.

直接编译到芯片支持的本机指令集是最快的语言所做的。
早在早期工程师实际上手写机器代码程序以来，针对本机代码一直是最有效的选择。

<aside name="hand">

Yes, they actually wrote machine code by hand. On punched cards. Which,
presumably, they punched *with their fists*.

是的，他们实际上是手工编写机器代码。在打孔卡上。大概是他们用拳头打的。

</aside>

If you've never written any machine code, or its slightly more human-palatable
cousin assembly code before, I'll give you the gentlest of introductions. Native
code is a dense series of operations, encoded directly in binary. Each
instruction is between one and a few bytes long, and is almost mind-numbingly
low level. "Move a value from this address to this register." "Add the integers
in these two registers." Stuff like that.

如果您以前从未编写过任何机器代码，或者它更易于人类接受的汇编代码，我将为您提供最温和的介绍。
本机代码是一系列密集的操作，直接以二进制编码。
每条指令的长度在一个到几个字节之间，并且几乎是令人麻木的低级。 
“将一个值从这个地址移动到这个寄存器。” “将这两个寄存器中的整数相加。”类似的东西。

The CPU cranks through the instructions, decoding and executing each one in
order. There is no tree structure like our AST, and control flow is handled by
jumping from one point in the code directly to another. No indirection, no
overhead, no unnecessary skipping around or chasing pointers.

CPU 处理指令，依次解码和执行每一条指令。
没有像我们的 AST 这样的树形结构，控制流是通过从代码中的一个点直接跳转到另一个点来处理的。
没有间接，没有开销，没有不必要的跳过或追逐指针。

Lightning fast, but that performance comes at a cost. First of all, compiling to
native code ain't easy. Most chips in wide use today have sprawling Byzantine
architectures with heaps of instructions that accreted over decades. They
require sophisticated register allocation, pipelining, and instruction
scheduling.

闪电般的速度，但这种性能是有代价的。首先，编译为本机代码并不容易。
当今广泛使用的大多数芯片都具有庞大的拜占庭式架构，其中包含数十年积累的大量指令。
它们需要复杂的寄存器分配、流水线和指令调度。

And, of course, you've thrown <span name="back">portability</span> out. Spend a
few years mastering some architecture and that still only gets you onto *one* of
the several popular instruction sets out there. To get your language on all of
them, you need to learn all of their instruction sets and write a separate back
end for each one.

而且，当然，您已经抛弃了便携性。花几年时间掌握一些架构，但仍然只能让你掌握几种流行的指令集之一。
为了让你的语言掌握所有这些，你需要学习他们所有的指令集，并为每个指令集编写一个单独的后端。

<aside name="back">

The situation isn't entirely dire. A well-architected compiler lets you
share the front end and most of the middle layer optimization passes across the
different architectures you support. It's mainly the code generation and some of
the details around instruction selection that you'll need to write afresh each
time.

情况并不完全可怕。一个架构良好的编译器可以让你在你支持的不同架构之间共享前端和大部分中间层优化。
主要是代码生成和指令选择的一些细节，您每次都需要重新编写。

The [LLVM][] project gives you some of this out of the box. If your compiler
outputs LLVM's own special intermediate language, LLVM in turn compiles that to
native code for a plethora of architectures.

[LLVM][] 项目为您提供了一些开箱即用的功能。如果您的编译器输出 LLVM 自己的特殊中间语言，
LLVM 会依次将其编译为大量架构的本机代码。

[llvm]: https://llvm.org/

</aside>

### What is bytecode?

Fix those two points in your mind. On one end, a tree-walk interpreter is
simple, portable, and slow. On the other, native code is complex and
platform-specific but fast. Bytecode sits in the middle. It retains the
portability of a tree-walker -- we won't be getting our hands dirty with
assembly code in this book. It sacrifices *some* simplicity to get a performance
boost in return, though not as fast as going fully native.

把这两点牢牢记在心里。一方面，tree-walk 解释器简单、便携且速度慢。
另一方面，本机代码复杂且特定于平台但速度很快。字节码位于中间。
它保留了tree-walker 的可移植性——我们不会在本书中接触到汇编代码。
它牺牲了一些简单性来获得性能提升作为回报，尽管速度不如完全原生。

Structurally, bytecode resembles machine code. It's a dense, linear sequence of
binary instructions. That keeps overhead low and plays nice with the cache.
However, it's a much simpler, higher-level instruction set than any real chip
out there. (In many bytecode formats, each instruction is only a single byte
long, hence "bytecode".)

在结构上，字节码类似于机器码。这是一个密集的线性二进制指令序列。
这样可以保持低开销并且与缓存配合得很好。
然而，它是一个比任何真正的芯片更简单、更高级别的指令集。 
（在许多字节码格式中，每条指令只有一个字节长，因此是“字节码”。）

Imagine you're writing a native compiler from some source language and you're
given carte blanche to define the easiest possible architecture to target.
Bytecode is kind of like that. It's an idealized fantasy instruction set that
makes your life as the compiler writer easier.

想象一下，您正在使用某种源语言编写本机编译器，并且全权委托您定义最简单的目标架构。
字节码就是这样。这是一个理想化的幻想指令集，让您作为编译器编写者的生活更轻松。

The problem with a fantasy architecture, of course, is that it doesn't exist. We
solve that by writing an *emulator* -- a simulated chip written in software that
interprets the bytecode one instruction at a time. A *virtual machine (VM)*, if
you will.

当然，幻想建筑的问题在于它不存在。
我们通过编写一个模拟器来解决这个问题——一个用软件编写的模拟芯片，一次解释一条指令的字节码。
一个虚拟机 (VM)，如果你愿意的话。

That emulation layer adds <span name="p-code">overhead</span>, which is a key
reason bytecode is slower than native code. But in return, it gives us
portability. Write our VM in a language like C that is already supported on all
the machines we care about, and we can run our emulator on top of any hardware
we like.

该仿真层增加了开销，这是字节码比本机代码慢的一个关键原因。
但作为回报，它给了我们便携性。
用我们关心的所有机器都支持的像 C 这样的语言编写我们的 VM，
我们可以在我们喜欢的任何硬件上运行我们的模拟器。

<aside name="p-code">

One of the first bytecode formats was [p-code][], developed for Niklaus Wirth's
Pascal language. You might think a PDP-11 running at 15MHz couldn't afford the
overhead of emulating a virtual machine. But back then, computers were in their
Cambrian explosion and new architectures appeared every day. Keeping up with the
latest chips was worth more than squeezing the maximum performance from each
one. That's why the "p" in p-code doesn't stand for "Pascal", but "portable".

最早的字节码格式之一是 [p-code][]，它是为 Niklaus Wirth 的 Pascal 语言开发的。
您可能认为以 15MHz 运行的 PDP-11 无法承受模拟虚拟机的开销。
但那时，计算机正处于寒武纪爆发期，每天都有新的架构出现。
跟上最新的芯片比从每个芯片中获得最大性能更有价值。
这就是为什么 p-code 中的“p”不代表“Pascal”，而是“portable”。

[p-code]: https://en.wikipedia.org/wiki/P-code_machine

</aside>

This is the path we'll take with our new interpreter, clox. We'll follow in the
footsteps of the main implementations of Python, Ruby, Lua, OCaml, Erlang, and
others. In many ways, our VM's design will parallel the structure of our
previous interpreter:

这就是我们将采用新的解释器clox 的路径。
我们将追随 Python、Ruby、Lua、OCaml、Erlang 等主要实现的脚步。
在许多方面，我们的 VM 的设计将与我们之前的解释器的结构平行：

<img src="image/chunks-of-bytecode/phases.png" alt="Phases of the two
implementations. jlox is Parser to Syntax Trees to Interpreter. clox is Compiler
to Bytecode to Virtual Machine." />

Of course, we won't implement the phases strictly in order. Like our previous
interpreter, we'll bounce around, building up the implementation one language
feature at a time. In this chapter, we'll get the skeleton of the application in
place and create the data structures needed to store and represent a chunk of
bytecode.

当然，我们不会严格按顺序执行阶段。
像我们之前的解释器一样，我们会四处走动，一次构建一个语言特性的实现。
在本章中，我们将获得应用程序的框架，并创建存储和表示一大块字节码所需的数据结构。

## Getting Started

Where else to begin, but at `main()`? <span name="ready">Fire</span> up your
trusty text editor and start typing.

除了 `main()` 还能从哪里开始呢？ 启动你喜欢的文本编辑器并开始输入。

<aside name="ready">

Now is a good time to stretch, maybe crack your knuckles. A little montage music
wouldn't hurt either.

现在是伸展身体的好时机，也许可以掰开你的指关节。一点蒙太奇音乐也不会受到伤害。

</aside>

^code main-c

From this tiny seed, we will grow our entire VM. Since C provides us with so
little, we first need to spend some time amending the soil. Some of that goes
into this header:

从这个小小的种子中，我们将培育出整个虚拟机。
由于 C 提供给我们的东西很少，我们首先需要花一些时间来修整土壤。
其中一些在头文件中：

^code common-h

There are a handful of types and constants we'll use throughout the interpreter,
and this is a convenient place to put them. For now, it's the venerable `NULL`,
`size_t`, the nice C99 Boolean `bool`, and explicit-sized integer types --
`uint8_t` and friends.

我们将在整个解释器中使用一些类型和常量，这是放置它们的方便位置。
现在，它是古老的 `NULL`、`size_t`、漂亮的 C99 布尔型 `bool` 
和显式大小的整数类型 -- `uint8_t` 和朋友。

## Chunks of Instructions

Next, we need a module to define our code representation. I've been using
"chunk" to refer to sequences of bytecode, so let's make that the official name
for that module.

接下来，我们需要一个模块来定义我们的代码表示。
我一直在使用“块”来指代字节码序列，所以让我们将其作为该模块的正式名称。

^code chunk-h

In our bytecode format, each instruction has a one-byte **operation code**
(universally shortened to **opcode**). That number controls what kind of
instruction we're dealing with -- add, subtract, look up variable, etc. We
define those here:

在我们的字节码格式中，每条指令都有一个单字节的操作码（普遍缩写为操作码）。
这个数字控制着我们正在处理什么样的指令——加法、减法、查找变量等。我们在这里定义它们：

^code op-enum (1 before, 2 after)

For now, we start with a single instruction, `OP_RETURN`. When we have a
full-featured VM, this instruction will mean "return from the current function".
I admit this isn't exactly useful yet, but we have to start somewhere, and this
is a particularly simple instruction, for reasons we'll get to later.

现在，我们从一条指令“OP_RETURN”开始。当我们有一个全功能的虚拟机时，
这个指令将意味着“从当前函数返回”。我承认这还不是很有用，但我们必须从某个地方开始，
这是一个特别简单的指令，原因我们稍后会谈到。

### A dynamic array of instructions

Bytecode is a series of instructions. Eventually, we'll store some other data
along with the instructions, so let's go ahead and create a struct to hold it
all.

字节码是一系列指令。最终，我们将连同指令一起存储一些其他数据，所以让我们继续创建一个结构来保存所有数据。

^code chunk-struct (1 before, 2 after)

At the moment, this is simply a wrapper around an array of bytes. Since we don't
know how big the array needs to be before we start compiling a chunk, it must be
dynamic. Dynamic arrays are one of my favorite data structures. That sounds like
claiming vanilla is my favorite ice cream <span name="flavor">flavor</span>, but
hear me out. Dynamic arrays provide:

目前，这只是一个字节数组的包装器。
由于在开始编译块之前我们不知道数组需要多大，因此它必须是动态的。
动态数组是我最喜欢的数据结构之一。这听起来像是声称香草是我最喜欢的冰淇淋flavor，但请听我说完。
动态数组提供：

<aside name="flavor">

Butter pecan is actually my favorite.

黄油山核桃实际上是我的最爱。

</aside>

* Cache-friendly, dense storage (缓存友好的密集存储)

* Constant-time indexed element lookup (常量时间按索引查找元素)

* Constant-time appending to the end of the array (常量时间添加元素到数组末尾)

Those features are exactly why we used dynamic arrays all the time in jlox under
the guise of Java's ArrayList class. Now that we're in C, we get to roll our
own. If you're rusty on dynamic arrays, the idea is pretty simple. In addition
to the array itself, we keep two numbers: the number of elements in the array we
have allocated ("capacity") and how many of those allocated entries are actually
in use ("count").

这些特性正是我们以 Java 的 ArrayList 类为幌子在 jlox 中一直使用动态数组的原因。
既然我们在 C 中，我们就可以自己动手了。如果您对动态数组不熟悉，那么这个想法很简单。
除了数组本身，我们还保留了两个数字：
我们已分配的数组中的元素数量（“容量”）和实际使用的这些已分配条目的数量（“计数”）。

^code count-and-capacity (1 before, 2 after)

When we add an element, if the count is less than the capacity, then there is
already available space in the array. We store the new element right in there
and bump the count.

当我们添加一个元素时，如果计数小于容量，那么数组中已经有可用空间。
我们将新元素存储在那里并增加计数。

<img src="image/chunks-of-bytecode/insert.png" alt="Storing an element in an
array that has enough capacity." />

If we have no spare capacity, then the process is a little more involved.

如果我们没有多余的容量，那么这个过程就有点复杂了。

<img src="image/chunks-of-bytecode/grow.png" alt="Growing the dynamic array
before storing an element." class="wide" />

1.  <span name="amortized">Allocate</span> a new array with more capacity.
2.  Copy the existing elements from the old array to the new one.
3.  Store the new `capacity`.
4.  Delete the old array.
5.  Update `code` to point to the new array.
6.  Store the element in the new array now that there is room.
7.  Update the `count`.

<aside name="amortized">

Copying the existing elements when you grow the array makes it seem like
appending an element is *O(n)*, not *O(1)* like I said above. However, you need
to do this copy step only on *some* of the appends. Most of the time, there is
already extra capacity, so you don't need to copy.

在扩展数组时复制现有元素会使附加元素看起来像是 O(n)，而不是 O(1)，就像我上面所说的那样。
但是，您只需要对某些附加项执行此复制步骤。大多数时候，已经有额外的容量，所以你不需要复制。

To understand how this works, we need [**amortized
analysis**](https://en.wikipedia.org/wiki/Amortized_analysis). That shows us
that as long as we grow the array by a multiple of its current size, when we
average out the cost of a *sequence* of appends, each append is *O(1)*.

要了解它是如何工作的，我们需要 [摊销分析](https:en.wikipedia.orgwikiAmortized_analysis)。
这向我们表明，只要我们将数组增长其当前大小的倍数，当我们平均一系列追加的成本时，每个追加都是 O(1)。

</aside>

We have our struct ready, so let's implement the functions to work with it. C
doesn't have constructors, so we declare a function to initialize a new chunk.

我们已经准备好结构体，所以让我们实现使用它的函数。 
C 没有构造函数，所以我们声明一个函数来初始化一个新的块。

^code init-chunk-h (1 before, 2 after)

And implement it thusly:

并因此实现它：

^code chunk-c

The dynamic array starts off completely empty. We don't even allocate a raw
array yet. To append a byte to the end of the chunk, we use a new function.

动态数组开始时完全是空的。我们甚至还没有分配原始数组。
为了将一个字节附加到块的末尾，我们使用了一个新函数。

^code write-chunk-h (1 before, 2 after)

This is where the interesting work happens.

这就是有趣的工作发生的地方。

^code write-chunk

The first thing we need to do is see if the current array already has capacity
for the new byte. If it doesn't, then we first need to grow the array to make
room. (We also hit this case on the very first write when the array is `NULL`
and `capacity` is 0.)

我们需要做的第一件事是查看当前数组是否已经具有新字节的容量。
如果没有，那么我们首先需要扩大阵列以腾出空间。 
（当数组为“NULL”且“容量”为 0 时，我们也在第一次写入时遇到了这种情况。）

To grow the array, first we figure out the new capacity and grow the array to
that size. Both of those lower-level memory operations are defined in a new
module.

要扩大阵列，首先我们要计算出新容量并将阵列扩大到该大小。
这两个较低级别的内存操作都在一个新模块中定义。

^code chunk-c-include-memory (1 before, 2 after)

This is enough to get us started.

这足以让我们开始。

^code memory-h

This macro calculates a new capacity based on a given current capacity. In order
to get the performance we want, the important part is that it *scales* based on
the old size. We grow by a factor of two, which is pretty typical. 1.5&times; is
another common choice.

该宏根据给定的当前容量计算新容量。为了获得我们想要的性能，重要的部分是它基于旧尺寸进行缩放。
我们增长了两倍，这是非常典型的。 1.5× 是另一个常见的选择。

We also handle when the current capacity is zero. In that case, we jump straight
to eight elements instead of starting at one. That <span
name="profile">avoids</span> a little extra memory churn when the array is very
small, at the expense of wasting a few bytes on very small chunks.

我们还处理当前容量为零的情况。在这种情况下，我们直接跳到八个元素，而不是从一个开始。
当数组非常小时，避免了一点额外的内存流失，代价是在非常小的块上浪费了几个字节。

<aside name="profile">

I picked the number eight somewhat arbitrarily for the book. Most dynamic array
implementations have a minimum threshold like this. The right way to pick a
value for this is to profile against real-world usage and see which constant
makes the best performance trade-off between extra grows versus wasted space.

我为这本书选择了数字 8 有点武断。大多数动态数组实现都有这样的最小阈值。
为此选择值的正确方法是根据实际使用情况进行分析，
并查看哪个常量在额外增长与浪费空间之间做出最佳性能权衡。

</aside>

Once we know the desired capacity, we create or grow the array to that size
using `GROW_ARRAY()`.

一旦我们知道所需的容量，我们就可以使用 `GROW_ARRAY()` 创建或将数组增长到该大小。

^code grow-array (2 before, 2 after)

This macro pretties up a function call to `reallocate()` where the real work
happens. The macro itself takes care of getting the size of the array's element
type and casting the resulting `void*` back to a pointer of the right type.

这个宏美化了对真正工作发生的`reallocate()`的函数调用。
宏本身负责获取数组元素类型的大小并将生成的“void”转换回正确类型的指针。

This `reallocate()` function is the single function we'll use for all dynamic
memory management in clox -- allocating memory, freeing it, and changing the
size of an existing allocation. Routing all of those operations through a single
function will be important later when we add a garbage collector that needs to
keep track of how much memory is in use.

这个 `reallocate()` 函数是我们将用于 clox 中的所有动态内存管理的单个函数——
分配内存、释放它以及更改现有分配的大小。
当我们添加一个需要跟踪正在使用的内存量的垃圾收集器时，通过单个函数路由所有这些操作将很重要。

The two size arguments passed to `reallocate()` control which operation to
perform:

传递给 `reallocate()` 的两个大小参数控制要执行的操作：

<table>
  <thead>
    <tr>
      <td>oldSize</td>
      <td>newSize</td>
      <td>Operation</td>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>0</td>
      <td>Non&#8209;zero</td>
      <td>Allocate new block.</td>
    </tr>
    <tr>
      <td>Non&#8209;zero</td>
      <td>0</td>
      <td>Free allocation.</td>
    </tr>
    <tr>
      <td>Non&#8209;zero</td>
      <td>Smaller&nbsp;than&nbsp;<code>oldSize</code></td>
      <td>Shrink existing allocation.</td>
    </tr>
    <tr>
      <td>Non&#8209;zero</td>
      <td>Larger&nbsp;than&nbsp;<code>oldSize</code></td>
      <td>Grow existing allocation.</td>
    </tr>
  </tbody>
</table>

That sounds like a lot of cases to handle, but here's the implementation:

这听起来需要处理很多情况，但这里是实现：

^code memory-c

When `newSize` is zero, we handle the deallocation case ourselves by calling
`free()`. Otherwise, we rely on the C standard library's `realloc()` function.
That function conveniently supports the other three aspects of our policy. When
`oldSize` is zero, `realloc()` is equivalent to calling `malloc()`.

当 `newSize` 为零时，我们通过调用 `free()` 自己处理释放情况。
否则，我们依赖 C 标准库的 `realloc()` 函数。
该功能方便地支持我们政策的其他三个方面。当 `oldSize` 为零时，`realloc()` 等价于调用 `malloc()`。

The interesting cases are when both `oldSize` and `newSize` are not zero. Those
tell `realloc()` to resize the previously allocated block. If the new size is
smaller than the existing block of memory, it simply <span
name="shrink">updates</span> the size of the block and returns the same pointer
you gave it. If the new size is larger, it attempts to grow the existing block
of memory.

有趣的情况是 `oldSize` 和 `newSize` 都不为零。
这些告诉 `realloc()` 调整先前分配的块的大小。
如果新的大小小于现有的内存块，它只是更新块的大小并返回你给它的相同指针。
如果新的大小更大，它会尝试增加现有的内存块。

It can do that only if the memory after that block isn't already in use. If
there isn't room to grow the block, `realloc()` instead allocates a *new* block
of memory of the desired size, copies over the old bytes, frees the old block,
and then returns a pointer to the new block. Remember, that's exactly the
behavior we want for our dynamic array.

只有在该块之后的内存尚未使用时，它才能做到这一点。
如果没有空间来增长块，`realloc()` 会分配一个所需大小的新内存块，复制旧字节，
释放旧块，然后返回指向新块的指针。请记住，这正是我们想要的动态数组的行为。

Because computers are finite lumps of matter and not the perfect mathematical
abstractions computer science theory would have us believe, allocation can fail
if there isn't enough memory and `realloc()` will return `NULL`. We should
handle that.

因为计算机是有限的物质块，而不是计算机科学理论让我们相信的完美数学抽象，
所以如果没有足够的内存，分配可能会失败，而 `realloc()` 将返回 `NULL`。我们应该处理它。

^code out-of-memory (1 before, 1 after)

There's not really anything *useful* that our VM can do if it can't get the
memory it needs, but we at least detect that and abort the process immediately
instead of returning a `NULL` pointer and letting it go off the rails later.

如果虚拟机无法获得所需的内存，它实际上并没有什么用处，但我们至少检测到这一点并立即中止进程，
而不是返回一个“NULL”指针并让它在以后脱离轨道。

<aside name="shrink">

Since all we passed in was a bare pointer to the first byte of memory, what does
it mean to "update" the block's size? Under the hood, the memory allocator
maintains additional bookkeeping information for each block of heap-allocated
memory, including its size.

由于我们传入的只是指向内存第一个字节的裸指针，“更新”块的大小是什么意思？
在底层，内存分配器为每个堆分配内存块维护额外的簿记信息，包括其大小。

Given a pointer to some previously allocated memory, it can find this
bookkeeping information, which is necessary to be able to cleanly free it. It's
this size metadata that `realloc()` updates.

给定一个指向一些先前分配的内存的指针，它可以找到这个簿记信息，这是能够干净地释放它所必需的。
`realloc()` 更新的是这个大小的元数据。

Many implementations of `malloc()` store the allocated size in memory right
*before* the returned address.

`malloc()` 的许多实现在返回地址之前将分配的大小存储在内存中。

</aside>

OK, we can create new chunks and write instructions to them. Are we done? Nope!
We're in C now, remember, we have to manage memory ourselves, like in Ye Olden
Times, and that means *freeing* it too.

好的，我们可以创建新的块并向它们写入指令。我们完了吗？不！我们现在在 C 中，
记住，我们必须自己管理内存，就像在 Ye Olden Times 中一样，这也意味着释放它。

^code free-chunk-h (1 before, 1 after)

The implementation is:

实现是：

^code free-chunk

We deallocate all of the memory and then call `initChunk()` to zero out the
fields leaving the chunk in a well-defined empty state. To free the memory, we
add one more macro.

我们释放所有内存，然后调用“initChunk()”将字段清零，使块处于明确定义的空状态。
为了释放内存，我们再添加一个宏。

^code free-array (3 before, 2 after)

Like `GROW_ARRAY()`, this is a wrapper around a call to `reallocate()`. This one
frees the memory by passing in zero for the new size. I know, this is a lot of
boring low-level stuff. Don't worry, we'll get a lot of use out of these in
later chapters and will get to program at a higher level. Before we can do that,
though, we gotta lay our own foundation.

像 `GROW_ARRAY()` 一样，这是对 `reallocate()` 调用的包装。这个通过为新大小传递零来释放内存。
我知道，这是很多无聊的低级内容。不用担心，我们会在后面的章节中大量使用这些，
并且会在更高的层次上进行编程。不过，在我们能够做到这一点之前，我们必须先打好自己的基础。

## Disassembling Chunks

Now we have a little module for creating chunks of bytecode. Let's try it out by
hand-building a sample chunk.

现在我们有了一个用于创建字节码块的小模块。让我们通过手工构建一个示例块来尝试一下。

^code main-chunk (1 before, 1 after)

Don't forget the include.

不要忘记包含。

^code main-include-chunk (1 before, 2 after)

Run that and give it a try. Did it work? Uh... who knows? All we've done is push
some bytes around in memory. We have no human-friendly way to see what's
actually inside that chunk we made.

运行它并试一试。它奏效了吗？呃……谁知道？我们所做的只是在内存中推送一些字节。
我们没有人性化的方式来查看我们制作的那个块中的实际内容。

To fix this, we're going to create a **disassembler**. An **assembler** is an
old-school program that takes a file containing human-readable mnemonic names
for CPU instructions like "ADD" and "MULT" and translates them to their binary
machine code equivalent. A *dis*assembler goes in the other direction -- given a
blob of machine code, it spits out a textual listing of the instructions.

为了解决这个问题，我们将创建一个反汇编程序。汇编器是一个老式程序，
它获取一个包含人类可读的 CPU 指令（如“ADD”和“MULT”）的助记符名称的文件，
并将它们转换为等效的二进制机器代码。
反汇编程序则朝另一个方向发展——给定一团机器代码，它会输出指令的文本列表。

We'll implement something <span name="printer">similar</span>. Given a chunk, it
will print out all of the instructions in it. A Lox *user* won't use this, but
we Lox *maintainers* will certainly benefit since it gives us a window into the
interpreter's internal representation of code.

我们将实现一些类似的。给定一个块，它将打印出其中的所有指令。 Lox 用户不会使用它，但我们 Lox 维护者肯定会受益，因为它为我们提供了一个了解解释器内部代码表示的窗口。

<aside name="printer">

In jlox, our analogous tool was the [AstPrinter class][].

在 jlox 中，我们的类似工具是 [AstPrinter 类][]。

[astprinter class]: representing-code.html#a-not-very-pretty-printer

</aside>

In `main()`, after we create the chunk, we pass it to the disassembler.

在 `main()` 中，创建块后，我们将其传递给反汇编程序。

^code main-disassemble-chunk (2 before, 1 after)

Again, we whip up <span name="module">yet another</span> module.

再次，我们掀起了另一个模块。

<aside name="module">

I promise you we won't be creating this many new files in later chapters.

我向你保证，我们不会在后面的章节中创建这么多新文件。

</aside>

^code main-include-debug (1 before, 2 after)

Here's that header:

这是头文件：

^code debug-h

In `main()`, we call `disassembleChunk()` to disassemble all of the instructions
in the entire chunk. That's implemented in terms of the other function, which
just disassembles a single instruction. It shows up here in the header because
we'll call it from the VM in later chapters.

在 `main()` 中，我们调用 `disassembleChunk()` 来反汇编整个块中的所有指令。
这是根据另一个功能实现的，它只是反汇编一条指令。
它显示在标题中，因为我们将在后面的章节中从 VM 中调用它。

Here's a start at the implementation file:

这是实现文件的开始：

^code debug-c

To disassemble a chunk, we print a little header (so we can tell *which* chunk
we're looking at) and then crank through the bytecode, disassembling each
instruction. The way we iterate through the code is a little odd. Instead of
incrementing `offset` in the loop, we let `disassembleInstruction()` do it for
us. When we call that function, after disassembling the instruction at the given
offset, it returns the offset of the *next* instruction. This is because, as
we'll see later, instructions can have different sizes.

为了反汇编一个块，我们打印一个小标题（这样我们就可以知道我们正在查看哪个块），
然后通过字节码来反汇编每条指令。我们遍历代码的方式有点奇怪。
我们让 `disassembleInstruction()` 为我们完成，而不是在循环中增加 `offset`。
当我们调用该函数时，在给定偏移量处反汇编指令后，它返回下一条指令的偏移量。
这是因为，正如我们稍后将看到的，指令可以有不同的大小。

The core of the "debug" module is this function:

“调试”模块的核心是这个函数：

^code disassemble-instruction

First, it prints the byte offset of the given instruction -- that tells us where
in the chunk this instruction is. This will be a helpful signpost when we start
doing control flow and jumping around in the bytecode.

首先，它打印给定指令的字节偏移量——它告诉我们这条指令在块中的什么位置。
当我们开始进行控制流并在字节码中跳转时，这将是一个有用的路标。

Next, it reads a single byte from the bytecode at the given offset. That's our
opcode. We <span name="switch">switch</span> on that. For each kind of
instruction, we dispatch to a little utility function for displaying it. On the
off chance that the given byte doesn't look like an instruction at all -- a bug
in our compiler -- we print that too. For the one instruction we do have,
`OP_RETURN`, the display function is:

接下来，它从给定偏移量的字节码中读取一个字节。那是我们的操作码。我们对此switch。
对于每一种指令，我们派发一个小实用函数来显示它。
如果给定的字节看起来根本不像一条指令——我们编译器中的一个错误——我们也会打印它。
对于我们有的一条指令，`OP_RETURN`，显示函数是：

<aside name="switch">

We have only one instruction right now, but this switch will grow throughout the
rest of the book.

我们现在只有一个指令，但这个转换将在本书的其余部分中不断增加。

</aside>

^code simple-instruction

There isn't much to a return instruction, so all it does is print the name of
the opcode, then return the next byte offset past this instruction. Other
instructions will have more going on.

return 指令没有太多内容，所以它所做的只是打印操作码的名称，
然后返回该指令之后的下一个字节偏移量。其他说明将有更多内容。

If we run our nascent interpreter now, it actually prints something:

如果我们现在运行我们新生的解释器，它实际上会打印一些东西：

```text
== test chunk ==
0000 OP_RETURN
```

It worked! This is sort of the "Hello, world!" of our code representation. We
can create a chunk, write an instruction to it, and then extract that
instruction back out. Our encoding and decoding of the binary bytecode is
working.

有效！这有点像“你好，世界！”我们的代码表示。我们可以创建一个块，向其写入指令，然后将该指令提取出来。
我们对二进制字节码的编码和解码工作正常。

## Constants

Now that we have a rudimentary chunk structure working, let's start making it
more useful. We can store *code* in chunks, but what about *data*? Many values
the interpreter works with are created at runtime as the result of operations.

现在我们已经有了一个基本的块结构，让我们开始让它更有用。
我们可以将代码存储在块中，但是数据呢？解释器使用的许多值是在运行时作为操作的结果创建的。

```lox
1 + 2;
```

The value 3 appears nowhere in the code here. However, the literals `1` and `2`
do. To compile that statement to bytecode, we need some sort of instruction that
means "produce a constant" and those literal values need to get stored in the
chunk somewhere. In jlox, the Expr.Literal AST node held the value. We need a
different solution now that we don't have a syntax tree.

值 3 没有出现在此处的代码中。但是，文字 `1` 和 `2` 可以。
为了将该语句编译为字节码，我们需要某种指令，意思是“产生一个常量”，
并且这些文字值需要存储在某处的块中。在 jlox 中，Expr.Literal AST 节点保存该值。
现在我们需要一个不同的解决方案，因为我们没有语法树。

### Representing values

We won't be *running* any code in this chapter, but since constants have a foot
in both the static and dynamic worlds of our interpreter, they force us to start
thinking at least a little bit about how our VM should represent values.

我们不会在本章中运行任何代码，但是由于常量在我们的解释器的静态和动态世界中都有立足点，
它们迫使我们至少开始思考一下我们的 VM 应该如何表示值。

For now, we're going to start as simple as possible -- we'll support only
double-precision, floating-point numbers. This will obviously expand over time,
so we'll set up a new module to give ourselves room to grow.

现在，我们将尽可能简单地开始——我们将仅支持双精度浮点数。这显然会随着时间的推移而扩展，
因此我们将设置一个新模块，为自己提供成长空间。

^code value-h

This typedef abstracts how Lox values are concretely represented in C. That way,
we can change that representation without needing to go back and fix existing
code that passes around values.

此 typedef 抽象了 Lox 值如何在 C 中具体表示。
这样，我们可以更改该表示，而无需返回并修复传递值的现有代码。

Back to the question of where to store constants in a chunk. For small
fixed-size values like integers, many instruction sets store the value directly
in the code stream right after the opcode. These are called **immediate
instructions** because the bits for the value are immediately after the opcode.

回到将常量存储在块中的位置的问题。对于像整数这样的固定大小的小值，
许多指令集将值直接存储在操作码之后的代码流中。这些被称为立即指令，因为值的位紧跟在操作码之后。

That doesn't work well for large or variable-sized constants like strings. In a
native compiler to machine code, those bigger constants get stored in a separate
"constant data" region in the binary executable. Then, the instruction to load a
constant has an address or offset pointing to where the value is stored in that
section.

这不适用于像字符串这样的大的或可变大小的常量。在机器代码的本机编译器中，
那些较大的常量存储在二进制可执行文件的单独“常量数据”区域中。
然后，加载常量的指令有一个地址或偏移量，指向该部分中存储值的位置。

Most virtual machines do something similar. For example, the Java Virtual
Machine [associates a **constant pool**][jvm const] with each compiled class.
That sounds good enough for clox to me. Each chunk will carry with it a list of
the values that appear as literals in the program. To keep things <span
name="immediate">simpler</span>, we'll put *all* constants in there, even simple
integers.

大多数虚拟机都做类似的事情。例如，Java 虚拟机 [关联一个常量池][jvm const] 与每个编译的类。
对我来说，这对 clox 来说已经足够好了。每个块都将携带一个值列表，这些值在程序中显示为文字。
为了让事情更简单，我们将把所有的常量都放在那里，甚至是简单的整数。

[jvm const]: https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.4

<aside name="immediate">

In addition to needing two kinds of constant instructions -- one for immediate
values and one for constants in the constant table -- immediates also force us
to worry about alignment, padding, and endianness. Some architectures aren't
happy if you try to say, stuff a 4-byte integer at an odd address.

除了需要两种常量指令——一种用于立即数，一种用于常量表中的常量——
立即数还迫使我们担心对齐、填充和字节序。如果您尝试说将 4 字节整数填充到奇数地址，
则某些体系结构会不满意。

</aside>

### Value arrays

The constant pool is an array of values. The instruction to load a constant
looks up the value by index in that array. As with our <span
name="generic">bytecode</span> array, the compiler doesn't know how big the
array needs to be ahead of time. So, again, we need a dynamic one. Since C
doesn't have generic data structures, we'll write another dynamic array data
structure, this time for Value.

常量池是一个值数组。加载常量的指令通过该数组中的索引查找值。与我们的bytecode数组一样，
编译器不知道数组需要提前多大。所以，再一次，我们需要一个动态的。
由于 C 没有通用数据结构，我们将编写另一个动态数组数据结构，这次是 Value。

<aside name="generic">

Defining a new struct and manipulation functions each time we need a dynamic
array of a different type is a chore. We could cobble together some preprocessor
macros to fake generics, but that's overkill for clox. We won't need many more
of these.

每次我们需要不同类型的动态数组时定义一个新的结构和操作函数是一件苦差事。
我们可以拼凑一些预处理器宏来伪造泛型，但这对 clox 来说太过分了。我们不需要更多这些。

</aside>

^code value-array (1 before, 2 after)

As with the bytecode array in Chunk, this struct wraps a pointer to an array
along with its allocated capacity and the number of elements in use. We also
need the same three functions to work with value arrays.

与 Chunk 中的字节码数组一样，该结构包含一个指向数组的指针及其分配的容量和正在使用的元素数量。
我们还需要相同的三个函数来处理值数组。

^code array-fns-h (1 before, 2 after)

The implementations will probably give you déjà vu. First, to create a new one:

这些实现可能会给您带来似曾相识的感觉。首先，创建一个新的：

^code value-c

Once we have an initialized array, we can start <span name="add">adding</span>
values to it.

一旦我们有一个初始化的数组，我们就可以开始向它添加值。

<aside name="add">

Fortunately, we don't need other operations like insertion and removal.

幸运的是，我们不需要像插入和移除这样的其他操作。

</aside>

^code write-value-array

The memory-management macros we wrote earlier do let us reuse some of the logic
from the code array, so this isn't too bad. Finally, to release all memory used
by the array:

我们之前编写的内存管理宏确实让我们重用了代码数组中的一些逻辑，所以这还不算太糟糕。
最后，释放数组使用的所有内存：

^code free-value-array

Now that we have growable arrays of values, we can add one to Chunk to store the
chunk's constants.

现在我们有了可增长的值数组，我们可以向 Chunk 添加一个来存储块的常量。

^code chunk-constants (1 before, 1 after)

Don't forget the include.

不要忘记包含。

^code chunk-h-include-value (1 before, 2 after)

Ah, C, and its Stone Age modularity story. Where were we? Right. When we
initialize a new chunk, we initialize its constant list too.

啊，C，以及它的石器时代模块化故事。我们刚刚说到哪了？对。
当我们初始化一个新块时，我们也会初始化它的常量列表。

^code chunk-init-constant-array (1 before, 1 after)

Likewise, we free the constants when we free the chunk.

同样，我们在释放块时释放常量。

^code chunk-free-constants (1 before, 1 after)

Next, we define a convenience method to add a new constant to the chunk. Our
yet-to-be-written compiler could write to the constant array inside Chunk
directly -- it's not like C has private fields or anything -- but it's a little
nicer to add an explicit function.

接下来，我们定义一个方便的方法来向块中添加一个新常量。
我们尚未编写的编译器可以直接写入 Chunk 中的常量数组——
它不像 C 有私有字段或任何东西——但添加显式函数会更好一些。

^code add-constant-h (1 before, 2 after)

Then we implement it.

然后我们实现它。

^code add-constant

After we add the constant, we return the index where the constant was appended
so that we can locate that same constant later.

添加常量后，我们返回添加常量的索引，以便我们稍后可以找到相同的常量。

### Constant instructions

We can *store* constants in chunks, but we also need to *execute* them. In a
piece of code like:

我们可以将常量存储在块中，但我们还需要执行它们。在一段代码中，如：

```lox
print 1;
print 2;
```

The compiled chunk needs to not only contain the values 1 and 2, but know *when*
to produce them so that they are printed in the right order. Thus, we need an
instruction that produces a particular constant.

编译后的块不仅需要包含值 1 和 2，还需要知道何时生成它们，以便以正确的顺序打印它们。
因此，我们需要一条产生特定常数的指令。

^code op-constant (1 before, 1 after)

When the VM executes a constant instruction, it <span name="load">"loads"</span>
the constant for use. This new instruction is a little more complex than
`OP_RETURN`. In the above example, we load two different constants. A single
bare opcode isn't enough to know *which* constant to load.

当VM执行一个常量指令时，它"加载"这个常量以供使用。
这个新指令比 `OP_RETURN` 稍微复杂一些。在上面的例子中，我们加载了两个不同的常量。
单个裸操作码不足以知道要加载哪个常量。

<aside name="load">

I'm being vague about what it means to "load" or "produce" a constant because we
haven't learned how the virtual machine actually executes code at runtime yet.
For that, you'll have to wait until you get to (or skip ahead to, I suppose) the
[next chapter][vm].

我对“加载”或“生成”常量的含义含糊不清，因为我们还没有了解虚拟机如何在运行时实际执行代码。
为此，您必须等到（或跳到，我想）[下一章][vm]。

[vm]: a-virtual-machine.html

</aside>

To handle cases like this, our bytecode -- like most others -- allows
instructions to have <span name="operand">**operands**</span>. These are stored
as binary data immediately after the opcode in the instruction stream and let us
parameterize what the instruction does.

为了处理这样的情况，我们的字节码——和大多数其他的一样——允许指令具有操作数。
这些在指令流中的操作码之后立即存储为二进制数据，让我们参数化指令的作用。

<img src="image/chunks-of-bytecode/format.png" alt="OP_CONSTANT is a byte for
the opcode followed by a byte for the constant index." />

Each opcode determines how many operand bytes it has and what they mean. For
example, a simple operation like "return" may have no operands, where an
instruction for "load local variable" needs an operand to identify which
variable to load. Each time we add a new opcode to clox, we specify what its
operands look like -- its **instruction format**.

每个操作码确定它有多少个操作数字节以及它们的含义。例如，像“return”这样的简单操作可能没有操作数，
其中“加载局部变量”的指令需要一个操作数来标识要加载的变量。
每次我们向 clox 添加一个新的操作码时，我们都会指定它的操作数是什么样的——它的指令格式。

<aside name="operand">

Bytecode instruction operands are *not* the same as the operands passed to an
arithmetic operator. You'll see when we get to expressions that arithmetic
operand values are tracked separately. Instruction operands are a lower-level
notion that modify how the bytecode instruction itself behaves.

字节码指令操作数与传递给算术运算符的操作数不同。当我们处理算术操作数值被单独跟踪的表达式时，
您会看到。指令操作数是一个较低级别的概念，它修改字节码指令本身的行为方式。

</aside>

In this case, `OP_CONSTANT` takes a single byte operand that specifies which
constant to load from the chunk's constant array. Since we don't have a compiler
yet, we "hand-compile" an instruction in our test chunk.

在这种情况下，`OP_CONSTANT` 采用一个单字节操作数，该操作数指定从块的常量数组中加载哪个常量。
由于我们还没有编译器，所以我们在测试块中“手动编译”了一条指令。

^code main-constant (1 before, 1 after)

We add the constant value itself to the chunk's constant pool. That returns the
index of the constant in the array. Then we write the constant instruction,
starting with its opcode. After that, we write the one-byte constant index
operand. Note that `writeChunk()` can write opcodes or operands. It's all raw
bytes as far as that function is concerned.

我们将常量值本身添加到块的常量池中。这将返回数组中常量的索引。
然后我们编写常量指令，从它的操作码开始。
之后，我们编写一字节常量索引操作数。请注意，`writeChunk()` 可以写入操作码或操作数。
就该功能而言，这都是原始字节。

If we try to run this now, the disassembler is going to yell at us because it
doesn't know how to decode the new instruction. Let's fix that.

如果我们现在尝试运行它，反汇编程序会冲我们大喊大叫，因为它不知道如何解码新指令。让我们解决这个问题。

^code disassemble-constant (1 before, 1 after)

This instruction has a different instruction format, so we write a new helper
function to disassemble it.

该指令具有不同的指令格式，因此我们编写了一个新的辅助函数来反汇编它。

^code constant-instruction

There's more going on here. As with `OP_RETURN`, we print out the name of the
opcode. Then we pull out the constant index from the subsequent byte in the
chunk. We print that index, but that isn't super useful to us human readers. So
we also look up the actual constant value -- since constants *are* known at
compile time after all -- and display the value itself too.

这里还有更多事情要做。与 `OP_RETURN` 一样，我们打印出操作码的名称。
然后我们从块中的后续字节中提取常量索引。
我们打印该索引，但这对我们人类读者来说并不是超级有用。
所以我们还要查找实际的常量值——毕竟常量在编译时是已知的——并且也显示值本身。

This requires some way to print a clox Value. That function will live in the
"value" module, so we include that.

这需要某种方式来打印 clox 值。该函数将存在于“值”模块中，因此我们将其包含在内。

^code debug-include-value (1 before, 2 after)

Over in that header, we declare:

在该头文件中，我们声明：

^code print-value-h (1 before, 2 after)

And here's an implementation:

这是一个实现：

^code print-value

Magnificent, right? As you can imagine, this is going to get more complex once
we add dynamic typing to Lox and have values of different types.

壮观，对吧？可以想象，一旦我们将动态类型添加到 Lox 并具有不同类型的值，这将变得更加复杂。

Back in `constantInstruction()`, the only remaining piece is the return value.

回到`constantInstruction()`，唯一剩下的就是返回值。

^code return-after-operand (1 before, 1 after)

Remember that `disassembleInstruction()` also returns a number to tell the
caller the offset of the beginning of the *next* instruction. Where `OP_RETURN`
was only a single byte, `OP_CONSTANT` is two -- one for the opcode and one for
the operand.

请记住，`disassembleInstruction()` 还返回一个数字来告诉调用者下一条指令开始的偏移量。 
`OP_RETURN` 只是一个字节，`OP_CONSTANT` 是两个——一个用于操作码，一个用于操作数。

## Line Information

Chunks contain almost all of the information that the runtime needs from the
user's source code. It's kind of crazy to think that we can reduce all of the
different AST classes that we created in jlox down to an array of bytes and an
array of constants. There's only one piece of data we're missing. We need it,
even though the user hopes to never see it.

块包含运行时需要的用户源代码中的几乎所有信息。
认为我们可以将我们在 jlox 中创建的所有不同的 AST 类缩减为一个字节数组和一个常量数组，这有点疯狂。
我们只丢失了一条数据。我们需要它，即使用户希望永远看不到它。

When a runtime error occurs, we show the user the line number of the offending
source code. In jlox, those numbers live in tokens, which we in turn store in
the AST nodes. We need a different solution for clox now that we've ditched
syntax trees in favor of bytecode. Given any bytecode instruction, we need to be
able to determine the line of the user's source program that it was compiled
from.

当发生运行时错误时，我们会向用户显示有问题的源代码的行号。
在 jlox 中，这些数字存在于令牌中，我们又将其存储在 AST 节点中。
既然我们已经抛弃了语法树，转而使用字节码，我们就需要一个不同的 clox 解决方案。
给定任何字节码指令，我们需要能够确定编译它的用户源程序的行。

There are a lot of clever ways we could encode this. I took the absolute <span
name="side">simplest</span> approach I could come up with, even though it's
embarrassingly inefficient with memory. In the chunk, we store a separate array
of integers that parallels the bytecode. Each number in the array is the line
number for the corresponding byte in the bytecode. When a runtime error occurs,
we look up the line number at the same index as the current instruction's offset
in the code array.

有很多聪明的方法可以对它进行编码。我采用了我能想到的绝对最简单的方法，
尽管它在内存方面的效率低得令人尴尬。在块中，我们存储了一个与字节码平行的单独整数数组。
数组中的每个数字都是字节码中相应字节的行号。
当发生运行时错误时，我们在代码数组中查找与当前指令的偏移量相同的索引处的行号。

<aside name="side">

This braindead encoding does do one thing right: it keeps the line information
in a *separate* array instead of interleaving it in the bytecode itself. Since
line information is only used when a runtime error occurs, we don't want it
between the instructions, taking up precious space in the CPU cache and causing
more cache misses as the interpreter skips past it to get to the opcodes and
operands it cares about.

这种无脑编码确实做了一件正确的事情：它将行信息保存在一个单独的数组中，而不是将其交错在字节码本身中。
由于行信息仅在发生运行时错误时使用，我们不希望在指令之间使用它，
占用 CPU 缓存中的宝贵空间并导致更多缓存未命中，因为解释器跳过它以获取它关心的操作码和操作数关于。

</aside>

To implement this, we add another array to Chunk.

为了实现这一点，我们向 Chunk 添加了另一个数组。

^code chunk-lines (1 before, 1 after)

Since it exactly parallels the bytecode array, we don't need a separate count or
capacity. Every time we touch the code array, we make a corresponding change to
the line number array, starting with initialization.

由于它与字节码数组完全平行，因此我们不需要单独的计数或容量。每次触摸代码数组，
我们都会对行号数组进行相应的更改，从初始化开始。

^code chunk-null-lines (1 before, 1 after)

And likewise deallocation:

同样释放：

^code chunk-free-lines (1 before, 1 after)

When we write a byte of code to the chunk, we need to know what source line it
came from, so we add an extra parameter in the declaration of `writeChunk()`.

当我们将一个字节的代码写入块时，我们需要知道它来自哪个源代码行，
因此我们在 `writeChunk()` 的声明中添加了一个额外的参数。

^code write-chunk-with-line-h (1 before, 1 after)

And in the implementation:

在实现中：

^code write-chunk-with-line (1 after)

When we allocate or grow the code array, we do the same for the line info too.

当我们分配或增长代码数组时，我们也对行信息做同样的事情。

^code write-chunk-line (2 before, 1 after)

Finally, we store the line number in the array.

最后，我们将行号存储在数组中。

^code chunk-write-line (1 before, 1 after)

### Disassembling line information

Alright, let's try this out with our little, uh, artisanal chunk. First, since
we added a new parameter to `writeChunk()`, we need to fix those calls to pass
in some -- arbitrary at this point -- line number.

好吧，让我们用我们的小，呃，手工块来试试这个。
首先，由于我们向 `writeChunk()` 添加了一个新参数，
我们需要修复这些调用以传入一些——此时是任意的——行号。

^code main-chunk-line (1 before, 2 after)

Once we have a real front end, of course, the compiler will track the current
line as it parses and pass that in.

当然，一旦我们有了真正的前端，编译器就会在解析时跟踪当前行并将其传入。

Now that we have line information for every instruction, let's put it to good
use. In our disassembler, it's helpful to show which source line each
instruction was compiled from. That gives us a way to map back to the original
code when we're trying to figure out what some blob of bytecode is supposed to
do. After printing the offset of the instruction -- the number of bytes from the
beginning of the chunk -- we show its source line.

现在我们有了每条指令的行信息，让我们好好利用它。在我们的反汇编器中，
显示每条指令是从哪个源代码行编译的很有帮助。
当我们试图弄清楚一些字节码应该做什么时，这为我们提供了一种映射回原始代码的方法。
在打印指令的偏移量之后——从块开始的字节数——我们显示它的源代码行。

^code show-location (2 before, 2 after)

Bytecode instructions tend to be pretty fine-grained. A single line of source
code often compiles to a whole sequence of instructions. To make that more
visually clear, we show a `|` for any instruction that comes from the same
source line as the preceding one. The resulting output for our handwritten
chunk looks like:

字节码指令往往非常细粒度。单行源代码通常编译成整个指令序列。
为了使这一点在视觉上更清晰，我们为来自与前一条相同源代码行的任何指令显示一个`|`。
我们的手写块的结果输出如下所示：

```text
== test chunk ==
0000  123 OP_CONSTANT         0 '1.2'
0002    | OP_RETURN
```

We have a three-byte chunk. The first two bytes are a constant instruction that
loads 1.2 from the chunk's constant pool. The first byte is the `OP_CONSTANT`
opcode and the second is the index in the constant pool. The third byte (at
offset 2) is a single-byte return instruction.

我们有一个三字节的块。前两个字节是从块的常量池中加载 1.2 的常量指令。
第一个字节是“OP_CONSTANT”操作码，第二个是常量池中的索引。
第三个字节（偏移量 2）是单字节返回指令。

In the remaining chapters, we will flesh this out with lots more kinds of
instructions. But the basic structure is here, and we have everything we need
now to completely represent an executable piece of code at runtime in our
virtual machine. Remember that whole family of AST classes we defined in jlox?
In clox, we've reduced that down to three arrays: bytes of code, constant
values, and line information for debugging.

在剩下的章节中，我们将用更多种类的指令来充实这一点。但是基本结构就在这里，
我们现在拥有了在我们的虚拟机中在运行时完全表示一段可执行代码所需的一切。
还记得我们在 jlox 中定义的整个 AST 类家族吗？在 clox 中，我们将其缩减为三个数组：
代码字节、常量值和用于调试的行信息。

This reduction is a key reason why our new interpreter will be faster than jlox.
You can think of bytecode as a sort of compact serialization of the AST, highly
optimized for how the interpreter will deserialize it in the order it needs as
it executes. In the [next chapter][vm], we will see how the virtual machine does
exactly that.

这种减少是我们的新解释器比 jlox 更快的一个关键原因。
您可以将字节码视为 AST 的一种紧凑序列化，高度优化了解释器在执行时将如何按需要的顺序反序列化它。
在 [下一章][vm] 中，我们将看到虚拟机是如何做到这一点的。

<div class="challenges">

## Challenges

1.  Our encoding of line information is hilariously wasteful of memory. Given
    that a series of instructions often correspond to the same source line, a
    natural solution is something akin to [run-length encoding][rle] of the line
    numbers.

    Devise an encoding that compresses the line information for a
    series of instructions on the same line. Change `writeChunk()` to write this
    compressed form, and implement a `getLine()` function that, given the index
    of an instruction, determines the line where the instruction occurs.

    *Hint: It's not necessary for `getLine()` to be particularly efficient.
    Since it is called only when a runtime error occurs, it is well off the
    critical path where performance matters.*

2.  Because `OP_CONSTANT` uses only a single byte for its operand, a chunk may
    only contain up to 256 different constants. That's small enough that people
    writing real-world code will hit that limit. We could use two or more bytes
    to store the operand, but that makes *every* constant instruction take up
    more space. Most chunks won't need that many unique constants, so that
    wastes space and sacrifices some locality in the common case to support the
    rare case.

    To balance those two competing aims, many instruction sets feature multiple
    instructions that perform the same operation but with operands of different
    sizes. Leave our existing one-byte `OP_CONSTANT` instruction alone, and
    define a second `OP_CONSTANT_LONG` instruction. It stores the operand as a
    24-bit number, which should be plenty.

    Implement this function:

    ```c
    void writeConstant(Chunk* chunk, Value value, int line) {
      // Implement me...
    }
    ```

    It adds `value` to `chunk`'s constant array and then writes an appropriate
    instruction to load the constant. Also add support to the disassembler for
    `OP_CONSTANT_LONG` instructions.

    Defining two instructions seems to be the best of both worlds. What
    sacrifices, if any, does it force on us?

3.  Our `reallocate()` function relies on the C standard library for dynamic
    memory allocation and freeing. `malloc()` and `free()` aren't magic. Find
    a couple of open source implementations of them and explain how they work.
    How do they keep track of which bytes are allocated and which are free?
    What is required to allocate a block of memory? Free it? How do they make
    that efficient? What do they do about fragmentation?

    *Hardcore mode:* Implement `reallocate()` without calling `realloc()`,
    `malloc()`, or `free()`. You are allowed to call `malloc()` *once*, at the
    beginning of the interpreter's execution, to allocate a single big block of
    memory, which your `reallocate()` function has access to. It parcels out
    blobs of memory from that single region, your own personal heap. It's your
    job to define how it does that.

</div>

[rle]: https://en.wikipedia.org/wiki/Run-length_encoding

<div class="design-note">

## Design Note: Test Your Language

We're almost halfway through the book and one thing we haven't talked about is
*testing* your language implementation. That's not because testing isn't
important. I can't possibly stress enough how vital it is to have a good,
comprehensive test suite for your language.

I wrote a [test suite for Lox][tests] (which you are welcome to use on your own
Lox implementation) before I wrote a single word of this book. Those tests found
countless bugs in my implementations.

[tests]: https://github.com/munificent/craftinginterpreters/tree/master/test

Tests are important in all software, but they're even more important for a
programming language for at least a couple of reasons:

*   **Users expect their programming languages to be rock solid.** We are so
    used to mature, stable compilers and interpreters that "It's your code, not
    the compiler" is [an ingrained part of software culture][fault]. If there
    are bugs in your language implementation, users will go through the full
    five stages of grief before they can figure out what's going on, and you
    don't want to put them through all that.

*   **A language implementation is a deeply interconnected piece of software.**
    Some codebases are broad and shallow. If the file loading code is broken in
    your text editor, it -- hopefully! -- won't cause failures in the text
    rendering on screen. Language implementations are narrower and deeper,
    especially the core of the interpreter that handles the language's actual
    semantics. That makes it easy for subtle bugs to creep in caused by weird
    interactions between various parts of the system. It takes good tests to
    flush those out.

*   **The input to a language implementation is, by design, combinatorial.**
    There are an infinite number of possible programs a user could write, and
    your implementation needs to run them all correctly. You obviously can't
    test that exhaustively, but you need to work hard to cover as much of the
    input space as you can.

*   **Language implementations are often complex, constantly changing, and full
    of optimizations.** That leads to gnarly code with lots of dark corners
    where bugs can hide.

[fault]: https://blog.codinghorror.com/the-first-rule-of-programming-its-always-your-fault/

All of that means you're gonna want a lot of tests. But *what* tests? Projects
I've seen focus mostly on end-to-end "language tests". Each test is a program
written in the language along with the output or errors it is expected to
produce. Then you have a test runner that pushes the test program through your
language implementation and validates that it does what it's supposed to.
Writing your tests in the language itself has a few nice advantages:

*   The tests aren't coupled to any particular API or internal architecture
    decisions of the implementation. This frees you to reorganize or rewrite
    parts of your interpreter or compiler without needing to update a slew of
    tests.

*   You can use the same tests for multiple implementations of the language.

*   Tests can often be terse and easy to read and maintain since they are
    simply scripts in your language.

It's not all rosy, though:

*   End-to-end tests help you determine *if* there is a bug, but not *where* the
    bug is. It can be harder to figure out where the erroneous code in the
    implementation is because all the test tells you is that the right output
    didn't appear.

*   It can be a chore to craft a valid program that tickles some obscure corner
    of the implementation. This is particularly true for highly optimized
    compilers where you may need to write convoluted code to ensure that you
    end up on just the right optimization path where a bug may be hiding.

*   The overhead can be high to fire up the interpreter, parse, compile, and
    run each test script. With a big suite of tests -- which you *do* want,
    remember -- that can mean a lot of time spent waiting for the tests to
    finish running.

I could go on, but I don't want this to turn into a sermon. Also, I don't
pretend to be an expert on *how* to test languages. I just want you to
internalize how important it is *that* you test yours. Seriously. Test your
language. You'll thank me for it.

</div>
