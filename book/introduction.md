> Fairy tales are more than true: not because they tell us that dragons exist,
> but because they tell us that dragons can be beaten.
>
> <cite>G.K. Chesterton by way of Neil Gaiman, <em>Coraline</em></cite>

I'm really excited we're going on this journey together. This is a book on
implementing interpreters for programming languages. It's also a book on how to
design a language worth implementing. It's the book I wish I'd had when I first
started getting into languages, and it's the book I've been writing in my <span
name="head">head</span> for nearly a decade.

_我真的很兴奋我们将一起踏上这段旅程。这是一本关于为编程语言实现解释器的书。
这也是一本关于如何设计一种值得实现的语言的书。这是我刚开始涉足语言领域时
希望拥有的书，也是我近十年来一直在脑海中写的书。_

<aside name="head">

To my friends and family, sorry I've been so absentminded!

_致我的朋友和家人，对不起，我一直心不在焉。_

</aside>

In these pages, we will walk step-by-step through two complete interpreters for
a full-featured language. I assume this is your first foray into languages, so
I'll cover each concept and line of code you need to build a complete, usable,
fast language implementation.

_在这些页面中，我们将逐步介绍两个完整的解释器，以获得功能齐全的语言。
我假设这是你第一次涉足语言领域，因此我将介绍构建完整、可用、快速的
语言实现所需的每个概念和代码行。_

In order to cram two full implementations inside one book without it turning
into a doorstop, this text is lighter on theory than others. As we build each
piece of the system, I will introduce the history and concepts behind it. I'll
try to get you familiar with the lingo so that if you ever find yourself at a
<span name="party">cocktail party</span> full of PL (programming language)
researchers, you'll fit in.

_为了在一本书里塞进两个完整的解释器实现，而不把它变成“门挡”，这本书相比其他书来说没那么注重理论。
在我们构建系统的每一部分时，我将介绍它背后的历史和概念。我会尽量让你熟悉这些术语，
这样能让你适应去参加一个充满 PL（编程语言）研究人员的“鸡尾酒派对”。_

<aside name="party">

Strangely enough, a situation I have found myself in multiple times. You
wouldn't believe how much some of them can drink.

_奇怪的是，我曾多次发现自己身处其中。你不会相信他们中的一些人可以喝多少。_

</aside>

But we're mostly going to spend our brain juice getting the language up and
running. This is not to say theory isn't important. Being able to reason
precisely and <span name="formal">formally</span> about syntax and semantics is
a vital skill when working on a language. But, personally, I learn best by
doing. It's hard for me to wade through paragraphs full of abstract concepts and
really absorb them. But if I've coded something, run it, and debugged it, then I
*get* it.

_但我们主要会花费我们的脑汁来模拟启动和运行语言。这并不是说理论不重要。
能够正儿八经地推理语法和语义是学习一门语言时的一项重要技能。
但是，就个人而言，我通过实践学习得最好。我很难通过充满抽象概念的段落并真正吸收它们。
但是，如果我编写了一些代码、运行它并调试了它，那么我就明白了。_

<aside name="formal">

Static type systems in particular require rigorous formal reasoning. Hacking on
a type system has the same feel as proving a theorem in mathematics.

_静态类型系统尤其需要严格的形式推理。破解类型系统与证明数学定理具有相同的感觉。_

It turns out this is no coincidence. In the early half of last century, Haskell
Curry and William Alvin Howard showed that they are two sides of the same coin:
[the Curry-Howard isomorphism][].

[the curry-howard isomorphism]: https://en.wikipedia.org/wiki/Curry%E2%80%93Howard_correspondence

_事实证明，这并非巧合。上个世纪上半叶，Haskell Curry 和 William Alvin Howard证明了他们是一枚硬币正反面：
[the Curry-Howard isomorphism][]._

[the curry-howard isomorphism]: https://en.wikipedia.org/wiki/Curry%E2%80%93Howard_correspondence

</aside>

That's my goal for you. I want you to come away with a solid intuition of how a
real language lives and breathes. My hope is that when you read other, more
theoretical books later, the concepts there will firmly stick in your mind,
adhered to this tangible substrate.

_这就是我给你的目标。我希望你能对真正的语言如何生活和呼吸有一个坚实的直觉。
我希望当你以后阅读其他更多理论书籍时，那里的概念会牢牢地留在你的脑海中，坚持这个有形的基础。_

## Why Learn This Stuff?

