---
title: go程序的启动引导
date: 2018-03-29 16:05:50
tags: go
---

go程序是如何启动的？

测试环境：
```
$ uname -a
Linux zxl 4.9.0-deepin13-amd64 #1 SMP PREEMPT Deepin 4.9.57-1 (2017-10-19) x86_64 GNU/Linux
$ go version
go version go1.10 linux/amd64
```
我们来写一个程序hello.go：
```
  package main
  func main(){
    println("Hello,World!")
  }
```
使用`go build -gcflags "-N -l" -o hello hello.go`编译，`-gcflags "-N -l"`关闭编译器代码优化和内联
运行`$ gdb ./hello`
```
(gdb) info files
Symbols from "/data/Study/go/hello".
Local exec file:
	`/data/Study/go/hello', file type elf64-x86-64.
	Entry point: 0x448ef0
(gdb) b *0x448ef0
Breakpoint 1 at 0x448ef0: file /usr/local/go/src/runtime/rt0_linux_amd64.s, line 8.
(gdb) b runtime.rt0_go
Breakpoint 2 at 0x445730: file /usr/local/go/src/runtime/asm_amd64.s, line 89.
(gdb) b runtime.args
Breakpoint 3 at 0x42f970: file /usr/local/go/src/runtime/runtime1.go, line 60.
(gdb) b runtime.osinit
Breakpoint 4 at 0x41ff80: file /usr/local/go/src/runtime/os_linux.go, line 272.
(gdb) b runtime.schedinit
Breakpoint 5 at 0x4245d0: file /usr/local/go/src/runtime/proc.go, line 472.
(gdb) b runtime.newproc
Breakpoint 6 at 0x42aec0: file /usr/local/go/src/runtime/proc.go, line 3235.
(gdb) b runtime.mstart
Breakpoint 7 at 0x426290: file /usr/local/go/src/runtime/proc.go, line 1170.
(gdb) b runtime.main
Breakpoint 8 at 0x423420: file /usr/local/go/src/runtime/proc.go, line 109
```
可以程序开始于`/usr/local/go/src/runtime/rt0_linux_amd64.s:8`,即`rt0_arm64_linux`函数， 源码包runtime下放可许多rt0_sys_arch.s文件
```
TEXT _rt0_arm64_linux(SB),NOSPLIT,$-8
	MOVD	0(RSP), R0	// argc
	ADD	$8, RSP, R1	// argv
	BL	main(SB)
```
来看`main`
```
TEXT main(SB),NOSPLIT,$-8
	MOVD	$runtime·rt0_go(SB), R2
	BL	(R2)
exit:
	MOVD $0, R0
	MOVD	$94, R8	// sys_exit
	SVC
	B	exit
```
源码中的`·`编译完之后会变成`.`
断点`b runtime.rt0_go`,定位到`/usr/local/go/src/runtime/asm_amd64.s, line 89`
```
TEXT runtime·rt0_go(SB),NOSPLIT,$0
  ...
  CALL	runtime·args(SB)//命令行参数初始化
  CALL	runtime·osinit(SB)// 系统初始化
  CALL	runtime·schedinit(SB)//调度器初始化

  // create a new goroutine to start program
  MOVQ	$runtime·mainPC(SB), AX		// entry
  PUSHQ	AX
  PUSHQ	$0			// arg size
  CALL	runtime·newproc(SB)//创建一个goroutine
  POPQ	AX
  POPQ	AX

  // start this M
  CALL	runtime·mstart(SB)
  MOVL	$0xf1, 0xf1  // crash
  RET
DATA	runtime·mainPC+0(SB)/8,$runtime·main(SB)
GLOBL	runtime·mainPC(SB),RODATA,$8
```
由此，汇编引导过程完成

断点`runtime.args`,在`/usr/local/go/src/runtime/runtime1.go, line 60`，初始化参数列表
```
func args(c int32, v **byte) {
	argc = c
	argv = v
	sysargs(c, v)
}
```

断点`runtime.osinit`,在`/usr/local/go/src/runtime/os_linux.go, line 272`，获取CPU核数
```
  func osinit() {
      ncpu = getproccount()
  }
