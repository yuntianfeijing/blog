---
title: go os/exec
date: 2018-03-29 13:55:26
tags: go
---
## go的标准包os/exec中包含了如下一些方法及结构

os/exec定义了如下结构：
```
type Cmd struct {
	Path string
	Args []string
	Env []string
	Dir string
	Stdin io.Reader
	Stdout io.Writer
	Stderr io.Writer
	ExtraFiles []*os.File
	SysProcAttr *syscall.SysProcAttr
	Process *os.Process
	ProcessState *os.ProcessState
	ctx             context.Context // nil means none
	lookPathErr     error           // LookPath error, if any.
	finished        bool            // when Wait was called
	childFiles      []*os.File
	closeAfterStart []io.Closer
	closeAfterWait  []io.Closer
	goroutine       []func() error
	errch           chan error // one send per goroutine
	waitDone        chan struct{}
}
```
exec.Command 返回用于执行name代表的命令的Cmd结构，Cmd中设置了Path和Args
`func Command(name string, arg ...string) *Cmd`

_先上代码：_
```
     import (
        "os/exec"
        "bytes"
        "log"
        "fmt"
      )
      func main() {
        cmd := exec.Command("ls","/home")
        var out bytes.Buffer
        cmd.Stdout = &out
        err := cmd.Start()
        if err != nil {
          log.Fatal(err)
        }
        log.Printf("Waiting for command to finish...")
        fmt.Println(cmd.Args)
        err = cmd.Wait()
        if err != nil {
          log.Printf("Command finished with error: %v", err)
        }
        fmt.Println(out.String())
        return out.String(),nil
      }
```
_着重说一下使用cmd.Start开始命令，不要忘记cmd.Wait，否则会产生僵尸进程，cmd.Wait　返回命令的返回值_


os/exec 包里的其他方法：
_在系统PATH中检索cmd，并返回其路径_
`func LookPath(file string) (string, error)`
_执行命令,并捕获命令输出_
`func (c *Cmd) Output() ([]byte, error)`
_运行一个指定的命令c，但不等待其完成_
`func (c *Cmd) Start() error`
_Wait等待命令退出，命令必须是由Start方法运行的_
`func (c *Cmd) Wait() error`
_运行指定的命令，并等待其完成_
```
func (c *Cmd) Run() error {
	if err := c.Start(); err != nil {
		return err
	}
	return c.Wait()
}
```
exec调用外部程序，参数不支持统配符、管道等