Every introduction to every compiler book seems to have this section. I don't
know what it is about programming languages that causes such existential doubt.
I don't think ornithology books worry about justifying their existence. They
assume the reader loves birds and start teaching.

_每本编译器书籍的介绍似乎都有这一节。我不知道是什么原因导致了关于编程语言的这种存在性怀疑。
我不认为鸟类学书籍担心证明它们的存在。他们假先假设读者喜欢鸟类然后开始教学。_

But programming languages are a little different. I suppose it is true that the
odds of any of us creating a broadly successful, general-purpose programming
language are slim. The designers of the world's widely used languages could fit
in a Volkswagen bus, even without putting the pop-top camper up. If joining that
elite group was the *only* reason to learn languages, it would be hard to
justify. Fortunately, it isn't.

_但是编程语言有点不同。我想我们中的任何人创建一种广泛成功的通用编程语言的可能性确实很小。
一辆大众巴士就能装进世界上所有广泛使用的语言的设计者，即使没有把弹出式露营车放在上面。
如果加入那个精英群体是学习语言的唯一原因，那将很难被证明是合理的。 幸运的是，事实并非如此。_

### Little languages are everywhere

For every successful general-purpose language, there are a thousand successful
niche ones. We used to call them "little languages", but inflation in the jargon
economy led to the name "domain-specific languages". These are pidgins
tailor-built to a specific task. Think application scripting languages, template
engines, markup formats, and configuration files.

_对于每一种成功的通用语言，都能对应一千种成功的小众语言。我们过去称它们为“小语言”，
但行话经济的膨胀导致了“特定领域语言”的名称。这些是为特定任务量身定制的 pidgins。
想想应用程序脚本语言、模板引擎、标记格式和配置文件。_

<span name="little"></span><img src="image/introduction/little-languages.png" alt="A random selection of little languages." />

<aside name="little">

A random selection of some little languages you might run into.

_这里随机选择了一些你可能遇到的小语言。_

</aside>

Almost every large software project needs a handful of these. When you can, it's
good to reuse an existing one instead of rolling your own. Once you factor in
documentation, debuggers, editor support, syntax highlighting, and all of the
other trappings, doing it yourself becomes a tall order.

_几乎每个大型软件项目都需要其中的一小部分。如果可以，最好重用现有的而不是用自己的。
一旦你考虑到文档、调试器、编辑器支持、语法高亮和所有其他陷阱，你自己做这件事就变得很艰巨了。_

But there's still a good chance you'll find yourself needing to whip up a parser
or other tool when there isn't an existing library that fits your needs. Even
when you are reusing some existing implementation, you'll inevitably end up
needing to debug and maintain it and poke around in its guts.

_但是，当没有一个现有的库可以满足你的需要时，你仍然有很大的可能发现自己需要开发一个分析器或其他工具。
即使你在重复使用一些现有的实现，你也不可避免地需要对它进行调试和维护，并在它的内部进行探究。_

### Languages are great exercise

Long distance runners sometimes train with weights strapped to their ankles or
at high altitudes where the atmosphere is thin. When they later unburden
themselves, the new relative ease of light limbs and oxygen-rich air enables
them to run farther and faster.

_长跑运动员有时会将重物绑在脚踝上或在空气稀薄的高海拔地区进行训练。
当他们后来卸下自己的负担时，轻巧的四肢和富氧的空气使他们能够跑得更远更快。_

Implementing a language is a real test of programming skill. The code is complex
and performance critical. You must master recursion, dynamic arrays, trees,
graphs, and hash tables. You probably use hash tables at least in your
day-to-day programming, but do you *really* understand them? Well, after we've
crafted our own from scratch, I guarantee you will.

_实现一种语言是对编程技能的真正考验。代码复杂且性能至关重要。
你必须掌握递归、动态数组、树、图形和哈希表。
你可能至少在日常编程中使用哈希表，但你真的了解它们吗？
好吧，在我们从头开始制作自己的作品之后，我保证你会的。_

While I intend to show you that an interpreter isn't as daunting as you might
believe, implementing one well is still a challenge. Rise to it, and you'll come
away a stronger programmer, and smarter about how you use data structures and
algorithms in your day job.

_虽然我打算向你展示解释器并不像你想象的那样令人生畏，但实现它仍然是一个挑战。
坚持下去，你会成为一个更强大的程序员，并且你在今后的日常工作中能更好地使用数据结构和算法。_

### One more reason

