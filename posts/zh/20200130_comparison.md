# Rust 编译模型之殇

*发表于 2020.01.30  作者：Brian Anderson* 


![img1](../../imgs/rust-compile-time-adventures.png)

*Rust 编译缓慢的根由在于语言的设计。*

我的意思并非是此乃 Rust 语言的*设计目标*。正如语言设计者们相互争论时经常说的那样，编程语言的设计总是充满了各种权衡。其中最主要的权衡就是：**运行时性能** 和 **编译时性能**。而 Rust 团队几乎总是选择运行时而非编译时。

因此，Rust 编译时间很慢。这有点让人恼火，因为 Rust 在其他方面的表现都非常好，唯独 Rust 编译时间却表现如此糟糕。

## Rust 与 TiKV 的编译时冒险：第1集

在 [PingCAP](https://pingcap.com/)，我们基于 Rust 开发了分布式存储系统 [TiKV](https://github.com/tikv/tikv/) 。然而它的编译速度慢到足以让公司里的许多人不愿使用 Rust。我最近花了一些时间，与TiKV团队及其社区中的其他几人一起调研了 TiKV 编译时间缓慢的问题。

通过这一系列博文，我将会讨论在这个过程中的收获：

* 为什么 Rust 编译那么慢，或者说让人感觉那么慢；
* Rust 的发展如何造就了编译时间的缓慢；
* 编译时用例；
* 我们测量过的，以及想要测量但还没有或者不知道如何测量的项目；
* 改善编译时间的一些思路；
* 事实上未能改善编译时间的思路；
* TiKV 编译时间的历史演进
* 有关如何组织 Rust 项目可加速编译的建议；
* 最近和未来，上游将对编译时间的改进；

---

本集包括如下内容:

*  [PingCAP的阴影：TiKV 编译次数 “余额不足”](https://pingcap.com/blog/rust-compilation-model-calamity/#the-spectre-of-poor-rust-compile-times-at-pingcap) 
*  [概览: TiKV 编译时冒险历程](https://pingcap.com/blog/rust-compilation-model-calamity/#preview-the-tikv-compile-time-adventure-so-far) 
*  [造就编译时间缓慢的 Rust 设计](https://pingcap.com/blog/rust-compilation-model-calamity/#rusts-designs-for-poor-compilation-time) 	   
    *  [Rust 的自举](https://pingcap.com/blog/rust-compilation-model-calamity/#bootstrapping-rust) 
	*  [（非）良性循环](https://pingcap.com/blog/rust-compilation-model-calamity/#unvirtuous-cycles) 
	*  [运行时优先于编译时的早期决策](https://pingcap.com/blog/rust-compilation-model-calamity/#early-decisions-that-favored-run-time-over-compile-time) 
*  [改善 Rust 编译时间的最新进展](https://pingcap.com/blog/rust-compilation-model-calamity/#recent-work-on-rust-compile-times) 
*  [下集预告](https://pingcap.com/blog/rust-compilation-model-calamity/#in-the-next-episode) 
*  [鸣谢](https://pingcap.com/blog/rust-compilation-model-calamity/#thanks) 

## PingCAP的阴影：TiKV 编译次数 “余额不足”

At [PingCAP](https://pingcap.com/en/) , my colleagues use Rust to write  [TiKV](https://github.com/tikv/tikv/) , the storage node of  [TiDB](https://github.com/pingcap/tidb) , our distributed database. They do this because they want this most important node in the system to be fast and reliable by construction, at least to the greatest extent reasonable.

在 [PingCAP](https://pingcap.com/en/)，我的同事用Rust写 [TiKV](https://github.com/tikv/tikv/)。它是我们的分布式数据库 [TiDB](https://github.com/pingcap/tidb) 的存储节点。采用这样的架构，是因为他们希望该系统中作为最重要的节点，能被构造的快速且可靠，至少是在一个最大程度的合理范围内。（译注：通常情况下人们认为快和可靠是很难同时做到的，人们只能在设计/构造的时候做出权衡。选择 Rust 是为了尽可能让 TiKV 能够在尽可能合理的情况下去提高它的速度和可靠性。）

It was mostly a great decision, and most people internally are mostly happy about it.

这是一个很棒的决定，并且团队内大多数人对此都非常满意。

But many complain about how long it takes to build. For some, a full rebuild might take 15 minutes in development mode, and 30 minutes in release mode. To developers of large systems projects, this might not sound so bad, but it’s much slower than what many developers expect out of modern programming environments. TiKV is a relatively large Rust codebase, with 2 million lines of Rust. In comparison, Rust itself contains over 3 million lines of Rust, and  [Servo](https://github.com/servo/servo)  contains 2.7 million (see  [full line counts here](https://gist.github.com/brson/31b6f8c5467b050779ce9aa05d41aa84) ).

但是许多人抱怨构建的时间太长。有时，在开发模式下完全重新构建需要花费 15 分钟，而在发布模式则需要 30 分钟。对于大型系统项目的开发者而言，这看上去可能并不那么糟糕。但是它与许多开发者从现代的开发环境中期望得到的速度相比则慢了很多。TiKV 是一个相当巨大的代码库，它拥有 200 万行 Rust 代码。相比之下，Rust 自身包含超过 300 万行 Rust 代码，而 [Servo](https://github.com/servo/servo)  包含 270 万行（请参阅 [此处的完整行数统计](https://gist.github.com/brson/31b6f8c5467b050779ce9aa05d41aa84) ）。

Other nodes in TiDB are written in Go, which of course comes with a different set of advantages and disadvantages from Rust. Some of the Go developers at PingCAP resent having to wait for the Rust components to build. They are used to a rapid build-test cycle.

TiDB 中的其他节点是用 Go 编写的，当然，Go 与 Rust 有不同的优点和缺点。PingCAP 的一些 Go 开发人员对不得不等待 Rust 组件的构建而表示不满。因为他们习惯于快速的构建-测试迭代。

Rust developers, on the other hand, are used to taking a lot of coffee breaks (or tea, cigarettes, sobbing, or whatever as the case may be — Rust developers have the spare time to nurse their demons).

在GO开发人员忙碌工作的同时，Rust 开发人员却在编译时间休息(喝咖啡、喝茶、抽烟，或者诉苦)。Rust开发人员有多余的时间（Go 开发者却没有）来跨越内心的“阴影（译注：据说，TiKV 一天只有 24 次编译机会，用一次少一次）”。

## 概览: TiKV 编译时冒险历程

The first entry in this series is just a story about the history of Rust with respect to compilation time. Since it might take several more entries before we dive into concrete technical details of what we’ve done with TiKV’s compile times, here’s a pretty graph to capture your imagination, without comment.

本系列的第一篇文章只是关于 Rust 在编译时间方面的历史演进。因为在我们深入研究TiKV编译时间的具体技术细节之前，可能需要更多的篇章。所以，这里先放一个漂亮的图表,无需多言。

![img2](../../imgs/rust-compile-times-tikv.svg)

* TiKV 的 Rust 编译时间*

## 造就编译时间缓慢的 Rust 设计

Rust was designed for slow compilation times.

Rust 编译缓慢的根由在于语言的设计。

I mean, that wasn’t *the goal*. As is often cautioned in debates among their designers, programming language design is full of tradeoffs. One of those fundamental tradeoffs is *run-time performance* vs. *compile-time performance*, and the Rust team nearly always (if not always) chose run-time over compile-time.

我的意思并非是此乃 Rust 语言的*设计目标*。正如语言设计者们相互争论时经常说的那样，编程语言的设计总是充满了各种权衡。其中最主要的权衡就是：**运行时性能** 和 **编译时性能**。而 Rust 团队几乎总是选择运行时而非编译时。

The intentional run-time/compile-time tradeoff isn’t the only reason Rust compile times are horrific, but it’s a big one. There are also language designs that are not crucial for run-time performance, but accidentally bad for compile-time performance. The Rust compiler was also implemented in ways that inhibit compile-time performance.

刻意的运行时/编译时权衡不是 Rust 编译时间差劲的唯一原因，但这是一个大问题。还有一些语言设计对运行时性能并不是至关重要，但却意外地有损于编译时性能。Rust编译器的实现方式也抑制了编译时性能。

So there are intrinsic language-design reasons and accidental language-design reasons for Rust’s bad compile times. Those mostly can’t be fixed ever, although they may be mitigated by compiler improvements, design patterns, and language evolution. There are also accidental compiler-architecture reasons for Rust’s bad compile times, which can generally be fixed through enormous engineering effort and time.

所以，Rust 编译时间的差劲，既是刻意为之的造就，又有出于设计之外的原因。尽管编译器的改善、设计模式和语言的发展可能会缓解这些问题，但这些问题大多无法得到解决。还有一些偶然的编译器架构原因导致了 Rust 的编译时间很慢，这些需要通过大量的工程时间和精力来修复。

If fast compilation time was not a core Rust design principle, what were Rust’s core design principles? Here are a few:

如果迅速地编译不是 Rust 的核心设计原则，那么 Rust 的核心设计原则是什么呢？下面列出几个核心设计原则：

* *Practicality* — it should be a language that can be and is used in the real world.
* *Pragmatism* — it should admit concessions to human usability and integration into systems as they exist today.
* *Memory-safety* — it must enforce memory safety, and not admit segmentation faults and other such memory-access violations.
* *Performance* — it must be in the same performance class as C++.
* *Concurrency* — it must provide modern solutions to writing concurrent code.

* *实用性（Practicality）* — 它应该是一种可以在现实世界中使用的语言。
* *务实（Pragmatism）* — 它应该是符合人性化体验，并且能与现有系统方便集成的语言。
* *内存安全性（Memory-safety）* — 它必须加强内存安全，不允许出现段错误和其他类似的内存访问违规操作。
* *高性能（Performance）* — 它必须拥有能和 C++ 比肩的性能。
* *高并发（Concurrency）* — 它必须为编写并发代码提供现代化的解决方案。

But it’s not like the designers didn’t put /any/ consideration into fast compile times. For example, for any analysis Rust needs to do, the team tried to ensure reasonable bounds on computational complexity. Rust’s design history though is one of increasingly being sucked into a swamp of poor compile-time performance.

但这并不是说设计者没有为编译速度做*任何*考虑。例如，对于编译Rust代码所要做的任何编译步骤，团队试图确保算法复杂的合理性。然而，Rust 的设计历史也是其一步步陷入糟糕的编译时性能沼泽的历史。

Story time.

讲故事的时间到了。

## Rust 的自举

I don’t remember when I realized that Rust’s bad compile times were a strategic problem for the language, potentially a fatal mistake in the face of competition from future low-level programming languages. For the first few years, hacking almost entirely on the Rust compiler itself, I wasn’t too concerned, and I don’t think most of my peers were either. I mostly remember that Rust compile time was always bad, and like, whatever, I can deal with that.

我不记得自己是什么时候才开始意识到，Rust 糟糕的编译时间其实是该语言的一个战略问题。在面对未来底层编程语言的竞争时可能会是一个致命的错误。在最初的几年里，我几乎完全是对Rust编译器进行Hacking（非常规暴力测试），我并不太关心编译时间的问题，我也不认为其他大多数同事会太关心该问题。我印象中大部分时间Rust编译时总是很糟糕，但不管怎样，我能处理好。

When I worked daily on the Rust compiler, it was common for me to have at least three copies of the repository on the computer, hacking on one while all the others were building and testing. I would start building workspace 1, switch terminals, remember what’s going on over here in workspace 2, hack on that for a while, start building in workspace 2, switch terminals, etc. Little flow, constant context switching.


针对 Rust 编译器工作的时候，我通常都会在计算机上至少保留三份存储库副本，在其他所有的编译器都在构建和测试时，我就会 Hacking 其中的一份。我会开始编译工作空间 1，切换终端，回想起在工作空间 2 发生了什么，做一下修改，然后再开始编译工作空间 2，切换终端，等等。整个流程比较零碎且经常切换上下文。

This was (and probably is) typical of other Rust developers too. I still do the same thing hacking on TiKV today.

这（可能）也是其他 Rust 开发者的日常。我现在对 TiKV 也经常在做类似的 Hacking 测试。

So, historically, how bad have Rust compile times been? A simple barometer here is to see how Rust’s self-hosting times have changed over the years, which is the time it takes Rust to build itself. Rust building itself is not directly comparable to Rust building other projects, for a variety of reasons, but I think it will be illustrative.

那么，从历史上看，Rust 编译时间有多糟糕呢？这里有一个简单的统计表，可以看到 Rust 的自举（Self-Hosting）时间在过去几年里发生了怎样的变化，也就是使用 Rust 来构建它自己的时间。出于各种原因，虽然 Rust 构建自己不能直接与 Rust 构建其他项目相比，但我认为这能说明一些问题。

The  [first Rust compiler](https://gist.github.com/brson/31b6f8c5467b050779ce9aa05d41aa84/edit) , from 2010, called rustboot, was written in OCaml, and it’s ultimate purpose was to build a second compiler, rustc, written in Rust, and begin the self-hosting bootstrap cycle. In addition to being written in Rust, rustc would also use  [LLVM](https://llvm.org/)  as its backend for generating machine code, instead of rustboot’s hand-written x86 code-generator.

首个 [Rust 编译器](https://gist.github.com/brson/31b6f8c5467b050779ce9aa05d41aa84/edit) 叫做 rustboot，始于 2010 年，是用OCaml编写的，它最终目的是被用于构建第二个由 Rust 实现的编译器 rustc，并由此开启了 Rust 自举的历程。 除了基于 Rust 编写之外，rustc 还使用了 [LLVM](https://llvm.org/) 作为后端来生成机器代码，来代替之前 rustboot 的手写 x86 代码生成器。

Rust needed to become self-hosting as a means of “dog-fooding” the language — writing the Rust compiler in Rust meant that the Rust authors needed to use their own language to write practical software early in the language design process. It was hoped that self-hosting could lead to a useful and practical language.

Rust需要自举，那样就可以作为一种“自产自销（Dog-Fooding）”的语言。使用 Rust 编写编译器意味着Rust 的作者们需要在语言设计过程的早期，使用自己的语言来编写实用的软件。在实现自举的过程中让 RUST 变成一种实用的语言。

The first time Rust built itself was on April 20, 2011.  [It took one hour](https://mail.mozilla.org/pipermail/rust-dev/2011-April/000330.html) , which was a laughably long time. At least it was back then.

Rust 第一次自举构建是在 2011 年 4 月 20 日。该过程总共花了一个小时，在当时我们觉得这甚至很可笑。

That first super-slow bootstrap was an anomaly of bad code-generation and other easily fixable early bugs (probably, I don’t exactly recall). rustc’s performance quickly improved, and Graydon quickly  [threw away the old rustboot compiler](https://github.com/rust-lang/rust/commit/6997adf76342b7a6fe03c4bc370ce5fc5082a869)  since there was nowhere near enough manpower and motivation to maintain parallel implementations.

最初那个超级慢的自举程序慢的有些反常，在于其包含了糟糕的代码生成和其他容易修复的早期错误(可能，我记不清了)。rustc 的性能很快得到了改善，Graydon 很快就[抛弃了旧的 rustboot 编译器](https://github.com/rust-lang/rust/commit/6997adf76342b7a6fe03c4bc370ce5fc5082a869) ，因为没有足够的人力和动力来维护两套实现。

This is where the long, gruelling history of Rust’s tragic compile times began, 11 months after it was initially released in June 2010.

在 2010 年 6 月首次发布的 11 个月之后，Rust 编译时长的可悲漫长而艰难的历史就此开始了。

*注意*

I wanted to share historic self-hosting times here, but after many hours and obstacles attempting to build Rust revisions from 2011, I finally gave up and decided I just had to publish this piece without them. Instead, here are some madeup numbers:

我本想在这里分享一些有历史意义的自举时间，但在经历了数小时，以及试图从2011年开始构建Rust修订版的障碍之后，我终于放弃了，决定在没有它们的情况下发布这篇文章。作为补充，这里作一个类比：

* /7 femto-bunnies/ - rustboot building Rust prior to being retired
* /49 kilo-hamsters/ - rustc building Rust immediately after rustboot’s retirement
* /188 giga-sloths/ - rustc building Rust in 2020

* * 兔子飞奔几米 （7）* - rustboot 构建 Rust 的时间
* * 仓鼠狂奔一公里 （49）* - 在 rustboot 退役后使用 rustc 构建 Rust 的时间
* * 树獭移动一万米 （188）*  - 在 2020 年构建 rustc 所需的时间

Anyway, last time I bootstrapped Rust a few months ago, it took over five hours.

反正，几个月前我构建 Rust 的时候，花了五个小时。

The Rust language developers became acclimated to Rust’s poor self-hosting times and failed to recognize or address the severity of the problem of bad compile times during Rust’s crucial early design phase.

Rust 语言开发者们已经适应了 Rust 糟糕的自举时间，并且在 Rust 的关键早期设计阶段未能识别或处理糟糕编译时间问题的严重性。

## （非）良性循环

In the Rust project, we like processes that reinforce and build upon themselves. This is one of the keys to Rust’s success, both as a language and a community.

在Rust项目中，我们喜欢能够增强自身基础的流程。 无论是作为语言还是社区，这都是Rust取得成功的关键之一。

As an obvious, hugely-successful example, consider  [Servo](https://github.com/servo/servo) . Servo is a web browser built in Rust, and Rust was created with the explicit purpose of building Servo. Rust and Servo are sister-projects. They were created by the same team (initially), at roughly the same time, and they evolved together. Not only was Rust built to create Servo, but Servo was built to inform the design of Rust.

一个明显非常成功的例子就是 [Servo](https://github.com/servo/servo)。 Servo 是一个基于 Rust 构建的 Web 浏览器，并且 Rust 也是为了构建 Servo 而诞生。Rust 和 Servo 是姊妹项目。它们是由同一个（初始）团队，在（大致）同一时间创造的，并同时进化。不只是为了创造 Servo 而创建 Rust，而且 Servo 也是为了解 Rust 的设计而构建的。

The initial few years of both projects were extremely difficult, with both projects evolving in parallel. The often-used metaphor of the  [Ship of Theseus](https://en.wikipedia.org/wiki/Ship_of_Theseus)  is apt - we were constantly rebuilding Rust in order to sail the seas of Servo. There is no doubt that the experience of building Servo with Rust while simultaneously building the language itself led directly to many of the good decisions that make Rust the practical language it is.

这两个项目最初的几年都非常困难，两个项目都是并行发展的。此处非常适合用 [忒修斯之船](https://en.wikipedia.org/wiki/Ship_of_Theseus) 做比喻 —— 我们不断地重建 Rust，以便在 Sevro 的海洋中畅行。毫无疑问，使用 Rust 构建 Servo 的经验，来构建 Rust 语言本身，直接促进了很多好的决定，使得 Rust 成为了实用的语言。

Here are some cursory examples of the Servo-Rust feedback loop:

这里有一些关于 Servo-Rust 反馈回路的例子：

* Labeled break and continue  [was implemented in order to auto-generate an HTML parser](https://github.com/rust-lang/rust/issues/2216) .

* 为了[自动生成HTML解析器](https://github.com/rust-lang/rust/issues/2216)，实现了带标签的 break 和 continue 。 

* Owned closures  [were implemented after analyzing closure usage in Servo](https://github.com/rust-lang/rust/issues/2549#issuecomment-19588158) .

* [在分析了 Servo 内闭包使用情况之后实现了](https://github.com/rust-lang/rust/issues/2549#issuecomment-19588158) ，所有权闭包（Owned closures）。

* External function calls used to be considered safe.  [This changed in part due to experience in Servo](https://github.com/rust-lang/rust/issues/2628#issuecomment-9384243) .

* 外部函数调用曾经被认为是安全的。[这部分变化（改为了 Unsafe ）得益于 Servo 的经验](https://github.com/rust-lang/rust/issues/2628#issuecomment-9384243) 

* The migration from green-threading to native threading was informed by the experience of building Servo, observing the FFI overhead of Servo’s SpiderMonkey integration, and profiling “hot splits”, where the green thread stacks needed to be expanded and contracted.

* 从绿色线程迁移到本地线程，也是由构建 Sevro、观察 Servo 中 SpiderMonkey 集成的 FFI 开销以及剖析“hot splits”的经验所决定的，其中绿色线程堆栈需要扩展和收缩。

The co-development of Rust and Servo created a  [virtuous cycle](https://en.wikipedia.org/wiki/Virtuous_circle_and_vicious_circle)  that allowed both projects to thrive. Today, Servo components are deeply integrated into Firefox, ensuring that Rust cannot die while Firefox lives.

Rust 和 Servo 的共同发展创造了一个 [良性循环](https://en.wikipedia.org/wiki/Virtuous_circle_and_vicious_circle) ，使这两个项目蓬勃发展。今天，Servo 组件被深度集成到火狐（Firefox）中，确保在火狐存活的时候，Rust 不会死去。

Mission accomplished.

任务完成了。

![img3](../../imgs/rust-compile-mission-completed.png)


The previously-mentioned early self-hosting was similarly crucial to Rust's design, making Rust a superior language for building Rust compilers. Likewise, Rust and  [WebAssembly](https://webassembly.org/)  were developed in close collaboration (the author of  [Emscripten](https://github.com/emscripten-core/emscripten) , the author of  [Cranelift](https://github.com/CraneStation/cranelift) , and I had desks next to each other for years), making WASM an excellent platform for running Rust, and Rust well-suited to target WASM.

前面提到的早期自举对 Rust 的设计同样至关重要，使得 Rust 成为构建 Rust 编译器的优秀语言。同样，Rust 和 [WebAssembly](https://webassembly.org/)  是在密切合作下开发的（我与 [Emscripten](https://github.com/emscripten-core/emscripten)  的作者，[Cranelift](https://github.com/CraneStation/cranelift) 的作者并排工作好几年)，这使得 WASM 成为了一个运行 Rust 的优秀平台，而 Rust 也非常适合 WASM。

Sadly there was no such reinforcement to drive down Rust compile times. The opposite is probably true — the more Rust became known as a /fast/ language, the more important it was to be /the fastest/ language. And, the more Rust's developers got used to developing their Rust projects across multiple branches, context switching between builds, the less pressure was felt to address compile times.

遗憾的是，没有这样的增强来缩短 Rust 编译时间。事实可能正好相反 —— Rust越是被认为是一种快速语言，它成为最快的语言就越重要。而且，Rust 的开发人员越习惯于跨多个分支开发他们的 Rust 项目，在构建之间切换上下文，就越不需要考虑编译时间。

This only really changed once Rust 1.0 was released in 2015 and started to receive wider use.

直到 2015 年 Rust 1.0 发布并开始得到更广泛的应用后，这种情况才真正有所改变。

For years Rust  [slowly boiled](https://en.wikipedia.org/wiki/Boiling_frog)  in its own poor compile times, not realizing how bad it had gotten until it was too late. It was 1.0. Those decisions were locked in.

多年来，Rust 在糟糕的编译时间[“温水中”被慢慢“亨煮”](https://en.wikipedia.org/wiki/Boiling_frog)，当意识到它已经变得多么糟糕时，已为时已晚。已经 1.0 了。那些（设计）决策早已被锁定了。

Too many tired metaphors in this section. Sorry about that.

这一节包含了太多令人厌倦的隐喻，抱歉了。

## 运行时优先于编译时的早期决策

If Rust is designed for poor compile time, then what are those designs specifically? I describe a few briefly here. The next episode in this series will go into further depth. Some have greater compile-time impact than others, but I assert that all of them cause more time to be spent in compilation than alternative designs.

如果是 Rust 设计导致了糟糕的编译时间，那么这些设计具体又是什么呢? 我会在这里简要地描述一些。本系列的下一集将会更加深入。有些在编译时的影响比其他的更大，但是我断言，所有这些都比其他的设计耗费更多的编译时间。

Looking at some of these in retrospect, I am tempted to think that “well, of course Rust /must/ have feature /foo/", and it's true that Rust would be a completely different language without many of these features. However, language designs are tradeoffs and none of these were predestined to be part of Rust.

现在回想起来，我不禁会想，“当然，Rust 必须有这些特性”。确实，如果没有这些特性，Rust将会是另一门完全不同的语言。然而，语言设计是折衷的，这些并不是注定要成 Rust 的部分。

* /Borrowing/ — Rust's defining feature. Its sophisticated pointer analysis spends compile-time to make run-time safe.

* *借用（Borrowing）* —— Rust 的典型功能。其复杂的指针分析以编译时的花费来换取运行时安全。

* /Monomorphization/ — Rust translates each generic instantiation into its own machine code, creating code bloat and increasing compile time.

* *单态化（Monomorphization）* —— Rust 将每个泛型实例转换为各自的机器代码，从而导致代码膨胀并增加了编译时间。

* /Stack unwinding/ — stack unwinding after unrecoverable exceptions traverses the callstack backwards and runs cleanup code. It requires lots of compile-time book-keeping and code generation.

* *栈展开（Stack unwinding）* —— 不可恢复异常发生后，栈展开向后遍历调用栈并运行清理代码。它需要大量的编译时登记（book-keeping）和代码生成。

* /Build scripts/ — build scripts allow arbitrary code to be run at compile-time, and pull in their own dependencies that need to be compiled. Their unknown side-effects and unknown inputs and outputs limit assumptions tools can make about them, which e.g. limits caching opportunities.

* *构建脚本（Build scripts）* —— 构建脚本允许在编译时运行任意代码，并引入它们自己需要编译的依赖项。它们未知的副作用和未知的输入输出限制了工具对它们的假设，例如限制了缓存的可能。

* /Macros/ — macros require multiple passes to expand, expand to often surprising amounts of hidden code, and impose limitations on partial parsing. Procedural macros have negative impacts similar to build scripts.

* *宏（Macros）* —— 宏需要多次遍历才能展开，展开得到的隐藏代码量惊人，并对部分解析施加限制。 过程宏与构建脚本类似，具有负面影响。

* /LLVM backend/ — LLVM produces good machine code, but runs relatively slowly.

* *LLVM 后端（LLVM backend）* —— LLVM产生良好的机器代码，但编译相对较慢。

* /Relying too much on the LLVM optimizer/ — Rust is well-known for generating a large quantity of LLVM IR and letting LLVM optimize it away. This is exacerbated by duplication from monomorphization.

* *过于依赖LLVM优化器（Relying too much on the LLVM optimizer）* —— Rust 以生成大量LLVM IR 并让 LLVM 对其进行优化而闻名。单态化则会加剧这种情况。

* /Split compiler/package manager/ — although it is normal for languages to have a package manager separate from the compiler, in Rust at least this results in both cargo and rustc having imperfect and redundant information about the overall compilation pipeline. As more parts of the pipeline are short-circuited for efficiency, more metadata needs to be transferred between instances of the compiler, mostly through the filesystem, which has overhead.

* *拆分编译器/软件包管理器（Split compiler/package manager）* —— 尽管对于语言来说，将包管理器与编译器分开是很正常的，但是在 Rust 中，至少这会导致 cargo 和 rustc 同时携带关于整个编译流水线的不完善和冗余的信息。当流水线的更多部分被短路以便提高效率时，则需要在编译器实例之间传输更多的元数据。这主要是通过文件系统进行传输，会产生开销。

* /Per-compilation-unit code-generation/ — rustc generates machine code each time it compiles a crate, but it doesn't need to — with most Rust projects being statically linked, the machine code isn't needed until the final link step. There may be efficiencies to be achieved by completely separating analysis and code generation.

* *每个编译单元的代码生成（Per-compilation-unit code-generation）* —— rustc每次编译单包（crate）时都会生成机器码，但是它不需要这样做，因为大多数 Rust 项目都是静态链接的，直到最后一个链接步骤才需要机器码。可以通过完全分离分析和代码生成来提高效率。

* /Single-threaded compiler/ — ideally, all CPUs are occupied for the entire compilation. This is not close to true with Rust today. And with the original compiler being single-threaded, the language is not as friendly to parallel compilation as it might be. There are efforts going into parallelizing the compiler, but it may never use all your cores.

* *单线程的编译器（Single-threaded compiler）* —— 理想情况下，整个编译过程都将占用所有CPU。 然而，Rust并非如此。由于原始编译器是单线程的，因此该语言对并行编译不够友好。目前正在努力使编译器并行化，但它可能永远不会使用所有 CPU 核心。

* /Trait coherence/ — Rust’s traits have a property called “coherence”, which makes it impossible to define implementations that conflict with each other. Trait coherence imposes restrictions on where code is allowed to live. As such, it is difficult to decompose Rust abstractions into, small, easily-parallelizable compilation units.

* *trait 一致性（trait coherence）* —— Rust 的 trait（特质）需要遵循“一致性（conherence）”，这使得开发者不可能定义相互冲突的实现。trait 一致性对允许代码驻留的位置施加了限制。这样，很难将 Rust 抽象分解为更小的、易于并行化的编译单元。

* /Tests next to code/ — Rust encourages tests to reside in the same codebase as the code they are testing. With Rust’s compilation model, this requires compiling and linking that code twice, which is expensive, particularly for large crates.

* *“亲密”的代码测试（Tests next to code）* —— Rust 鼓励测试代码与功能代码驻留在同一代码库中。 由于 Rust 的编译模型，这需要将该代码编译和链接两次，这份开销非常昂贵，尤其是对于有很多包（crate）的大型项目而言。

## 改善 Rust 编译时间的最新进展

The situation isn’t hopeless. Not at all. There is always work going on to improve Rust compile times, and there are still many avenues to be explored. I’m hopeful that we’ll continue to see improvements. Here is a selection of the activities I’m aware of from the last year or two. Thanks to everybody who helps with this problem.

现状并非没有改善的希望。一直有很多工作在努力改善 Rust 的编译时间，但仍有许多途径可以探索。我希望我们能持续看到进步。以下是我最近一两年所知道的一些进展。感谢所有为该问题提供帮助的人。

* The Rust compile-time  [master issue](https://github.com/rust-lang/rust/issues/48547) 
	* Tracks various work to improve compile times
	* Contains a great overview of factors that affect Rust compilation performance and potential mitigation strategies 

* Rust 编译时[主要问题](https://github.com/rust-lang/rust/issues/48547) 
	* 跟踪各种工作以缩短编译时间
	* 全面概述了影响 Rust 编译性能的因素和潜在的缓解策略


* Pipelined compilation ([1](https://github.com/rust-lang/rust/issues/60988),[2](https://github.com/rust-lang/cargo/issues/6660),[3](https://internals.rust-lang.org/t/evaluating-pipelined-rustc-compilation/10199))
	* Typechecks downstream crates in parallel with upstream codegen. Now on by default on the stable channel
	* Developed by  [@alexcrichton](https://github.com/alexcrichton)  and  [@nikomatsakis](https://github.com/nikomatsakis) .

* 流水线编译 ([1](https://github.com/rust-lang/rust/issues/60988),[2](https://github.com/rust-lang/cargo/issues/6660),[3](https://internals.rust-lang.org/t/evaluating-pipelined-rustc-compilation/10199))
	* 与上游代码生成并行地对下游包进行类型检查。现在默认情况下在稳定（Stable）频道上
	* 由 [@alexcrichton](https://github.com/alexcrichton)  和  [@nikomatsakis](https://github.com/nikomatsakis) 开发

* Parallel rustc ([1](https://internals.rust-lang.org/t/parallelizing-rustc-using-rayon/6606),[2](https://github.com/rust-lang/rust/issues/48685),[3](https://internals.rust-lang.org/t/help-test-parallel-rustc/11503/14))
	* Runs analysis phases of the compiler in parallel. Not yet available on the stable channel
	* Developed by  [@Zoxc](https://github.com/Zoxc) ,  [@michaelwoerister](https://github.com/michaelwoerister) ,  [@oli-obk](http://github.com/oli-obk) , and others

* 并行 rustc ([1](https://internals.rust-lang.org/t/parallelizing-rustc-using-rayon/6606),[2](https://github.com/rust-lang/rust/issues/48685),[3](https://internals.rust-lang.org/t/help-test-parallel-rustc/11503/14))
	* 并行运行编译器的分析阶段。稳定（Stable）频道尚不可用
	* 由 [@Zoxc](https://github.com/Zoxc) ,  [@michaelwoerister](https://github.com/michaelwoerister) ,  [@oli-obk](http://github.com/oli-obk) , 以及其他一些人开发


*  [MIR-level constant propagation](https://blog.rust-lang.org/inside-rust/2019/12/02/const-prop-on-by-default.html) 
	* Performs constant propagation on MIR, which reduces duplicated LLVM work on monomorphized functions
	* Developed by  [@wesleywiser](https://github.com/wesleywiser) 

*  [MIR 级别的常量传播（constant propagation）](https://blog.rust-lang.org/inside-rust/2019/12/02/const-prop-on-by-default.html) 
	* 在 MIR 上执行常量传播，从而减少了 LLVM 对单态函数的重复工作
	* 由 [@wesleywiser](https://github.com/wesleywiser) 开发

*  [MIR optimizations](https://github.com/rust-lang/rust/pulls?q=mir-opt) 
	* Optimizing MIR should be faster than optimizeng monomorphized LLVM IR
	* Not in stable compilers yet
	* Developed by  [@wesleywiser](https://github.com/wesleywiser)  and others

*  [MIR 优化](https://github.com/rust-lang/rust/pulls?q=mir-opt) 
	* 优化 MIR 应该比优化单态 LLVM IR 更快
	* 稳定（Stable）编译器尚不可用
	* 由 [@wesleywiser](https://github.com/wesleywiser) 和其他人一起开发

* cargo build -Ztimings ([1](https://internals.rust-lang.org/t/exploring-crate-graph-build-times-with-cargo-build-ztimings/10975),[2](https://github.com/rust-lang/cargo/issues/7405))
	* Collects and graphs information about cargo’s parallel build timings
	* Developed by  [@ehuss](https://github.com/ehuss)  and  [@luser](https://github.com/luser) 

* cargo build -Ztimings ([1](https://internals.rust-lang.org/t/exploring-crate-graph-build-times-with-cargo-build-ztimings/10975),[2](https://github.com/rust-lang/cargo/issues/7405))
	* 收集并图形化有关 Cargo 并行建造时间的信息
	* 由 [@ehuss](https://github.com/ehuss)  和  [@luser](https://github.com/luser) 开发

* rustc -Zself-profile ([1](https://rust-lang.github.io/rustc-guide/profiling.html),[2](https://github.com/rust-lang/rust/issues/58967),[3](https://github.com/rust-lang/rust/pull/51657))
	* Generates detailed information about rustc’s internal performance
	* Developed by  [@wesleywiser](https://github.com/wesleywiser)  and  [@michaelwoerister](https://github.com/michaelwoerister) 

* rustc -Zself-profile ([1](https://rust-lang.github.io/rustc-guide/profiling.html),[2](https://github.com/rust-lang/rust/issues/58967),[3](https://github.com/rust-lang/rust/pull/51657))
	* 生成有关 rustc 内部性能的详细信息 
	* 由 [@wesleywiser](https://github.com/wesleywiser)  和  [@michaelwoerister](https://github.com/michaelwoerister) 开发

*  [Shared monomorphizations](https://github.com/rust-lang/rust/issues/47317) 
	* Reduces code bloat by deduplicating monomorphizations that occur in multiple crates
	* Enabled by default if the optimization level is less than 3.
	* Developed by  [@michaelwoerister](https://github.com/michaelwoerister) 

*  [共享单态化（Shared monomorphizations）](https://github.com/rust-lang/rust/issues/47317) 
	* 通过消除多个包（crate）中出现的单态化来减少代码膨胀
	* 如果优化级别小于 3，则默认启用
	* 由[@michaelwoerister](https://github.com/michaelwoerister) 开发

*  [Cranelift backend](https://www.reddit.com/r/rust/comments/enxgwh/cranelift_backend_for_rust/) 
	* Reduced debug compile times by used  [cranelift](https://github.com/bytecodealliance/cranelift)  for code generation.
	* Developed by  [@bjorn3](https://github.com/bjorn3) 

*  [Cranelift 后端](https://www.reddit.com/r/rust/comments/enxgwh/cranelift_backend_for_rust/) 
	* 通过使用 [cranelift](https://github.com/bytecodealliance/cranelift) 来生成代码，减少了 Debug 模式的编译时间。
	* 由 [@bjorn3](https://github.com/bjorn3) 开发

*  [perf.rust-lang.org](https://perf.rust-lang.org/) 
	* Rust’s compile-time performance is tracked in detail. Benchmarks continue to be added.
	* Developed by [@nrc](https://github.com/nrc) ,  [@Mark-Simulacrum](https://github.com/Mark-Simulacrum) ,  [@nnethercote](https://github.com/nnethercote)  and many more

*  [perf.rust-lang.org](https://perf.rust-lang.org/) 
	* 详细跟踪了 Rust 的编译时性能，基准测试持续增加中
	* 由 [@nrc](https://github.com/nrc) ,  [@Mark-Simulacrum](https://github.com/Mark-Simulacrum) ,  [@nnethercote](https://github.com/nnethercote) 以及其他人一起开发

*  [cargo-bloat](https://github.com/RazrFalcon/cargo-bloat) 
	* Finds what occupies the most space in binaries. Bloat is correlated with compile time
	* Developed by  [@RazrFalcon](https://github.com/RazrFalcon)  and others

*  [cargo-bloat](https://github.com/RazrFalcon/cargo-bloat) 
	* 查找二进制文件中占用最多空间的地方。膨胀（Bloat）会影响编译时间
	* 由 [@RazrFalcon](https://github.com/RazrFalcon) 和其他人一起开发

*  [cargo-feature-analyst](https://github.com/psinghal20/cargo-feature-analyst) 
	* Finds unused features
	* Developed by  [@psinghal20](https://github.com/psinghal20) 

*  [cargo-feature-analyst](https://github.com/psinghal20/cargo-feature-analyst) 
	* 发现未使用的特性（features）
	* 由 [@psinghal20](https://github.com/psinghal20) 开发

*  [cargo-udeps](https://github.com/est31/cargo-udeps) 
	* Finds unused crates
	* Developed by  [@est31](https://github.com/est31) 

*  [cargo-udeps](https://github.com/est31/cargo-udeps) 
	* 发现未使用的包 （crate）
	* 由 [@est31](https://github.com/est31) 开发

*  [twiggy](https://github.com/rustwasm/twiggy) 
	* Profiles code size, which is correlated with compile time
	* Developed by  [@fitzgen](https://github.com/fitzgen) ,  [@data-pup](https://github.com/data-pup) , and others

*  [twiggy](https://github.com/rustwasm/twiggy) 
	* 分析代码大小，该大小与编译时间相关
	* 由 [@fitzgen](https://github.com/fitzgen) ,  [@data-pup](https://github.com/data-pup) 以及其他人一起开发

*  [rust-analyzer](https://github.com/rust-analyzer/rust-analyzer) 
	* A new language server for Rust with faster response time than the original  [RLS](https://github.com/rust-lang/rls) 
	* Developed by  [@matklad](https://github.com/matklad) ,  [@flodiebold](https://github.com/flodiebold) ,  [@kjeremy](https://github.com/kjeremy) , and many others

*  [rust-analyzer](https://github.com/rust-analyzer/rust-analyzer) 
	* 用于Rust的新语言服务器，其响应时间比原始  [RLS](https://github.com/rust-lang/rls) 更快
	* 由 [@matklad](https://github.com/matklad) ,  [@flodiebold](https://github.com/flodiebold) ,  [@kjeremy](https://github.com/kjeremy) 以及其他人一起开发

*  [“How to alleviate the pain of Rust compile times”](https://vfoley.xyz/rust-compile-speed-tips/) 
	* Blog post by vfoley

*  [“如何缓解 Rust 编译时间带来的痛苦”](https://vfoley.xyz/rust-compile-speed-tips/)
	* vfoley 写的博文

*  [“Thoughts on Rust bloat”](https://raphlinus.github.io/rust/2019/08/21/rust-bloat.html) 
	* Blog post by  [@raphlinus](https://github.com/raphlinus) 
	
*  [“关于 Rust 代码膨胀的思考”](https://raphlinus.github.io/rust/2019/08/21/rust-bloat.html) 
	* [@raphlinus](https://github.com/raphlinus) 写的博文

* Nicholas Nethercote’s work on rustc optimization
	*  [“How to speed up the Rust compiler in 2019”](https://blog.mozilla.org/nnethercote/2019/07/17/how-to-speed-up-the-rust-compiler-in-2019/) 
	*  [“The Rust compiler is still getting faster”](https://blog.mozilla.org/nnethercote/2019/07/25/the-rust-compiler-is-still-getting-faster/) 
	*  [“Visualizing Rust compilation”](https://blog.mozilla.org/nnethercote/2019/10/10/visualizing-rust-compilation/) 
	*  [“How to speed up the Rust compiler some more in 2019”](https://blog.mozilla.org/nnethercote/2019/10/11/how-to-speed-up-the-rust-compiler-some-more-in-2019/) 
	*  [“How to speed up the Rust compiler one last time in 2019”](https://blog.mozilla.org/nnethercote/2019/12/11/how-to-speed-up-the-rust-compiler-one-last-time-in-2019/) 

* Nicholas Nethercote 对 rustc 的优化工作
	*  [“2019 年 Rust 编译器如何提速”](https://blog.mozilla.org/nnethercote/2019/07/17/how-to-speed-up-the-rust-compiler-in-2019/) 
	*  [“Rust 编译器的速度持续变快”](https://blog.mozilla.org/nnethercote/2019/07/25/the-rust-compiler-is-still-getting-faster/) 
	*  [“可视化 Rust 编译”](https://blog.mozilla.org/nnethercote/2019/10/10/visualizing-rust-compilation/) 
	*  [“如何在 2019 年进一步提升 Rust 编译器的速度”](https://blog.mozilla.org/nnethercote/2019/10/11/how-to-speed-up-the-rust-compiler-some-more-in-2019/)
	*  [“如何在 2019 年最后一次提升 Rust 编译器”](https://blog.mozilla.org/nnethercote/2019/12/11/how-to-speed-up-the-rust-compiler-one-last-time-in-2019/) 

I apologize to any person or project I didn’t credit.

对于未上榜的人员或项目，我需要说一声抱歉。

## 下集预告

So Rust dug itself deep into a corner over the years and will probably be digging itself back out until the end of time (or the end of Rust — same thing, really). Can Rust compile-time be saved from Rust’s own run-time success? Will TiKV ever build fast enough to satisfy my managers?

所以多年来，Rust 把自己深深地逼进了一个死角，而且很可能会持续逼进，直到玩完。Rust 的编译时能否从 Rust 自身的运行时成功中得到拯救？TiKV 的构建速度能否让我的管理者满意吗？

In the next episode, we’ll deep-dive into the specifics of Rust’s language design that cause it to compile slowly.

在下一集中，我们将深入讨论 Rust 语言设计的细节，这些细节会导致它编译缓慢。

Stay Rusty, friends.

继续享受 Rust 吧，朋友们！

## 鸣谢

A number of people helped with this blog series. Thanks especially to Niko Matsakis, Graydon Hoare, and Ted Mielczarek for their insights, and Calvin Weng for proofreading and editing.

很多人参与了本系列博客。特别感谢 Niko Matsakis、Graydon Hoare 和 Ted Mielczarek 的真知卓见，以及 Calvin Weng 的校对和编辑。

## 关于作者

 [Brian Anderson](https://github.com/brson)  is one of the co-founders of the Rust programming language and its sister project, the Servo web browser. He is now working in PingCAP as a senior database engineer.

 [Brian Anderson](https://github.com/brson) 是 Rust 编程语言及其姊妹项目 Servo Web 浏览器的共同创始人之一。 他现在在 PingCAP 担任高级数据库工程师。