```

断点`runtime.schedinit`,在`/usr/local/go/src/runtime/proc.go, line 472`，调度器初始化
```
// The bootstrap sequence is:
//	call osinit
//	call schedinit
//	make & queue new G
//	call runtime·mstart
// The new G calls runtime·main.
func schedinit() {
  // raceinit must be the first call to race detector.
  // In particular, it must be done before mallocinit below calls racemapshadow.
  _g_ := getg()
    if raceenabled {
    _g_.racectx, raceprocctx0 = raceinit()
  }
  sched.maxmcount = 10000 //最大系统线程数限制　　runtime/debug.SetMaxThreads
  //初始化栈、内存分配器，调度器相关
  tracebackinit()
  moduledataverify()
  stackinit()
  mallocinit()
  mcommoninit(_g_.m)
  alginit()       // maps must not be used before this call
  modulesinit()   // provides activeModules
  typelinksinit() // uses maps, activeModules
  itabsinit()     // uses activeModules
  msigsave(_g_.m)
  initSigmask = _g_.m.sigmask
  //命令行参数与环境变量初始化
  goargs()
  goenvs()
  //处理GODEBUG,GOTRACEBACK调试相关的环境变量
  parsedebugvars()
  //垃圾回收器的初始化
  gcinit()
  //通过CPU Core和GOMAXPROCS环境变量确定P的数量
  sched.lastpoll = uint64(nanotime())
  procs := ncpu
  if n, ok := atoi32(gogetenv("GOMAXPROCS")); ok && n > 0 {
    procs = n
  }
  // 调整P的数量
  if procresize(procs) != nil {
    throw("unknown runnable goroutine during bootstrap")
  }
  // For cgocheck > 1, we turn on the write barrier at all times
  // and check all pointer writes. We can't do this until after
  // procresize because the write barrier needs a P.
  if debug.cgocheck > 1 {
    writeBarrier.cgo = true
    writeBarrier.enabled = true
    for _, p := range allp {
      p.wbBuf.reset()
    }
  }
  if buildVersion == "" {
    // Condition should never trigger. This code just serves
    // to ensure runtime·buildVersion is kept in the resulting binary.
    buildVersion = "unknown"
  }
}
```
创建一个goroutine(G)，断点`runtime.newproc`,在`/usr/local/go/src/runtime/proc.go, line 3235`//G就是goroutine实现的核心结构了，G维护了goroutine需要的栈、程序计数器以及它所在的M等信息。
启动一个内核级线程(M),一个开始执行程序，断点`runtime.mstart`,在`/usr/local/go/src/runtime/proc.go, line 1170` //M代表内核级线程，一个M就是一个线程，goroutine就是跑在M之上的；M是一个很大的结构，里面维护小对象内存cache（mcache）、当前执行的goroutine、随机数发生器等等非常多的信息。

关于调度器是如何调度的,请查看[goroutine与调度器](https://studygolang.com/articles/1855)其形象的描述了go调度器的工作方式

而`runtime.main` 即　`/usr/local/go/src/runtime/proc.go`中的`mian`
```
Breakpoint 8, runtime.main () at /usr/local/go/src/runtime/proc.go:109
109	func main() {
(gdb) bt
#0  runtime.main () at /usr/local/go/src/runtime/proc.go:109
#1  0x0000000000447fc1 in runtime.goexit () at /usr/local/go/src/runtime/asm_amd64.s:2361
#2  0x0000000000000000 in ?? ()
(gdb)
```
```
func main() {
  ...
  // Max stack size is 1 GB on 64-bit, 250 MB on 32-bit.Using decimal instead of binary GB and MB because they look nicer in the stack overflow failure message.
  if sys.PtrSize == 8 {
    maxstacksize = 1000000000
  } else {
    maxstacksize = 250000000
  }
  // Allow newproc to start new Ms.
  mainStarted = true
  //启动系统后台监控（定期垃圾回收，并发调度）
  systemstack(func() {
    newm(sysmon, nil)
  })

  //在初始化过程中,上主线程锁,大多数程序并不关心，但少数确实需要由主线程发出的某些调用，可以在初始化过程中的main_main调用runtime.lockosthread来保存锁。
  lockOSThread()
  ...
  //执行runtime包里的所有init函数　must be before defer
  runtime_init()
  ...
  // Defer unlock so that runtime.Goexit during init does the unlock too.
  needUnlock := true
  defer func() {
    if needUnlock {
      unlockOSThread()
    }
  }()
  //启动垃圾回收器后台操作
  gcenable()
  ...
  //
  fn := main_init //调用所有用户程序涉及包的init函数
  fn()
  //解锁
  needUnlock = false
  unlockOSThread()
  ...
  fn = main_main // 执行用户逻辑入口
  fn()
}
```
至此，一个go程序的启动逻辑完成