This last reason is hard for me to admit, because it's so close to my heart.
Ever since I learned to program as a kid, I felt there was something magical
about languages. When I first tapped out BASIC programs one key at a time I
couldn't conceive how BASIC *itself* was made.

_最后一个原因我很难承认，因为它太贴近我的内心了。
自从我小时候学会编程以来，我就觉得语言有一些神奇的东西。
当我第一次敲击 BASIC 程序时，我无法想象 BASIC 本身是如何制作的。_

Later, the mixture of awe and terror on my college friends' faces when talking
about their compilers class was enough to convince me language hackers were a
different breed of human -- some sort of wizards granted privileged access to
arcane arts.

_后来，当我的大学朋友谈论他们的编译器课程时，他们脸上混合着敬畏和恐惧，
这足以让我相信语言黑客是一个不同的人类——某种被授予特权因而可以接触奥秘艺术的巫师。_

It's a charming <span name="image">image</span>, but it has a darker side. *I*
didn't feel like a wizard, so I was left thinking I lacked some inborn quality
necessary to join the cabal. Though I've been fascinated by languages ever since
I doodled made-up keywords in my school notebook, it took me decades to muster
the courage to try to really learn them. That "magical" quality, that sense of
exclusivity, excluded *me*.

_这是一个迷人的形象，但它有一个黑暗的一面。
我不觉得自己是个巫师，所以我就觉得自己缺乏一些加入阴谋集团所必需的先天素质。
尽管自从我在学校的笔记本上涂抹了一些捏造的关键词后，我就对语言很着迷，
但我花了几十年的时间才鼓起勇气去尝试真正学习它们。
那种 "神奇 "的品质，那种排他性的感觉，把我排除在外。_

<aside name="image">

And its practitioners don't hesitate to play up this image. Two of the seminal
texts on programming languages feature a [dragon][] and a [wizard][] on their
covers.

[dragon]: https://en.wikipedia.org/wiki/Compilers:_Principles,_Techniques,_and_Tools
[wizard]: https://mitpress.mit.edu/sites/default/files/sicp/index.html

_而其从业者也毫不犹豫地渲染这一形象。两本关于编程语言的开创性书籍封面上都有一条[龙][]和一个[巫师][]。_

[龙]: https://en.wikipedia.org/wiki/Compilers:_Principles,_Techniques,_and_Tools
[巫师]: https://mitpress.mit.edu/sites/default/files/sicp/index.html

</aside>

When I did finally start cobbling together my own little interpreters, I quickly
learned that, of course, there is no magic at all. It's just code, and the
people who hack on languages are just people.

_当我终于开始拼凑我自己的小解释器时，我很快就知道，根本就没有什么魔法。
它只是代码，而编写语言的也只是人。_

There *are* a few techniques you don't often encounter outside of languages, and
some parts are a little difficult. But not more difficult than other obstacles
you've overcome. My hope is that if you've felt intimidated by languages and
this book helps you overcome that fear, maybe I'll leave you just a tiny bit
braver than you were before.

_有一些技术是你在语言之外不常遇到的，而且有些部分是有点困难的。但不会比其他你已经克服的障碍更难。
我的希望是，如果你对语言感到恐惧，而这本书能帮助你克服这种恐惧，也许我会让你比以前更勇敢一丁点。_

And, who knows, maybe you *will* make the next great language. Someone has to.

_而且，谁知道呢，也许你会创造出下一个伟大的语言。总得有人去做。_

## How the Book Is Organized

This book is broken into three parts. You're reading the first one now. It's a
couple of chapters to get you oriented, teach you some of the lingo that
language hackers use, and introduce you to Lox, the language we'll be
implementing.

_这本书分为三部分。你现在读的是第一部分。它是几章节，为了让你掌握方向，教你一些语言黑客使用的术语，
并向你介绍我们将要实现的语言Lox。_

Each of the other two parts builds one complete Lox interpreter. Within those
parts, each chapter is structured the same way. The chapter takes a single
language feature, teaches you the concepts behind it, and walks you through an
implementation.

_其他两部分中的每一部分都构建了一个完整的Lox解释器。在两个部分中，每一章的结构都是一样的。
每一章以一个单一的语言特性，告诉你其背后的概念，并引导你完成一个自己的实现。_

It took a good bit of trial and error on my part, but I managed to carve up the
two interpreters into chapter-sized chunks that build on the previous chapters
but require nothing from later ones. From the very first chapter, you'll have a
working program you can run and play with. With each passing chapter, it grows
increasingly full-featured until you eventually have a complete language.

