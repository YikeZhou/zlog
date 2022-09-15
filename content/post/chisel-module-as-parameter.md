---
title: "将 Chisel Module 作为类参数"
date: 2021-08-22T12:00:00+08:00
lastmod: 2022-09-15T18:31:49+08:00
draft: false
keywords: ["Chisel", "Scala"]
description: "在 Chisel 里使用 `Module` 作为生成另一个 `Module` 的参数"
tags: ["Chisel", "Scala"]
categories: ["编程"]
author: "Yike Zhou"

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: false
toc: true
autoCollapseToc: true
postMetaInFooter: false
hiddenFromHomePage: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: false
reward: false
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false

# You unlisted posts you might want not want the header or footer to show
hideHeaderAndFooter: false

# You can enable or disable out-of-date content warning for individual post.
# Comment this out to use the global config.
#enableOutdatedInfoWarning: false

flowchartDiagrams:
  enable: false
  options: ""

sequenceDiagrams:
  enable: false
  options: ""

---

<!--more-->
## 使用方法

现在我们的需求是：在同一个 Generator 里可以使用端口定义相同、内部实现不同的子模块，且子模块可以通过参数进行配置。

首先定义一个特质 `MyAdder`，它可以理解为 `Mod1` 和 `Mod2` 两个 `Module` 的接口定义。
`Mod1` 和 `Mod2` 是两个接口相同、功能不同的模块。

```scala
// Provides a more specific interface since generic Module
// provides no compile-time information on generic module's IOs.
trait MyAdder {
    def in1: UInt
    def in2: UInt
    def out: UInt
}

class Mod1 extends RawModule with MyAdder {
    val in1 = IO(Input(UInt(8.W)))
    val in2 = IO(Input(UInt(8.W)))
    val out = IO(Output(UInt(8.W)))
    out := in1 + in2
}

class Mod2 extends RawModule with MyAdder {
    val in1 = IO(Input(UInt(8.W)))
    val in2 = IO(Input(UInt(8.W)))
    val out = IO(Output(UInt(8.W)))
    out := in1 - in2
}
```

类 `X` 接收一个实现了 `MyAdder` 特质的对象作为参数。`: =>` 表示参数会延迟求值。

```scala
class X[T <: BaseModule with MyAdder](genT: => T) extends Module {
    val io = IO(new Bundle {
        val in1 = Input(UInt(8.W))
        val in2 = Input(UInt(8.W))
        val out = Output(UInt(8.W))
    })
    val subMod = Module(genT)
    io.out := subMod.out
    subMod.in1 := io.in1
    subMod.in2 := io.in2
}
```

生成对应 Verilog 代码，并进行比较。

```scala
println(ChiselStage.emitVerilog(new X(new Mod1)))
println(ChiselStage.emitVerilog(new X(new Mod2)))
```

当类参数 `genT = Mod1` 时，生成了 `X` 和 `Mod1` 两个模块，并且在 `X` 中实例化了 `Mod1` 模块。

```verilog
module Mod1(
  input  [7:0] in1,
  input  [7:0] in2,
  output [7:0] out
);
  assign out = in1 + in2; // @[polymorphism-and-parameterization.md 174:16]
endmodule

module X(
  input        clock,
  input        reset,
  input  [7:0] io_in1,
  input  [7:0] io_in2,
  output [7:0] io_out
);
  wire [7:0] subMod_in1; // @[polymorphism-and-parameterization.md 192:24]
  wire [7:0] subMod_in2; // @[polymorphism-and-parameterization.md 192:24]
  wire [7:0] subMod_out; // @[polymorphism-and-parameterization.md 192:24]
  Mod1 subMod ( // @[polymorphism-and-parameterization.md 192:24]
    .in1(subMod_in1),
    .in2(subMod_in2),
    .out(subMod_out)
  );
  assign io_out = subMod_out; // @[polymorphism-and-parameterization.md 193:12]
  assign subMod_in1 = io_in1; // @[polymorphism-and-parameterization.md 194:16]
  assign subMod_in2 = io_in2; // @[polymorphism-and-parameterization.md 195:16]
endmodule
```

当类参数 `genT = Mod2` 时，`X` 中的子模块变成了 `Mod2`（此处省略了重复的代码）。

```verilog
module Mod2(
  input  [7:0] in1,
  input  [7:0] in2,
  output [7:0] out
);
  assign out = in1 - in2; // @[polymorphism-and-parameterization.md 182:16]
endmodule

module X(/*...*/);
  /*...*/
  Mod2 subMod ( // @[polymorphism-and-parameterization.md 192:24]
    .in1(subMod_in1),
    .in2(subMod_in2),
    .out(subMod_out)
  );
  /*...*/
endmodule
```

{{% admonition note "更多" false %}}
- `RawModule` / `BaseModule` / `Module` 中：`RawModule` 没有隐式 `clock` 和 `reset` 端口；`BaseModule` 是其它 `Module` 的**抽象基类**（abstract base class）；`Module` 是定义一个模块时通常会使用的类。
- `MultiIOModule` 比起 `RawModule` 多了隐式 `clock` 和 `reset` 端口。
{{% /admonition %}}

## 参考资料

- Chisel/FIRRTL Doc: [Parametrization based on Modules](https://www.chisel-lang.org/chisel3/docs/explanations/polymorphism-and-parameterization.html#parametrization-based-on-modules)
- Module.scala: https://github.com/chipsalliance/chisel3/blob/v3.4.3/core/src/main/scala/chisel3/Module.scala
- RawModule.scala: https://github.com/chipsalliance/chisel3/blob/v3.4.3/core/src/main/scala/chisel3/RawModule.scala