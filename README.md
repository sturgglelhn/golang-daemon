# golang-daemon

**需求：使用 go 发布简易客户端，能够安装在windows与linux，定时采集进程信息并写入文件**

##### 一、golang程序编写

1、下载第三方包`github.com/kardianos/service` 和 `github.com/shirou/gopsutil/process`

````go
go get github.com/kardianos/service		//主要用于windows和linux作为服务运行的应用程序
go get github.com/shirou/gopsutil/process	//主要用于获取进程信息
````

2、代码编写，这里直接粘贴上代码，是我已经能运行成功，并且经过反复测试的代码

`````go
package main

import (
	"fmt"
	"github.com/kardianos/service"
	"github.com/shirou/gopsutil/process"
	"os"
	"time"
)

func main() {
	srvConfig := &service.Config{
		Name:        "MyUserExperienceService",
		DisplayName: "MyUserExperienceService数据提取",
		Description: "MyUserExperience数据提取服务",
	}
	prg := &program{}
	s, err := service.New(prg, srvConfig)
	if err != nil {
		fmt.Println(err)
	}
	if len(os.Args) > 1 {
		serviceAction := os.Args[1]
		switch serviceAction {
		case "install":
			err := s.Install()
			if err != nil {
				fmt.Println("安装服务失败: ", err.Error())
			} else {
				fmt.Println("安装服务成功")
			}
			return
		case "uninstall":
			err := s.Uninstall()
			if err != nil {
				fmt.Println("卸载服务失败: ", err.Error())
			} else {
				fmt.Println("卸载服务成功")
			}
			return
		case "start":
			err := s.Start()
			if err != nil {
				fmt.Println("运行服务失败: ", err.Error())
			} else {
				fmt.Println("运行服务成功")
			}
			return
		case "stop":
			err := s.Stop()
			if err != nil {
				fmt.Println("停止服务失败: ", err.Error())
			} else {
				fmt.Println("停止服务成功")
			}
			return
		}
	}

	err = s.Run()
	if err != nil {
		fmt.Println(err)
	}
}

type program struct{}

func (p *program) Start(s service.Service) error {
	fmt.Println("服务运行...")
	go p.run()
	return nil
}
func (p *program) run() {
	// 具体的服务实现
	t := time.Now()
	fileName := fmt.Sprintf("process-info-%s.txt", t.Format("2006-01-02-15-04-05"))
	file, err := os.Create(fileName)
	//file, err := os.OpenFile(fileName, os.O_CREATE|os.O_WRONLY, os.ModePerm)
	if err != nil {
		fmt.Println("创建文件失败：", err)
		return
	}
	defer file.Close()

	tick := time.Tick(3 * time.Second)

	for {
		select {
		case <-tick:
			now := t.Format("2006-01-02 15:04:05")
			processes, _ := process.Processes()
			for _, p := range processes {
				name, _ := p.Name()
				pid := p.Pid
				memory, _ := p.MemoryInfo()
				cpuPercent, _ := p.CPUPercent()
				//fmt.Printf("Process name: %s,PID:%d,Memory: Usage:%v,CPU uage:%f\n", name, pid, memory, cpuPercent)
				_, err := file.WriteString(fmt.Sprintf("time：%s,Process name: %s,PID:%d,Memory: Usage:%v,CPU uage:%f\n", now, name, pid, memory, cpuPercent))
				if err != nil {
					fmt.Println("写入信息失败：", err)
					return
				}
			}
			fmt.Printf("%s 写入成功 \n", now)
		}
	}
}
func (p *program) Stop(s service.Service) error {
	return nil
}
`````



##### 二、在windows中注册服务

1、在windows中编译成可执行文件，也就.exe文件

`````
go env		//查看go环境的相关信息
- set GOHOSTOS=windows	//发现是windows就可以了

go build 02_windows_daemon.go	//生成可执行文件
`````

<img src="C:\Users\haonanLI\AppData\Roaming\Typora\typora-user-images\image-20230418164338847.png" alt="image-20230418164338847" style="zoom: 150%;" />

2、安装可执行文件即可：02_windows_daemon install，一定要用管理员身份远程cmd，然后切换到文件目录运行此条命令，此时我们会发现windows服务中有你注册的对应名称：

<img src="C:\Users\haonanLI\AppData\Roaming\Typora\typora-user-images\image-20230418165614687.png" alt="image-20230418165614687" style="zoom:80%;" />

- 启动就可以了，程序会生成一个.txt的文件夹，在C:\Windows\System32中可以看到

<img src="C:\Users\haonanLI\AppData\Roaming\Typora\typora-user-images\image-20230418165828128.png" alt="image-20230418165828128" style="zoom: 80%;" />

自此windows中注册服务完成。





##### 二、Linux中把程序注册成服务

1、在windows中更改go的环境，生成可执行文件

````
go env
set GOOS=linux
go build 02_windows_daemon.go	//生成linux文件
````

把生成的文件上传到linux目录中，我自己上传在这里/root/golang/02_windows_daemon

![image-20230418170438672](C:\Users\haonanLI\AppData\Roaming\Typora\typora-user-images\image-20230418170438672.png)



2、systemd介绍

systemd 是 Linux 系统中的一个全新的初始化系统和系统管理器，它负责启动和管理系统所有进程，提供了许多功能：如系统引导、进程守护、挂载文件系统、计划任务、用户登录等。同时，systemd 也充当系统服务的管理器，它能够让管理员在配置和管理网络、设备和其他服务时更加方便。

在 `/etc/systemd/system` 目录下，系统管理员可以创建自定义的 systemd 服务配置文件，用以控制各种系统服务的启动、停止、重启、维护等行为。例如，管理员可以创建 `demo01.service` 文件来定义自己的服务，然后通过 `systemctl status demo01.service` 命令将其注册到 systemd 中，使其能够在系统启动时自动启动。

***常用命令：***

`````linux
// 启动服务
systemctl start demo01.service

// 查看服务状态
systemctl status demo01.service

// 停止服务
systemctl stop demo01.service

// 重启服务
systemctl restart <service>

// 重新加载服务配置文件
systemctl reload <service>

// 设置服务为开机自启动
systemctl reable <service>

// 关闭服务的开启自启动
systemctl disable <service>

// 列出所有 systemd 单位的状态信息
systemctl list-units

// 列出所有可用的 systemd 单位文件
systemctl list-unit-files
`````



3、需要在/etc/systemd/system目录下创建.server文件，这里以demo01.server为例：

`touch demo01.server		//创建配置系统服务文件`

`````
[Unit]			//指定服务的基本信息
Description=Hello World Service		//服务的名称
After=network.target				//表示服务在网络启动之后启动

[Service]		//定义服务的具体行为
ExecStart=/root/golang/02_windows_daemon	//表示服务启动时需要执行的命令或者路径
WorkingDirectory=/root/golang/info/			//设置程序执行的工作目录，输出到该目录
Restart=always	//表示服务异常退出后会自动重启
User=root		//表示该服务以root权限运行

[Install]		//指定如何安装这个服务
WantedBy=multi-user.target	//表示系统进入多用户模式时，这个服务将被安装并启动
`````



4、启动服务

`systemctl start demo01.service 		//启动服务`

`systemctl status demo01.service 		//查看服务`

自此我们就成功在linux系统中把golang注册成为了服务