_我花了很多时间试错，但我设法将两个解释器分成章节大小的块，这些块建立在前面的章节之上，
但不需要后面的章节。从第一章开始，你将拥有一个可以运行和使用的工作程序。
随着每一章的流逝，它的功能越来越丰富，直到你最终拥有一门完整的语言。_

Aside from copious, scintillating English prose, chapters have a few other
delightful facets:

_除了丰富、精彩纷呈的英文散文外，各章节还有一些其他令人愉快的方面：_

### The code

We're about *crafting* interpreters, so this book contains real code. Every
single line of code needed is included, and each snippet tells you where to
insert it in your ever-growing implementation.

_我们是关于制作解释器的，所以这本书包含真实的代码。
所需的每一行代码都包括在内，每个片段都会告诉你在不断增长的实现中将其插入到何处。_

Many other language books and language implementations use tools like [Lex][]
and <span name="yacc">[Yacc][]</span>, so-called **compiler-compilers**, that
automatically generate some of the source files for an implementation from some
higher-level description. There are pros and cons to tools like those, and
strong opinions -- some might say religious convictions -- on both sides.

_许多其他语言书籍和语言实现使用诸如 [Lex][] 和 [Yacc][] 之类的工具，即所谓的编译器-编译器，
它们会根据某些更高级别的描述自动为实现生成一些源文件。
像这样的工具有利有弊，而且双方都有强烈的意见——有些人可能会说是宗教信仰。_

<aside name="yacc">

Yacc is a tool that takes in a grammar file and produces a source file for a
compiler, so it's sort of like a "compiler" that outputs a compiler, which is
where we get the term "compiler-compiler".

_Yacc 是一个接受语法文件并为编译器生成源文件的工具，因此它有点像输出编译器的“编译器”，
这就是我们得到术语“编译器-编译器”的地方。_

Yacc wasn't the first of its ilk, which is why it's named "Yacc" -- *Yet
Another* Compiler-Compiler. A later similar tool is [Bison][], named as a pun on
the pronunciation of Yacc like "yak".

_Yacc 并不是同类产品中的第一个，这就是为什么它被命名为“Yacc”——Yet Another Compiler-Compiler。
后来类似的工具是 [Bison][]，命名为 Yacc 发音的双关语，如“yak”。_

<img src="image/introduction/yak.png" alt="A yak." />

[bison]: https://en.wikipedia.org/wiki/GNU_bison

If you find all of these little self-references and puns charming and fun,
you'll fit right in here. If not, well, maybe the language nerd sense of humor
is an acquired taste.

_如果你发现所有这些小小的自我引用和双关语既迷人又有趣，那么你将适合这里。
如果不是，那么也许语言书呆子的幽默感是一种后天习得的品味。_

</aside>

We will abstain from using them here. I want to ensure there are no dark corners
where magic and confusion can hide, so we'll write everything by hand. As you'll
see, it's not as bad as it sounds, and it means you really will understand each
line of code and how both interpreters work.

_我们将避免在这里使用它们。我想确保没有可以隐藏魔法和混乱的黑暗角落，所以我们将手写所有内容。
正如你将看到的，它并不像听起来那么糟糕，这意味着你将真正理解每一行代码以及两个解释器的工作方式。_

[lex]: https://en.wikipedia.org/wiki/Lex_(software)
[yacc]: https://en.wikipedia.org/wiki/Yacc

A book has different constraints from the "real world" and so the coding style
here might not always reflect the best way to write maintainable production
software. If I seem a little cavalier about, say, omitting `private` or
declaring a global variable, understand I do so to keep the code easier on your
eyes. The pages here aren't as wide as your IDE and every character counts.

_一本书与“现实世界”有不同的限制，因此这里的编码风格可能并不总是反映编写可维护生产软件的最佳方式。
如果我在省略 `private` 或声明全局变量方面显得有点傲慢，请理解我这样做是为了让代码更容易被你看到。
这里的页面没有你的IDE那么宽，每个字符都很重要。_

Also, the code doesn't have many comments. That's because each handful of lines
is surrounded by several paragraphs of honest-to-God prose explaining it. When
you write a book to accompany your program, you are welcome to omit comments
too. Otherwise, you should probably use `//` a little more than I do.

_此外，代码没有太多注释。那是因为每一句台词都被几段诚实的散文所包围。
当你写一本书来伴随你的程序时，你也可以省略评论。否则，你可能应该比我多使用一点`//` 。_

