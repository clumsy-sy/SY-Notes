# Falcor ReSTIR NRC 复现

## Falcor 流程

`fork` 项目 `git clone`，在项目文件的根目录下有 `setup.bat`，运行该批处理文件，会 clone 需要的子模块，运行 `setup_vs2022.bat` 可以生成解决方案，用 vs 打开 `build/windows-vs2022/Falcor.sln`，将想要的想要执行的程序右键设置为启动项即可，比如 `Mogwai`。


## RenderGraph & RenderPass

### Render Graph

**Render Graph** 是一个**有向无环图（DAG）**，vertex 是 Render Pass，edge 是 resource 的流动。它是对整个渲染流程进行一个更高层次的抽象。

**作用**
1. 解耦，也可以说是将原先固化的流程给模块化，变成可配置的。
2. 自动的资源管理，管理着一个资源池，所有虚拟资源所构建的实际资源在使用完之后，都会被放回资源池中。
3. 简化的多线程渲染、容易 debug、pass 复用

**三个阶段**

* setup : pass声明注册自己用到的资源，需要确定资源的状态(未实际分配GPU资源),将持久资源导入内存。
* compile ：对resource的声明周期进行计算，确定resource应该在何时分配资源、何时被回收。
* execute ：各个pass开始执行自己的Execute函数。

在 Falcor 中，大致流程，初始化，解析 py 文件，确定输入输出，添加 pass，addEdge，Compile，Execute。值得一提的是 编译会遍历整个 DAG ，确定需要执行的 Pass，生成 ExecuteList。

### Render Pass

**Render Pass** 是 Render Graph 中的一个节点

我们可以狭义地将 pass 理解为一个函数，它可以接受一个或多个输入，并产生一个或多个输出。这里的输入与输出只能是resource节点。pass的在拿到resource之后，调用自己内部的执行函数，执行函数一般是调用一次或若干次GPU，产生一个或多个结果，并将这些结果输出。
