---
title: 操作系统原理：C语言 threads多线程 实现单机聊天系统
date: 2020-04-24 13:59:02
copyright: true
tags: [Pipe管道, 单机聊天系统, C语言, 多线程]
categories:
- 技术笔记
- 操作系统原理

top: 95
---



&emsp;&emsp;这个实验会建立一个全双工系统（Full-Duplex），实现两个管道同时收发消息。在程序中会涉及到3个文件，2个管道，2个进程，4个线程。线程之间的拓扑图如下：
<!--more-->

![通信示意图](/SongXJ01/images/操作系统原理_C语言threads多线程实现单机聊天系统/通信示意图.png)

【完整代码附在文章最后】

----

## 创建连通管道
&emsp;&emsp;首先创建`fifo_create.c`文件来事先创建2个管道，分别为A发送B接收、A接收B发送。使用`mkfifo()`语句创建管道，分别标识为“A2B”、“B2A”。
访问权限为`0644`，第一位0不算，从左至右三个数字分别代表`rw-` `r--` `r--`对应转化为的二进制`110` `100` `100`。十进制第1个数字代表**文件所有者**的权限，十进制第2个数字代表**同组用户**的权限，十进制第3个数字代表**其他用户**的权限。
之后验证`mkfifo()`的返回值，验证管道是否创建成功。如果创建成功，当前目录下回多出来“A2B”、“B2A”两个文件。

```c
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>

int main(){
	int A2B = mkfifo("A2B", 0644);
	int B2A = mkfifo("B2A", 0644);
	if (A2B == -1)	
		printf("Falled to create the FIFO of A2B!\n");
	if (B2A == -1)
		printf("Falled to create the FIFO of B2A!\n");

	return 0;
} 
```

---
## 创建2个进程4个线程
### 连接标记符
&emsp;&emsp;创建`A.c`和`B.c`两个文件，在两个文件中均创建`send`和`receive`两个线程。因为两个文件类似，这里以`A.c`为例，`B.c`同理。
在`A.c`中首先定义一个全局变量，用来标记两个管道是否连接，如果均连接成功，则`flag=1`，有任意一个管道断开则`flag=0` 。

```c
//全局变量，标记是否连接，两个管道中有一个断开，则flag变为0
int flag = 1;	
```

### 发送消息
&emsp;&emsp;通过`send`函数实现发送消息。在管道连通之前，即在`open`语句等待管道连通。这里`open`语句使用只写的方式进行打开，如果管道连接成功，程序继续向下运行。
定义一个循环进行消息的多次发送，使用`fgets()`语句从指定的标准输入流`stdin`中读取一行，并把它存储在`buf`所指向的字符串内，多余的长度使用`\0`填充，而且`fgets()`语句包含换行符，所以在之后的`receive`函数中打印的时候不需要再添加换行符。
之后使用`write`语句将储存在`buf`中的全部字符串写入到刚才打开的`A2B`管道中。然后进行判断，如果刚才输入的是“88”则将标识符`flag`变成0，并打印下线提示，break出循环。如果输入的不是“88”，则在命令行打印`[A]:`，等待下一次输入。
最后关闭管道，退出线程。


```c
//A给B发送消息
void* send() {	
	char buf[1024];
	printf("Waiting...\n");
	int fd = open("A2B", O_WRONLY);
	printf("Connected!\n===============================================\n");
	if (fd == -1)
		printf("Open fifo error!\n");
	
	printf("[A]:");
	while (flag == 1){
		fgets(buf, 1023, stdin);
		write(fd, buf, 1023);
		if (buf[0] == '8' && buf[0] == '8') {
			flag = 0;
			printf("You have been offline.\n");
			fflush(stdout);
			break;
		}
		else if (flag == 1) 
			printf("[A]:");
	}
	close(fd);
	pthread_exit(NULL);	//退出线程
}
```

### 接收消息
&emsp;&emsp;通过`receive`函数实现接收消息的功能。在管道连通之前，即`open`语句等待管道连通。这里`open`语句使用只读的方式进行打开，如果管道连接成功，程序继续向下运行。因为在本实验中两个通道一定是同时连接的，在`send`函数中已经打印连接成功的提示，所以在`receive`中就没有重复打印。
&emsp;&emsp;定义一个循环进行消息的多次接收，使用`read`语句将管道中的字符串全部储存在`buf`中，之后在末尾加上一个`\0`，表示字符串的结束。
&emsp;&emsp;然后进行判断，如果刚才接收到的是“88”则将标识符`flag`变成0，打印对方的最后一句话，同时打印对方已经下线的提示，break出循环。如果输入的不是“88”，则在命令行打印对方输入的内容后继续打印`[A]:`，等待下一次输入。
&emsp;&emsp;在打印接收到的对方消息时需要先在前面加上4个退格符，删除之前输入的`[A]:`这四个字符。另外要使用`fflush(stdout);`语句强制刷新标准输出缓冲区。
&emsp;&emsp;最后关闭管道，退出线程。