While the book contains every line of code and teaches what each means, it does
not describe the machinery needed to compile and run the interpreter. I assume
you can slap together a makefile or a project in your IDE of choice in order to
get the code to run. Those kinds of instructions get out of date quickly, and
I want this book to age like XO brandy, not backyard hooch.

_虽然这本书包含每一行代码并教授每行代码的含义，但它没有描述编译和运行解释器所需的机制。
我假设你可以在你选择的 IDE 中将一个 makefile 或一个项目放在一起，以便让代码运行。
因为像这样的说明很快就会过时，我希望这本书能像 XO 白兰地一样陈年，而不是后院的酒。_

### Snippets

Since the book contains literally every line of code needed for the
implementations, the snippets are quite precise. Also, because I try to keep the
program in a runnable state even when major features are missing, sometimes we
add temporary code that gets replaced in later snippets.

_由于这本书实际上包含了实现所需的每一行代码，因此这些片段非常精确。
另外，因为即使缺少主要功能，我也会尝试使程序保持可运行状态，
因此有时我们会添加临时代码，这些代码会在以后的代码片段中被替换。_

A snippet with all the bells and whistles looks like this:

_一个包含所有花里胡哨的片段如下所示：_

<div class="codehilite"><pre class="insert-before">
      default:
</pre><div class="source-file"><em>lox/Scanner.java</em><br>
in <em>scanToken</em>()<br>
replace 1 line</div>
<pre class="insert">
        <span class="k">if</span> (<span class="i">isDigit</span>(<span class="i">c</span>)) {
          <span class="i">number</span>();
        } <span class="k">else</span> {
          <span class="t">Lox</span>.<span class="i">error</span>(<span class="i">line</span>, <span class="s">&quot;Unexpected character.&quot;</span>);
        }
</pre><pre class="insert-after">
        break;
</pre></div>
<div class="source-file-narrow"><em>lox/Scanner.java</em>, in <em>scanToken</em>(), replace 1 line</div>

In the center, you have the new code to add. It may have a few faded out lines
above or below to show where it goes in the existing surrounding code. There is
also a little blurb telling you in which file and where to place the snippet. If
that blurb says "replace _ lines", there is some existing code between the faded
lines that you need to remove and replace with the new snippet.

_你可以在中间添加新代码。它可能在上方或下方有一些淡出的行，以显示它在现有周围代码中的位置。
还有一个小简介告诉你在哪个文件中以及在哪里放置片段。
如果该简介显示"replace _ lines"，则在褪色的行之间存在一些现有代码，
你需要将其删除并替换为新的代码段。_

### Asides

<span name="joke">Asides</span> contain biographical sketches, historical
background, references to related topics, and suggestions of other areas to
explore. There's nothing that you *need* to know in them to understand later
parts of the book, so you can skip them if you want. I won't judge you, but I
might be a little sad.

_旁白包含传记草图、历史背景、对相关主题的引用以及其他领域的探索建议。
为了理解本书后面的部分，你不需要知道任何东西，所以如果你愿意，你可以跳过它们。
我不会评判你，但我可能会有点难过。_

<aside name="joke">

Well, some asides do, at least. Most of them are just dumb jokes and amateurish
drawings.

_好吧，至少有些旁白确实如此。他们中的大多数只是愚蠢的笑话和业余的图画。_

</aside>

### Challenges

Each chapter ends with a few exercises. Unlike textbook problem sets, which tend
to review material you already covered, these are to help you learn *more* than
what's in the chapter. They force you to step off the guided path and explore on
your own. They will make you research other languages, figure out how to
implement features, or otherwise get you out of your comfort zone.

_每章以一些练习结束。与教科书习题集不同的是，这些习题集往往会复习你已经涵盖的材料，
这些习题集旨在帮助你学习比本章内容更多的内容。他们迫使你离开引导路径并自行探索。
它们会让你研究其他语言，弄清楚如何实现功能，或者让你走出舒适区。_

<span name="warning">Vanquish</span> the challenges and you'll come away with a
broader understanding and possibly a few bumps and scrapes. Or skip them if you
want to stay inside the comfy confines of the tour bus. It's your book.

_战胜挑战，你会获得更广泛的理解，并且可能会遇到一些磕磕碰碰。
如果你想留在舒适的旅游巴士范围内，也可以跳过它们。这是你的书。_

<aside name="warning">

A word of warning: the challenges often ask you to make changes to the
interpreter you're building. You'll want to implement those in a copy of your
code. The later chapters assume your interpreter is in a pristine
("unchallenged"?) state.

