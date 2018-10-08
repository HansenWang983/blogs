---
title: "Service Computing 实验三：golang下CLI命令行实用程序开发基础"
date: 2018-10-08T11:38:52+08:00
lastmod: 2018-10-08T11:41:52+08:00
menu: "main"
weight: 50
author: "hansenbeast"
tags: [
    "Service Computing"
]
categories: [
    "Tech Blogs"
]
# you can close something for this content if you open it in config.toml.
comment: false
mathjax: false
---



# CLI 命令行实用程序开发基础

使用 golang  [开发 Linux 命令行实用程序](https://www.ibm.com/developerworks/cn/linux/shell/clutil/index.html) 中的 **selpg**

> selpg 允许用户指定从输入文本抽取的页的范围，这些输入文本可以来自文件或另一个进程。该实用程序从标准输入或从作为命令行参数给出的文件名读取文本输入。它允许用户指定来自该输入并随后将被输出的页面范围。

## 1、命令行准则：

1. 输入：

   ```bash
   $ command #终端（也就是用户的键盘）
   $ command input_file #command 应该读取文件 input_file
   $ command < input_file  #标准输入重定向为来自文件
   $ other_command | command #标准输入来自另一个程序的标准输出
   ```

2. 输出：

   ```bash
   $ command #标准输出同样也是终端（也就是用户的屏幕）
   $ command > output_file #标准输出重定向至文件
   $ command | other_command #command 的输出可以成为另一个程序的标准输入
   ```

3. 错误输出

   ```bash
   $ command #标准错误stderr同样也是终端（也就是用户的屏幕）
   $ command 2>error_file #错误重定向至文件（2是为了区别标准输出stdout）
   $ command >output_file 2>error_file #标准输出和标准错误都重定向至不同的文件
   $ command >output_file 2>&1 #标准输出和标准错误都重定向至同一的文件、
   ```

4. 命令行参数

   ```bash
   $ command mandatory_opts [ optional_opts ] [ other_args ] 
   # command 是命令本身的名称。
   # mandatory_opts 是为使命令正常工作必须出现的选项列表。
   # optional_opts 是可指定也可不指定的选项列表，这由用户来选择；但是，其中一些参数可能是互斥的，如同 selpg 的“-f”和“-l”选项的情况（详情见下文）。
   # other_args 是命令要处理的其它参数的列表；可以是文件名，或者非选项参数。

   # 所有选项参数都应以“-”（连字符）开头，选项可以附加参数（“-s20”）。如果出现可能代表文件名或其它任何东西的非选项参数（那些没有连字符作为前缀的other_args），应该在命令的最后出现。
   ```



## 2、selpg 程序逻辑

selpg 首先处理所有的命令行参数。在扫描了所有的选项参数（也就是那些以连字符为前缀的参数）后，如果 selpg 发现还有一个参数，则它会接受该参数为输入文件的名称并尝试打开它以进行读取。如果没有其它参数，则 selpg 假定输入来自标准输入。

1. 参数处理：

   ```bash
   # 强制参数（mandatory_opts）
   -sNumber 
   -eNumber
   # 可选参数 （optional_opts）
   -lNumber # 与“-f”互斥
   -f
   -dDestination
   ```

2. 输入处理：

   selpg 通过以下方法记住当前页号：

   - 如果输入是每页行数固定的，则 selpg 统计新行数，直到达到页长度后增加页计数器。
   - 如果输入是换页定界的，则 selpg 改为统计换页符。

   这两种情况下，只要页计数器的值在起始页和结束页之间这一条件保持为真，selpg 就会输出文本（逐行或逐字）。当那个条件为假（也就是说，页计数器的值小于起始页或大于结束页）时，则 selpg 不再写任何输出。


## 3、golang包的支持

​	[代码地址](https://github.com/hansenbeast/Service-Computing/blob/master/Assignment2-go-env/gowork/src/github.com/hansenbeast/selpg/selpg.go)

1. 使用 pflag 替代 goflag 以满足 Unix 命令行规范

   [pflag](https://github.com/spf13/pflag/tree/forOgier)

   ```bash 
   go get github.com/spf13/pflag #install到本地
   go test github.com/spf13/pflag #test
   import "github.com/spf13/pflag" #导入
   ```

   ```go
   // example
   // 将名为“flagname”的整型参数绑定到一个整型变量flagvar，注意与flag不同的是，需要添加shorthand参数
   var flagvar int
   func init() {
   	pflag.IntVar(&flagvar, "flagname", "f", 1234 , "help message for flagname")
       //pflag.IntVar(&variable_name, "flag_name", "shorthand", default_value , "help message for flag_name")
   }
   ```
   ```go
   /*================================= types =========================*/
   type selpg_args struct {
       start_page  int //起始页码
       end_page    int //终止页码
       in_filename  string //输入文件名
       page_len    int //每页行数
       form_deli   bool //是否按分页符分页
       print_dest string //打印机目的地
   }
   ```


2. 函数原型

   ```go
   /*================================= prototypes ====================*/
   func Usage(); //命令用法
   func Init(args *selpg_args); //初始化绑定参数的变量
   func process_args(args *selpg_args); //处理参数
   func process_input(args *selpg_args); //处理输入为标准输入或者文件重定向
   func print_write(args *selpg_args, line string, stdin io.WriteCloser); //处理输出为标准输出或者与标准输入关联的管道
   ```

3. 处理命令行参数

   在通常情况下，UNIX每个程序在开始运行的时刻，都会有3个已经打开的stream.。分别用来输入，输出和打印诊断和错误信息。通常他们会被连接到用户终端. 但也可以改变到其它文件或设备。Linux内核启动的时候默认打开的这三个I/O设备文件：标准输入文件stdin，标准输出文件stdout，标准错误输出文件stderr，分别得到文件描述符 0, 1, 2。

   同样在golang中导入os包，通过`fmt.Fprintf(os.Stderr, "Error message")`将错误信息输入stderr。

   ```go o
   func process_args(args *selpg_args) {
       //检查强制参数s，e
   	if args.start_page == -1 || args.end_page == -1 {
   		fmt.Fprintf(os.Stderr, "Error! %s: Not enough arguments\n\n", progname)
   		flag.Usage()
   		os.Exit(1)
   	}
   	...
       //判断s，e的值是否合法
   	if args.start_page > args.end_page || args.start_page < 1 || args.end_page < 1 {
   		fmt.Fprintln(os.Stderr, "Error! Page number invalid\n\n")
   		flag.Usage()
   		os.Exit(1)
   	}
   }
   ```


4. 处理输入

   ![1](Assets/1.png)

   ```go
   func process_input(args *selpg_args) {
       ...
       //记录结果
   	result := ""
   	//记录总行数
   	line_count := 0
   	//记录页码
   	page_count := 1
   	
   	if pflag.NArg() > 0 {
   		//读输入文件
   		args.in_filename = pflag.Arg(0)
   		output, err := os.Open(args.in_filename)
   		if err != nil {
   			fmt.Println(err)
   			os.Exit(1)
   		}
   		// 设置文件输入流的缓冲区
   		reader := bufio.NewReader(output)
   		// 按分页符’\f‘分页
   		if args.form_deli {
   			for pageNum := 0; pageNum <= args.end_page; pageNum++ {
   				line, err := reader.ReadString('\f')
   				if err != io.EOF && err != nil {
   					fmt.Println(err)
   					os.Exit(1)
   				}
   				if err == io.EOF {
   					break
   				}
   				page_count++
   				result += line
   			}
   		}else { //按行数分页
   			for {
                    //按行读取
   				line, err := reader.ReadString('\n')
   				if err != io.EOF && err != nil {
   					fmt.Println(err)
   					os.Exit(1)
   				}
   				if err == io.EOF {
   					break
   				}
   				line_count++
   				if(line_count > args.page_len){
   					page_count++
   					line_count = 1
   				}
   				if(page_count >= args.start_page && page_count <= args.end_page){
   					result += line
   				}
   			}
   		}
   	} else {
           //设置标准输入的缓冲区
   		scanner := bufio.NewScanner(os.Stdin)
   		for scanner.Scan() {
               //按行读取
   			line := scanner.Text()
   			line += "\n"
   			line_count++
   			if(line_count > args.page_len){
   				page_count++
   				line_count = 1
   			}
   			if page_count >= args.start_page && page_count <= args.end_page {
   				result += line
   			}
   		}
   	}
       //判断起始页码和终止页码是否在总页码的范围内
   	if(page_count < args.start_page || page_count < args.end_page){
   		fmt.Fprintln(os.Stderr, "\nError! Page number exceed the total number of the pages\n\n")
   		flag.Usage()
   		os.Exit(1)
   	}
       //如果合法，则处理输出
   	print_write(args, string(result), stdin)
   }

   ```



5. 管理子进程的标准输入

   ![2](Assets/2.png)

   ```go
   if args.print_dest != "" {
       // func Command(name string, arg ...string) *Cmd
   	// 函数返回一个*Cmd，用于使用给出的参数执行name指定的程序。返回值只设定了Path和Args两个参数。
       cmd = exec.Command("lp", "-d"+args.print_dest)
       
      	// func (c *Cmd) StdinPipe() (io.WriteCloser, error)
   	// StdinPipe方法返回一个在lp命令Start后与lp命令标准输入关联的管道。
       stdin, err = cmd.StdinPipe()
       
       if err != nil {
           fmt.Println(err)
       }
   } else {
       stdin = nil
   }

   //将输出写入stdin
   ...

   if args.print_dest != "" {
       stdin.Close()
       cmd.Stdout = os.Stdout
      
       //func (c *Cmd) Run() error
       //Run执行c包含的命令，并阻塞直到完成。
       cmd.Run()
   }
   ```

   ​

6. 处理输出

   ```go
   func print_write(args *selpg_args, line string, stdin io.WriteCloser) {
       //如果有打印机地址，则输出到标准输入的管道中
   	if args.print_dest != "" {
   		stdin.Write([]byte(line + "\n"))
   	} else {//否则输出到标准输出
   		fmt.Println(line)
   	}
   }
   ```

   ​




## 4、测试selpg的使用

测试文件input

```
line_1
line_2
...
line_146
```



1. `selpg -s=1 -e=1  < bin/input`标准输出省略了第21-72行

![3](Assets/3.png)

2. `selpg -s=1 -e=1  < bin/input > bin/output`

   输出文件output

![4](Assets/4.png)

3. `selpg -s=3 -e=3  < bin/input`

   ![5](Assets/5.png)

4. `selpg -s=3 -e=3 -l=10 < bin/input`

![6](Assets/6.png)

5. `selpg -s=3 -e=4  < bin/input`

   ![7](Assets/7.png)

6. `selpg -s=2 -e=1  < bin/input`

   ![8](Assets/8.png)

7. `selpg -s=2 -e=1  < bin/input 2>bin/output_err >bin/output`

   输出文件output

   ![9](Assets/9.png)

   错误信息输出文件output_err

   ![10](Assets/10.png)

8. ` cat bin/input | selpg -s=5 -e=7 -l=5`

   ![11](Assets/11.png)