```c
//A接收B发来的消息
void* receive() {	
	char buf[1024];
	int fd = open("B2A", O_RDONLY);
	if (fd == -1)
		printf("Open fifo error!\n");

	while (flag == 1){
		read(fd, buf, 1023);
		//如果对方发“88”则断开连接，退出进程
		if (buf[0] == '8' && buf[0] == '8') {
			flag = 0;
			memset(buf + 1023, '\0', 1);
			printf("\b\b\b\b[B]:%s", buf);
			printf("Person B have been offline.\n");
			fflush(stdout);	//强制刷新输出缓冲区
			break;
		}
		else if(flag == 1){
			memset(buf + 1023, '\0', 1);
			printf("\b\b\b\b[B]:%s", buf);//删除之前输入的[X]:这四个字符
			printf("[A]:");
			fflush(stdout);	//强制刷新输出缓冲区
		}
	}
	close(fd);
	pthread_exit(NULL);	//退出线程
}
```
### 双线程
&emsp;&emsp;在主函数中建立两个线程，并分别调用`send`和`receive`两个函数，并等待它们退出。


```c
int main(){
	pthread_t tid1, tid2;	//线程ID
	pthread_attr_t attr;	//线程属性
	pthread_attr_init(&attr); //设置默认线程属性

	//执行两个线程分别进行收发
	pthread_create(&tid1, &attr, send, NULL);
	pthread_create(&tid2, &attr, receive, NULL);

	//等待两个线程
	pthread_join(tid1, NULL);
	pthread_join(tid2, NULL);

	return 0;
} 
```


---



## 程序运行过程
1. 等待连接
![等待连接](/SongXJ01/images/操作系统原理_C语言threads多线程实现单机聊天系统/等待连接.png)


2. 管道连接成功
![管道连接成功](/SongXJ01/images/操作系统原理_C语言threads多线程实现单机聊天系统/管道连接成功.png)

3. 进行全工聊天
![进行全工聊天](/SongXJ01/images/操作系统原理_C语言threads多线程实现单机聊天系统/进行全工聊天.png)

4. 聊天结束
![聊天结束](/SongXJ01/images/操作系统原理_C语言threads多线程实现单机聊天系统/聊天结束.png)

---
## 总结

&emsp;&emsp;本次实验结合了**多线程**和**管道连接**的知识，实现了一个简单的单机聊天功能。在本次实验中有以下几个需要注意的地方：

### fgets() 的特点：
* `gets()`和`fgets()`都可以读取空格。
* `gets()`读取一行，不能设定读取容量，这一行有多少便读取多少，有可能会发生内存溢出。然而`fgets()`有三个参数，它是从 stream 流中读取 size 个字符存储到字符指针变量 s 所指向的内存空间。它的返回值是一个指针，指向字符串中第一个字符的地址。
* 在本实验中`fgets() `函数的读取大小均为1023，基本上满足了普通聊天的需要。`fgets()` 函数的size参数如果小于字符串的长度，那么字符串将会被截取；如果size大于字符串的长度则多余的部分系统会自动用 `\0` 填充。
 * `fgets() `函数会读取换行符，所以在receive接收后，打印时不需要再添加换行符。

### fflush()强制刷新缓冲区：
&emsp;&emsp;在本实验中使用`fflush()`和`stdout`、`stdin`结合，可以将标准输入输出流中的数据强制刷新在命令行中。特别是在打印完接收的数据后，再次打印`[A]:`等待输入的字符串时，如果不进行强制刷新，这个字符串不会立刻显示在命令行中，而是跟随下一次的输入或输出结果同时显示。

### “\b”退格：
&emsp;&emsp;因为每次打印完聊天消息后，都需要继续打印`[A]:` 等待输入，而如果未进行输入而是接收了对方的消息，那么`[A]:`这个字符串就多余了，需要使用4个`\b`删除。

---
## 实验的不足之处
* 本实验在判断字符串是否为“88”时，使用的方法比较笨，同时比较两个字符完成的，其实可以使用`strcmp`函数直接对两个字符串进行比较。
* 本实验的最后还有一个小bug，就是在`A`输入“88”之后，两个程序不能立刻退出，而是需要在`B`上输入一个回车后才能退出，`B`对`A`发送消息情况同样。这个bug我试了很多方法都不能很好的解决，希望路过的大佬提供建议。