_一句警告：挑战通常要求你对正在构建的解释器进行更改。你需要在代码副本中实现这些。
后面的章节假设你的解释器处于原始（“未受挑战”？）状态。_

</aside>

### Design notes

Most "programming language" books are strictly programming language
*implementation* books. They rarely discuss how one might happen to *design* the
language being implemented. Implementation is fun because it is so <span
name="benchmark">precisely defined</span>. We programmers seem to have an
affinity for things that are black and white, ones and zeroes.

_大多数 "编程语言 "的书都是严格意义上的编程语言实现书。他们很少讨论如何设计被实现的语言。
实现是有趣的，因为它是如此精确的定义。我们程序员似乎对黑白分明的事物有一种亲和力，即1和0。_

<aside name="benchmark">

I know a lot of language hackers whose careers are based on this. You slide a
language spec under their door, wait a few months, and code and benchmark
results come out.

_我认识很多语言黑客，他们的职业就是以此为基础的。
你将语言规范滑到他们的门下，等待几个月，代码和基准测试结果就会出来。_

</aside>

Personally, I think the world needs only so many implementations of <span
name="fortran">FORTRAN 77</span>. At some point, you find yourself designing a
*new* language. Once you start playing *that* game, then the softer, human side
of the equation becomes paramount. Things like which features are easy to learn,
how to balance innovation and familiarity, what syntax is more readable and to
whom.

_我个人认为，世界上只需要这么多“FORTRAN 77”的实现。在某些时候，你会发现自己在设计一种新的语言。
一旦你开始玩这个游戏，那么软性的、人性化的一面就变得非常重要。
比如哪些功能容易学习，如何平衡创新和熟悉程度，什么样的语法更易读，对谁更易读。_

<aside name="fortran">

Hopefully your new language doesn't hardcode assumptions about the width of a
punched card into its grammar.

_希望你的新语言不会将关于打孔卡片宽度的假设硬编码到其语法中。_

</aside>

All of that stuff profoundly affects the success of your new language. I want
your language to succeed, so in some chapters I end with a "design note", a
little essay on some corner of the human aspect of programming languages. I'm no
expert on this -- I don't know if anyone really is -- so take these with a large
pinch of salt. That should make them tastier food for thought, which is my main
aim.

_所有这些东西都深刻地影响着你的新语言的成功。我希望你的语言能够成功，所以在某些章节中，
我以 "设计说明 "作为结尾，这是一篇关于编程语言中人类方面的某个角落的小文章。
我不是这方面的专家，我不知道是否有人真的是专家，所以请带着一大把盐来阅读这些文章。
这应该会使它们成为更可口的思想食粮，这是我的主要目的。_

## The First Interpreter

We'll write our first interpreter, jlox, in <span name="lang">Java</span>. The
focus is on *concepts*. We'll write the simplest, cleanest code we can to
correctly implement the semantics of the language. This will get us comfortable
with the basic techniques and also hone our understanding of exactly how the
language is supposed to behave.

_我们将用Java编写我们的第一个解释器，jlox。重点是概念。
我们将编写最简单、最干净的代码来正确实现语言的语义。
这将使我们适应基本的技术，同时也磨练我们对语言的确切行为方式的理解。_

<aside name="lang">

The book uses Java and C, but readers have ported the code to [many other
languages][port]. If the languages I picked aren't your bag, take a look at
those.

_本书使用Java和C语言，但读者已将代码移植到[许多其他语言][port]。
如果我挑选的语言不是你的囊中之物，可以看看那些语言。_

[port]: https://github.com/munificent/craftinginterpreters/wiki/Lox-implementations

</aside>

Java is a great language for this. It's high level enough that we don't get
overwhelmed by fiddly implementation details, but it's still pretty explicit.
Unlike in scripting languages, there tends to be less complex machinery hiding
under the hood, and you've got static types to see what data structures you're
working with.

_对于这个问题，Java是一种很好的语言。它的水平很高，我们不会被繁琐的实现细节所淹没，但它仍然很明确。
与脚本语言不同，隐藏在引擎盖下的复杂机器往往较少，而且你有静态类型可以看到你正在使用的数据结构。_

I also chose Java specifically because it is an object-oriented language. That
paradigm swept the programming world in the '90s and is now the dominant way of
thinking for millions of programmers. Odds are good you're already used to
organizing code into classes and methods, so we'll keep you in that comfort
zone.

