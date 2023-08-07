



# Http-WebServer

Linux下 C轻量级Web服务器，Browser/Server（浏览器/服务器）结构。

通过浏览器来访问服务器，可以请求访问服务器的**图片，文本，音频，视频 或者目录文件**。



- 支持**高并发连接**数据交换。
- 只支持解析HTTP的GET请求报文。
- 通过对报文的解析，可以判断请求的是**目录**还是**文件**。



[TOC]



## single-file-server



非阻塞socket + epoll(ET和LT均实现) + 事件处理的并发模型。

这是一个不成熟练手的测试模型，只能访问**文件**。





## epoll-http-server



非阻塞socket + epoll(ET和LT均实现) + 事件处理的并发模型。 

这个模型可以访问**目录**或**文件**。

但**有BUG，服务器无法发送大文件给浏览器。**





## libevent-http-server



用Libevent库实现 的并发模型。可以访问**目录**或**文件**。

个人感觉这个版本是最好的，**无BUG**，因为它可以正常发送大文件给浏览器。







## Demo演示

- 浏览器端

![](https://github.com/YuGong123/http-server/blob/master/static/http.jpg)



- 请求视频文件演示(46.1M)

![](https://github.com/YuGong123/http-server/blob/master/static/single.jpg)



- 本地server端

![](https://github.com/YuGong123/http-server/blob/master/static/server.jpg)


  
- 压力测试
<video id="video" controls="" src="https://github.com/YuGong123/http-server/assets/91369699/4036cbb1-847b-462f-8fbb-1acfb6df3e91.mp4" preload="none"></video>


- 压力测试
<video id="video" controls="" src="https://github.com/YuGong123/http-server/assets/91369699/4036cbb1-847b-462f-8fbb-1acfb6df3e91" preload="none"></video>












## 关于项目的一些知识

### 1. errno为 **EINTR** , **EAGAIN** 和 **ECONNRESET**



经典错误代码：

```C
while ((n = read(fd, buf, sizeof(buf))) > 0) {		

    ret = send(cfd, buf, n, 0);

    if (ret == -1) { 
        printf("errno = %d\n", errno);
        perror("send error");	
        exit(1);
    }
}
```



情况一：

![](https://github.com/YuGong123/http-server/blob/master/static/Resource%20temporarily%20unavailable.jpg)



情况二：

![](https://github.com/YuGong123/http-server/blob/master/static/Connection%20reset%20by%20peer.jpg)



分析：

在网络通信中，read和send函数的返回值出现-1的情况，不能单纯地任务是出错。还有可能是阻塞。

尤其是 read、write、recv、send、recvfrom、sendto 函数，返回的errno为 **EINTR** 和 **EAGAIN** 时，不代表是一个错误，但会严重影响程序运行结果！通常使用continue来处理这种情况即可。



还有可能遇到浏览器端断开链接地情况，这个时候，errno为 **ECONNRESET**。 如果采用上述代码的话，server整个进程都会终止。毫无疑问，浏览器断开是浏览器自己的问题，server应该继续工作。应该采用 $break;$  语句断开该套接字。



**容错处理**

```C
// 循环读文件
char buf[4096] = {0};
int len = 0, ret = 0;
while( (len = read(fd, buf, sizeof(buf))) > 0 ) {   
    // 发送读出的数据
    ret = send(cfd, buf, len, 0);

    if (ret == -1) {
        // printf("errno = %d\n", errno);
        if (errno == EAGAIN) {
            // printf("-----------------EAGAIN\n");
            usleep(10);
            continue;
        }
        else if (errno == EINTR) {
            printf("-----------------EINTR\n");
            continue;
        }
        else if (errno == ECONNRESET) { // 对端重启，对端就要关闭。不能continue，会出bug。要break，让主进程建立新的socket。
            printf("-----------------Connection reset by peer.\n");
            break;
        }
        else {
            perror("send error");
            exit(1);
        }
    }
}
```







### 2. get_line()函数读取http协议

```c
// 获取一行 \r\n 结尾的数据，读取http协议的每一行 

int get_line(int cfd, char *buf, int size)
{
    int i = 0;
    char c = '\0';
    int n;
    while ((i < size-1) && (c != '\n')) {  
        n = recv(cfd, &c, 1, 0);    // 一次只读一个字符
        if (n > 0) {     
            if (c == '\r') {            
                n = recv(cfd, &c, 1, MSG_PEEK);     // 如果读到的字符是‘\r’,就在这个循环内，再来一次模拟读。
                if ((n > 0) && (c == '\n')) {       // 如果模拟读到的字符是‘\n’,就再来一次实际读。此时，c = '\r' --> c = '\n' --> c = '\n'          
                    recv(cfd, &c, 1, 0);
                } else {                       
                    c = '\r';                   // 很明显，如果模拟读到的字符 不是‘\n’，那么 c = '\r' --> c = '不是\n' --> c = '\r'      
                }
            }
            buf[i] = c;
            i++;
        } else {                // 如果 n <=0，说明读失败了，就将 c = '\n';那么下一场循环开始条件判断时，就会退出while循环。
            c = '\n';
        }
    }
    buf[i] = '\0';              // buf[2] = \0';
    
    // if (-1 == n)                // bun不能解开这里的注释，会出bug。个人猜测不能简单地认为读到-1是出错。
    // 	i = n;

    return i;                   // 返回实际读取的字节数--- 2。  a \r \n --> a \n \0
}
```







### 3. sscanf 函数+正则表达式 拆分http协议

```c
# 函数原型
int sscanf(const char *buffer，const char *format, [ argument ] ... )；
```



```c
// 读取一行http协议， 拆分， 获取 get 文件名 协议号	
char line[1024] = {0};
char method[16], path[256], protocol[16]; 

get_line(cfd, line, sizeof(line)); //读 http请求协议首行 GET /hello.c HTTP/1.1

sscanf(line, "%[^ ] %[^ ] %[^ ]", method, path, protocol);	
printf("cfd=%d, method=%s, path=%s, protocol=%s\n", cfd, method, path, protocol);

# method[GET], path[/hello.c], protocol[http/1.1]
```







### 4. stat()函数判断文件或目录

```c
struct stat st;
stat(path, &st);	// st是传出参数



// 如果是文件
if(S_ISREG(st.st_mode)) {       
    ....;
} 
// 如果是目录       
else if(S_ISDIR(st.st_mode)) {		
    ....;
}
```







### 5. scandir()函数快捷遍历目录

​	服务器端，可以使用文件操作时“递归遍历目录的”源码，实现遍历目录内文件名，回显给浏览器。另外标准C库中，提供了scandir函数，可以便捷的实现该功能。

```c
# 函数原型如下：
int scandir(const char *dirp, struct dirent ***namelist,
              			int (*filter)(const struct dirent *),
              			int (*compar)(const struct dirent **, const struct dirent **));

	scandir(待操作的目录，&子目录项列表数组，过滤器(通常NULL)，alphasort）；
	dirp: 待访问的目录名称
	struct dirent类型：（参考readdir()函数）
	struct dirent {
    	ino_t          	d_ino;       	/* inode number */
        off_t          	d_off;       		/* not an offset; see NOTES */
        unsigned short 	d_reclen;    		/* length of this record */
       	unsigned char  	d_type; 			
		char 			d_name[256]; 	/* filename */
     };
            
# compar:参数取如下函数即可:
		int alphasort(const void *a, const void *b); 默认是由的排序算法。
           
```



```c
# 调用：	
struct dirent** namelist;
int num = scandir(dirname, &namelist, NULL, alphasort);
for (i = 0;  i < num;  ++i)
    char* name = namelist[i]->d_name;
```







### 6. 借助telnet调试

​	可使用 telnet 命令，借助IP和port，模拟浏览器行为，在终端中对访问的服务器进行调试，方便**查看服务器回发给浏览器的http协议数据**。使用方式如下：

```c
$ telnet 127.0.0.1 9876(服务器的端口号)

GET /hello.c http/1.1

退出方式：
	先按 "CTRL + ]"，再输入 quit。
```



此时，终端中可查看到服务器回发给浏览器的http应答协议及数据内容。

```c
HTTP/1.1 200 OK
Content-Type:text/plain; charset=utf-8
Content-Length:490
Connection: close
```







### 7. 浏览器请求ico

​	准备一个favicon.ico 文件放置到 服务器提供访问的资源目录中。
​	浏览器在请求图片的同时，会请求一个ico图标，用于浏览器<title>标签文字部分前端的小图标显示。
​	这个ico的文件名固定——favicon.ico。因此，自行准备一个ico文件，放置于服务器提供给浏览器访问的目标目录即可。







### 8. HTTP协议基础

​	HTTP，超文本传输协议（ HyperText Transfer Protocol ）。互联网应用最为广泛的一种网络应用层协议。它可以减少网络传输，使浏览器更加高效。
​	通常HTTP消息包括客户机向服务器的请求消息和服务器向客户机的响应消息。

#### 请求消息(Request) 

浏览器 —> 发给 —> 服务器。主旨内容包含4部分：

1. **请求行: 说明请求类型, 要访问的资源, 以及使用的http版本**
2. **请求头: 说明服务器要使用的附加信息**
3. **空  行: 必须！, 即使没有请求数据**
4. **请求数据: 也叫主体, 可以添加任意的其他数据**



以下是浏览器发送给服务器的http协议头内容举例, 注意：9行的空行(\r\n)也是协议头的一部分：

1. ==**GET  /hello.c  HTTP/1.1**== 
2. Host: localhost:2222
3. User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux i686; rv:24.0) Gecko/201001    01 Firefox/24.0
4. Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
5. Accept-Language: zh-cn,zh;q=0.8,en-us;q=0.5,en;q=0.3
6. Accept-Encoding: gzip, deflate
7. Connection: keep-alive
8. If-Modified-Since: Fri, 18 Jul 2014 08:36:36 GMT
9. 



#### 响应消息(Response)

服务器 —> 发给 —> 浏览器。主旨内容包含4部分：

1. **状态行: 包括http协议版本号, 状态码, 状态信息**
2. **消息报头: 说明客户端要使用的一些附加信息**
3. **空  行: 必须!**
4. **响应正文: 服务器返回给客户端的文本信息**



以下是经服务器按照http协议，写回给浏览器的内容举例，1~9行是协议头部分。注意：9行\r\n的空行不可忽略

1. ==HTTP/1.1  200  Ok==
2. Server: xhttpd
3. Date: Fri, 18 Jul 2014 14:34:26 GMT
4. ==Content-Type: text/plain; charset=iso-8859-1 (必选项)==
5. Content-Length: 32  （ 要么不写 或者 传-1， 要写务必精确 ！ ）
6. Content-Language: zh-CN
7. Last-Modified: Fri, 18 Jul 2014 08:36:36 GMT
8. Connection: close 
9. 
20. #include <stdio.h>
21. 
22. int main(void)
23. {	
24. printf(“ Welcome to itcast ...  \n");
25. 
26. return 0;
27. }







#### 个人理解：

浏览器给服务器发数据，数据一定是用http协议发的。

浏览器，它跟服务器通信，一定是借助http协议来通信。所以一定用http协议发，且发的一定是请求协议。

但是，请求协议分两种，一种是get请求，另一种是post请求。

get请求是拿来主义，什么都不给你，就是要。

post请求是要的同时，会携带数据过去，给服务器一点东西。

本项目的服务器代码，主要服务于get请求。





### 9. chdir()函数改变工作目录

```c
// 改变进程工作目录
int ret = chdir(argv[2]);
if (ret != 0) {
    perror("chdir error");	
    exit(1);
}
```

./server 9876 /home/yugong/dir 时，服务器的当前工作目录是/home/yugong/dir ， 所以可以直接将客户端发来 请求hello.c时，可以直接用/hello.c获取