---
## 代码
### fifo_create.c
```c
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>

int main(){
	int A2B = mkfifo("A2B", 0644);
	int B2A = mkfifo("B2A", 0644);
	if (A2B == -1)	
		printf("Falled to create the FIFO of A2B!\n");
	if (B2A == -1)
		printf("Falled to create the FIFO of B2A!\n");

	return 0;
} 

```
### A.c

```c
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <pthread.h>

//全局变量，标记是否连接，两个管道中有一个断开，则flag变为0
int flag = 1;	

//A给B发送消息
void* send() {	
	char buf[1024];
	printf("Waiting...\n");
	int fd = open("A2B", O_WRONLY);
	printf("Connected!\n===============================================\n");
	if (fd == -1)
		printf("Open fifo error!\n");
	
	printf("[A]:");
	while (flag == 1){
		fgets(buf, 1023, stdin);
		write(fd, buf, 1023);
		if (buf[0] == '8' && buf[0] == '8') {
			flag = 0;
			printf("You have been offline.\n");
			fflush(stdout);
			break;
		}
		else if (flag == 1) 
			printf("[A]:");
	}
	close(fd);
	pthread_exit(NULL);	//退出线程
}

//A接收B发来的消息
void* receive() {	
	char buf[1024];
	int fd = open("B2A", O_RDONLY);
	if (fd == -1)
		printf("Open fifo error!\n");

	while (flag == 1){
		read(fd, buf, 1023);
		//如果对方发“88”则断开连接，退出进程
		if (buf[0] == '8' && buf[0] == '8') {
			flag = 0;
			memset(buf + 1023, '\0', 1);
			printf("\b\b\b\b[B]:%s", buf);
			printf("Person B have been offline.\n");
			fflush(stdout);	//强制刷新输出缓冲区
			break;
		}
		else if(flag == 1){
			memset(buf + 1023, '\0', 1);
			printf("\b\b\b\b[B]:%s", buf);//删除之前输入的[X]:这四个字符
			printf("[A]:");
			fflush(stdout);	//强制刷新输出缓冲区
		}
	}
	close(fd);
	pthread_exit(NULL);	//退出线程
}


int main(){
	pthread_t tid1, tid2;	//线程ID
	pthread_attr_t attr;	//线程属性
	pthread_attr_init(&attr); //设置默认线程属性

	//执行两个线程分别进行收发
	pthread_create(&tid1, &attr, send, NULL);
	pthread_create(&tid2, &attr, receive, NULL);

	//等待两个线程
	pthread_join(tid1, NULL);
	pthread_join(tid2, NULL);

	return 0;
} 

```

### B.c

```c
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <pthread.h>

int flag = 1;

//B给A发送消息
void* send() {	
	char buf[1024];
	printf("Waiting...\n");
	int fd = open("B2A", O_WRONLY);
	printf("Connected!\n===============================================\n");
	if (fd == -1)
		printf("Open fifo error!\n");
	
	printf("[B]:");
	while (flag == 1){
		fgets(buf, 1023, stdin);		
		write(fd, buf, 1023);
		if (buf[0] == '8' && buf[0] == '8') {
			flag = 0;
			printf("You have been offline.\n");
			fflush(stdout);
			break;
		}
		else if (flag == 1) 
			printf("[B]:");
	}

	close(fd);
	pthread_exit(NULL);	//退出线程
}

// B接收A发来的消息
void* receive() {	
	char buf[1024];
	int fd = open("A2B", O_RDONLY);
	if (fd == -1)
		printf("Open fifo error!\n");

	while (flag == 1) {
		read(fd, buf, 1023);
		if (buf[0] == '8' && buf[0] == '8') {
			flag = 0;
			memset(buf + 1023, '\0', 1);
			printf("\b\b\b\b[A]:%s", buf);
			printf("Person A have been offline.\n");
			fflush(stdout);
			break;
		}
		else if (flag == 1) {
			memset(buf + 1023, '\0', 1);
			printf("\b\b\b\b[A]:%s", buf);
			printf("[B]:");
			fflush(stdout);
		}
	}

	close(fd);
	pthread_exit(NULL);	//退出线程
}


int main(){
	pthread_t tid1, tid2;	//线程ID
	pthread_attr_t attr;	//线程属性
	pthread_attr_init(&attr); //设置默认线程属性

	//执行两个线程
	pthread_create(&tid1, &attr, send, NULL);
	pthread_create(&tid2, &attr, receive, NULL);

	//等待两个线程
	pthread_join(tid1, NULL);
	pthread_join(tid2, NULL);

	return 0;
} 

```