_我选择了Java，特别还是因为它是一种面向对象的语言。这种模式在90年代席卷了整个编程界，
现在已经成为数百万程序员的主流思维方式。你很可能已经习惯于将代码组织成类和方法，
所以我们会让你保持在这个舒适区。_

While academic language folks sometimes look down on object-oriented languages,
the reality is that they are widely used even for language work. GCC and LLVM
are written in C++, as are most JavaScript virtual machines. Object-oriented
languages are ubiquitous, and the tools and compilers *for* a language are often
written *in* the <span name="host">same language</span>.

_虽然学术界的语言人士有时会看不起面向对象的语言，但现实是，即使在语言工作中，它们也被广泛使用。
GCC和LLVM是用C++编写的，大多数JavaScript虚拟机也是如此。面向对象的语言无处不在，
一种语言的工具和编译器往往是用同一种语言编写的。_

<aside name="host">

A compiler reads files in one language, translates them, and outputs files in
another language. You can implement a compiler in any language, including the
same language it compiles, a process called **self-hosting**.

_编译器读取一种语言的文件，对其进行翻译，然后输出另一种语言的文件。
你可以用任何语言中实现一个编译器，包括用它编译同一种语言，这个过程称为“self-hosting”。_

You can't compile your compiler using itself yet, but if you have another
compiler for your language written in some other language, you use *that* one to
compile your compiler once. Now you can use the compiled version of your own
compiler to compile future versions of itself, and you can discard the original
one compiled from the other compiler. This is called **bootstrapping**, from
the image of pulling yourself up by your own bootstraps.

_你还不能用自己的编译器来编译，但如果你有另一个用其他语言编写的你的语言的编译器，
你就用那个编译器来编译一次你的编译器。现在你可以用你自己的编译器的编译版本来编译它的未来版本，
而你可以丢弃从其他编译器上编译的原始版本。这就是所谓的"bootstrapping"，
来自于用自己的靴拔把自己拉起来的形象。_

<img src="image/introduction/bootstrap.png" alt="Fact: This is the primary mode of transportation of the American cowboy." />

</aside>

And, finally, Java is hugely popular. That means there's a good chance you
already know it, so there's less for you to learn to get going in the book. If
you aren't that familiar with Java, don't freak out. I try to stick to a fairly
minimal subset of it. I use the diamond operator from Java 7 to make things a
little more terse, but that's about it as far as "advanced" features go. If you
know another object-oriented language, like C# or C++, you can muddle through.

_而且，最后，Java是非常流行的。这意味着你很有可能已经知道它了，所以你在书中要学的东西就少了。
如果你对Java不是那么熟悉，也不要惊慌。我试图坚持使用它的一个相当小的子集。
我使用了Java 7中的棱形运算符，使事情变得更加简洁，但就 "高级 "功能而言，也就这些了。
如果你知道另一种面向对象的语言，如C#或C++，你可以蒙混过关。_

By the end of part II, we'll have a simple, readable implementation. It's not
very fast, but it's correct. However, we are only able to accomplish that by
building on the Java virtual machine's own runtime facilities. We want to learn
how Java *itself* implements those things.

_在第二部分结束时，我们会有一个简单的、可读的实现。它不是非常快，但它是正确的。
然而，我们只有通过建立在Java虚拟机自身的运行时设施上才能实现这一点。
我们要学习Java本身如何实现这些东西。_

## The Second Interpreter

So in the next part, we start all over again, but this time in C. C is the
perfect language for understanding how an implementation *really* works, all the
way down to the bytes in memory and the code flowing through the CPU.

_因此，在下一部分，我们重新开始，但这次是用C语言。
C语言是理解一个实现如何真正工作的完美语言，一直到内存中的字节和流经CPU的代码。_

A big reason that we're using C is so I can show you things C is particularly
good at, but that *does* mean you'll need to be pretty comfortable with it. You
don't have to be the reincarnation of Dennis Ritchie, but you shouldn't be
spooked by pointers either.

_我们使用C语言的一个重要原因是，我可以向你展示C语言特别擅长的东西，
但这确实意味着你需要对它有相当的了解。你不必是Dennis Ritchie的转世，但你也不应该被指针吓到。_

If you aren't there yet, pick up an introductory book on C and chew through it,
then come back here when you're done. In return, you'll come away from this book
an even stronger C programmer. That's useful given how many language
implementations are written in C: Lua, CPython, and Ruby's MRI, to name a few.

