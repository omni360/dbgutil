项目介绍
========

这是一个模仿hopwatch做的Go语言命令行程序调试工具，原理很简单，就是打印变量和按条件暂停程序。

最早是真有趣团队中的同事做了一个简单的变量打印函数，可以支持嵌套结构的打印。最近看到hopwatch项目后，想到何不把暂停和条件暂停也集成进去呢？于是便有了现在这个项目。

为什么不用hopwatch呢？因为hopwatch要通过页面显示和操作，略繁琐了，Go是针对服务端应用设计的编程语言，所做的大多数都是命令行下运行的程序，只要能通过命令行操作和输出就够用了。

这个调试工具的特色有：

1. 打印变量的时候支持结构体的嵌套
2. 支持指针引用关系的打印
3. 内置了一个堆栈打印函数，内部做了忽略Go语言运行时库堆栈跟踪，可以让程序错误日志内容更清晰

这些代码原理虽然简单，自己写起来还是得花不少力气调试的，现在现成的工具就在眼前，赶紧下载使用吧！

用法说明
========

首先在需要调试的代码中使用以下方式引用调试模块：

	import "github.com/realint/dbgutil"

如果不巧你的项目里已经有一个叫debug的模块，没关系，引用时候重命名就可以：

	import dbg "github.com/realint/dbgutil"

然后在需要打印变量值的位置用以下方式打印变量：

	dbgutil.Display("a", a, "b", b, "c", c)

格式是一个字符串变量名 + 变量值，必须是成对出现，可以任意多对。

如果需要暂停程序（类似断点），请用以下代码：

	dbgutil.Break()

当程序运行到这句代码时会等待命令行输入回车，回车后程序就会继续运行了。

有些时候需要达到特定条件才暂停程序，请用以下代码：

	dbgutil.Display("a", a, "b", b, "c", c).Break(a == 0)

当a == 0的时候，程序就会暂停，等待命令行输入回车。

打印堆栈的函数是Stack，其中的参数是表示要忽略几个帧，因为通常调用栈打印的地方不是真正错误出现的地方，一般都是在外层有一个错误处理的包装，所以可以通过这个参数忽略掉不必要的堆栈跟踪信息，让错误日志更清晰。

效果演示
=======

下面演示以下牛逼的指针引用打印。

假设程序中有几个变量，他们之间通过指针互相引用，关系如下：

	type mytype struct {
		next *mytype
		prev *mytype
	}

	var v1 = new(mytype)
	var v2 = new(mytype)
	var v3 = new(mytype)

	v1.prev = v3
	v1.next = v2

	v2.prev = v1
	v2.next = v3

	v3.prev = v2
	v3.next = nil

当调用变量打印的时候：

	dbgutil.Display("value", v1).Break(true)

将输出以下牛逼的结果：

	2013/07/04 00:29:32 [Debug] xxx/api/api.go:54

	[Variables]
	value = &api.mytype{ next: &api.mytype{ next: &api.mytype{ next: nil, prev: & }, prev: & }, prev: & }
	        |                  |                  |                             |          |          |
	        |                  |                  |-----------------------------+----------+----------|
	        |                  |------------------------------------------------|          |
	        |------------------------------------------------------------------------------|


	[Stack]
	xxx/api/api.go:54
	    debug.Display("value", v1).Break(true)
	xxx/srv/server/main.go:96
	    err := api.Start(config.Server.Id, "0.0.0.0:"+config.Server.GamePort)

	press ENTER to continue

吓死人了，怎么那么强大？