_如果你还没有达到这个境界，那就拿起一本C语言的入门书，仔细研读，读完后再来这里。
作为回报，这本书会使你成为一个更强大的C语言程序员。
鉴于有很多语言的实现是用C语言写的，这很有用。Lua、CPython和Ruby的MRI，仅举几例。_

In our C interpreter, <span name="clox">clox</span>, we are forced to implement
for ourselves all the things Java gave us for free. We'll write our own dynamic
array and hash table. We'll decide how objects are represented in memory, and
build a garbage collector to reclaim them.

_在我们的 C 解释器 clox 中，我们被迫为自己实现所有 Java 免费提供的东西。
我们将编写我们自己的动态数组和哈希表。
我们将决定对象在内存中如何表示，并建立一个垃圾收集器来回收它们。_

<aside name="clox">

I pronounce the name like "sea-locks", but you can say it "clocks" or even
"cloch", where you pronounce the "x" like the Greeks do if it makes you happy.

_我把“clox”这个名字念成"sea-locks"，但你也可以说成"clocks"，甚至是"cloch"，
你可以像希腊人那样念"x"，如果这让你高兴的话。_

</aside>

Our Java implementation was focused on being correct. Now that we have that
down, we'll turn to also being *fast*. Our C interpreter will contain a <span
name="compiler">compiler</span> that translates Lox to an efficient bytecode
representation (don't worry, I'll get into what that means soon), which it then
executes. This is the same technique used by implementations of Lua, Python,
Ruby, PHP, and many other successful languages.

_我们的Java实现专注于正确性。现在我们已经完成了这一点，然后我们将目标转向速度。
我们的C语言解释器将包含一个编译器，将Lox翻译成有效的字节码表示
（别担心，我很快就会说到这是什么意思），然后执行。
这与Lua、Python、Ruby、PHP和其他许多成功语言的实现所使用的技术相同。_

<aside name="compiler">

Did you think this was just an interpreter book? It's a compiler book as well.
Two for the price of one!

_你以为这只是一本解释器的书吗？它也是一本编译器书。一书两用!_

</aside>

We'll even try our hand at benchmarking and optimization. By the end, we'll have
a robust, accurate, fast interpreter for our language, able to keep up with
other professional caliber implementations out there. Not bad for one book and a
few thousand lines of code.

_我们甚至会在基准测试和优化方面尝试我们的手。
到最后，我们将拥有一个强大、准确、快速的语言解释器， 能够跟上其他专业口径的实现。
对于一本书和几千行代码来说，这并不坏。_

<div class="challenges">

## Challenges

1.  There are at least six domain-specific languages used in the [little system
    I cobbled together][repo] to write and publish this book. What are they?

1.  Get a "Hello, world!" program written and running in Java. Set up whatever
    makefiles or IDE projects you need to get it working. If you have a
    debugger, get comfortable with it and step through your program as it runs.

1.  Do the same thing for C. To get some practice with pointers, define a
    [doubly linked list][] of heap-allocated strings. Write functions to insert,
    find, and delete items from it. Test them.

[repo]: https://github.com/munificent/craftinginterpreters
[doubly linked list]: https://en.wikipedia.org/wiki/Doubly_linked_list

</div>

<div class="design-note">

## Design Note: What's in a Name?

One of the hardest challenges in writing this book was coming up with a name for
the language it implements. I went through *pages* of candidates before I found
one that worked. As you'll discover on the first day you start building your own
language, naming is deviously hard. A good name satisfies a few criteria:

1.  **It isn't in use.** You can run into all sorts of trouble, legal and
    social, if you inadvertently step on someone else's name.

2.  **It's easy to pronounce.** If things go well, hordes of people will be
    saying and writing your language's name. Anything longer than a couple of
    syllables or a handful of letters will annoy them to no end.

3.  **It's distinct enough to search for.** People will Google your language's
    name to learn about it, so you want a word that's rare enough that most
    results point to your docs. Though, with the amount of AI search engines are
    packing today, that's less of an issue. Still, you won't be doing your users
    any favors if you name your language "for".

4.  **It doesn't have negative connotations across a number of cultures.** This
    is hard to be on guard for, but it's worth considering. The designer of
    Nimrod ended up renaming his language to "Nim" because too many people
    remember that Bugs Bunny used "Nimrod" as an insult. (Bugs was using it
    ironically.)

If your potential name makes it through that gauntlet, keep it. Don't get hung
up on trying to find an appellation that captures the quintessence of your
language. If the names of the world's other successful languages teach us
anything, it's that the name doesn't matter much. All you need is a reasonably
unique token.

</div>
