# ANA IBD Task 1 http

## 故事背景（仅供娱乐，无任务相关内容）

**Disclaimer:** 本故事纯属虚构，与现实中一切人物和团体无关。

> zbc发现执行DOS攻击以后，Mark没有崩溃，还在正常运行，于是他加大了攻击力度，开始执行DDOS攻击，企图摧毁Mark的整个服务。A1pha在发现zbc的恶意行为后，已经偷偷将Mark的实体迁移到一台除了A1pha自己知道以外没有人知道的服务器上。但是，A1pha为了戏弄zbc，没有关闭原来的服务器，而是和佐地圭在原来的服务器上开发并部署了一个MagicLang编写的http服务，对请求者返回胡言乱语。当zbc的botnet访问它们认为的"Mark"时，得到了一堆垃圾。于是zbc气急败坏，从暗网上雇佣了一些顶级黑客来偷取Mark的访问权限，他会成功吗？
>
> （后面的文字消失了，只留下了几个锟斤铐）

## 前置知识

本教程来源于**UC Berkeley**的**CS162**课程，英文原版文档在[这里](https://inst.eecs.berkeley.edu/~cs162/sp23/static/hw/hw-http/)。

像以前一样，从staff仓库pull启动代码，合并到你自己的student仓库中：

```bash
git pull staff master
```

如果弹出一个编辑器让你写Merge message，直接Ctrl+X，然后输入Y，接着回车即可。

以下是你将要实现的 HTTP 服务器的一些基本信息。

### HTTP 请求的结构
HTTP 请求报文的格式是

1. HTTP 请求行。HTTP请求行包含方法、请求 URI 和 HTTP 协议版本。
2. HTTP 标头行（header lines）。
3. 一行空行（即 CRLF 本身）。

HTTP 中使用的行结束符是 CRLF，在 C 语言中表示为 `\r\n`。

下面是谷歌浏览器向运行在本地主机（127.0.0.1）8000 端口的 HTTP 网络服务器发送的 HTTP 请求报文示例（CRLF 使用转义序列写出）：

```yaml
GET /hello.html HTTP/1.0\r\n
Host: 127.0.0.1:8000\r\n
Connection: keep-alive\r\n
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8\r\n
User-Agent: Chrome/45.0.2454.93\r\n
Accept-Encoding: gzip,deflate,sdch\r\n
Accept-Language: en-US,en;q=0.8\r\n
\r\n
```

HTTP头（header lines）提供了有关请求的信息。我们可以用浏览器自带的工具深入了解请求的结构。打开网页浏览器的开发工具，然后点击 "网络 "选项卡，可以查看请求任何网页时发送的标头（header lines）。下面是一些 HTTP 请求头类型：

1. Host：Host请求头包含 HTTP 请求 URL 的主机名部分（例如 ana.xjtu.edu.cn 或 127.0.0.1:8000）
2. User-Agent：User-Agent请求头用于标识 HTTP 客户端程序，形式为 Program-name/x.xx，其中 x.xx 是程序的版本。在上例中，谷歌 Chrome 浏览器将 User-Agent 设置为 Chrome/45.0.2454.93（至少在网络发展初期是这样的。现在的用户代理一般都是一团糟。如果你想知道为什么会出现这种情况，可以查看[背后的历史](https://webaim.org/blog/user-agent-string-history/)）。


### HTTP 响应的结构

HTTP 响应信息的格式是

1. 状态行。状态行包含 HTTP 协议版本、状态代码和对状态代码的描述。
2. HTTP 标头行（header lines）。
3. 一个空行（即本身为 CRLF 的一行）。
4. HTTP 请求的主体（即内容）。

下面是一个 HTTP 响应示例，状态代码为 200，正文由 HTML 文件组成（CRLF 使用转义序列写出）：

```
HTTP/1.0 200 OK\r\n
Content-Type: text/html\r\n
Content-Length: 84\r\n
\r\n
<html>\n
<body>\n
<h1>Hello World</h1>\n
<p>\n
Let's see if this works\n
</p>\n
</body>\n
</html>\n
```
#### 状态行
典型的状态行可能是 HTTP/1.0 200 OK（如上例）、HTTP/1.0 404 Not Found 等。

状态代码是一个三位整数，第一位确定响应的一般类别。(如果你想了解更多信息，请点击[此处](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)）：

1. 1xx 表示仅有一条信息
2. 2xx 表示成功
3. 3xx 将客户端重定向到另一个 URL
4. 4xx 表示客户端出错
5. 5xx 表示服务器出错

#### 标头行（header lines）
标头行提供有关响应的信息。下面是一些 HTTP 响应头类型：

1. Content-Type：附加到响应的数据的 MIME 类型，如 text/html 或 text/plain
2. Content-Length：响应正文的字节数

### 服务器概要

从网络角度看，基本 HTTP 网络服务器应实现以下功能。

1. 创建监听套接字（Socket）并将其绑定到端口
2. 等待客户端连接到端口
3. 接受客户端并获取新的连接套接字
4. 读入并解析 HTTP 请求
5. 执行以下两件事之一：（由命令行参数决定）
a. 从本地文件系统中提供文件，或生成 404 Not Found（未找到文件）
b. 将请求代理到另一个 HTTP 服务器。使用代理时，HTTP 服务器会将请求流式传输（此过程成为转发，也叫代理）到远程 HTTP 服务器。来自远程HTTP服务器的响应会被发回给客户端。

![代理结构图](https://s1.imagehub.cc/images/2023/09/23/http_proxy_diagram.png)

https 服务器要么处于文件模式，要么处于代理模式；它不会同时做这两件事。

6. 向客户端发送适当的 HTTP 响应头和附加文件/文档（或错误信息）

我们提供的代码已经实现了第 2-4 步。

### 用法
下面介绍如何从 shell 调用 http_server。我们已经帮你实现好了参数解析。本程序的参数格式如下：

```
./httpserver --help
Usage: ./httpserver --files any_directory_with_files/ [--port 8000 --num-threads 5]
       ./httpserver --proxy inst.eecs.berkeley.edu:80 [--port 8000 --num-threads 5]
```

可用选项有：

`--files`

选择提供文件的目录。应从 `hw-http/` 文件夹中提供文件（例如，如果当前在 hw-http/ 文件夹中，则只需使用 `--files www/`）。

`--proxy`

选择要代理的 "上游" HTTP 服务器。参数可以是冒号后的端口号（例如 `inst.eecs.berkeley.edu:80`）。如果未指定端口号，默认端口为 80。

`--port`

选择 HTTP 服务器侦听传入连接的端口。在文件模式和代理模式下均使用。如果未指定端口号，默认端口为 8000。如果要使用 0 至 1023 之间的端口号，则需要以root用户身份运行 HTTP 服务器。这些端口是 "保留 "端口，只能由 root 用户绑定。可以通过运行`sudo http_server_rs --port PORT --files www/`.

`--num-threads`

表示线程池中能并发处理客户端请求的线程数。该参数默认未被使用，你可以自行决定是否正确使用。

运行 `make` 会得到 4 个可执行文件：`httpserver`、`forkserver`、`threadserver` 和  `poolserver`。

**注意：**每次对源文件进行修改以后，可执行文件不会自动更新，你需要重新`make`。

### 访问你的服务器

你可以使用 curl 程序发送 HTTP 请求。使用 curl 的示例如下：

```
curl -v http://0.0.0.0:8000/
curl -v http://0.0.0.0:8000/index.html
curl -v http://0.0.0.0:8000/path/to/file
```

你也可以使用 netcat (nc)，直接通过网络套接字（Socket）打开与 HTTP 服务器的连接，然后键入 HTTP 请求（或从文件中导入）。

```
> nc -v 0.0.0.0 8000
Connection to 0.0.0.0 8000 port [tcp/*] succeeded!
> (Now, type out your HTTP request here.)
```

完成"针对目录的请求"任务后，你就可以打开网页浏览器访问 HTTP 服务器，进入 http://localhost:8000/。

![访问效果图](https://s1.imagehub.cc/images/2023/09/23/website.png)

### 故障排查


1. 出现`Failed to bind on socket: Address already in use`

    这意味着你有一个 `httpserver` 在后台运行。如果你的代码没有及时释放该进程，或者你断开了与虚拟机的连接却从未关闭过 `httpserver`，就可能发生这种情况。你可以运行` pkill -9 httpserver` 来解决这个问题。如果还不行，你可以通过`--port`指定一个不同的端口。通常会有很多人同时在教学机器上测试自己的http服务器，为了避免冲突，我们建议选择一个不是8000的端口进行你自己的测试。

2. 出现`Failed to bind on socket: Permission denied`

    如果使用的端口号小于 1024，可能会出现此错误。只有 root 用户可以使用“众所周知（Well Known）”的端口（1 至 1023），因此应选择较高的端口号（1024 至 65535）。


## Socket
在 `serve_forever` 函数中完成服务器套接字的设置。

1. 使用 `bind` 系统调用将套接字绑定到命令行指定的 IPv4 地址和端口（即 `server_port`）。
2. 使用 `listen` 系统调用开始监听进入的客户端。在此阶段，请将 `listen` 的 `backlog`参数设置为 1024。在性能负载测试中，你可以随意调整这个值，并讨论其对服务器性能的影响。

完成这部分后，`curl`应该会输出 "Empty reply from server".

## GET请求
### 针对文件的请求

实现 `handle_files_request`，以处理文件的 HTTP GET 请求。相应地，你需要调用 `serve_file`。你还应该能够处理对 files 目录下子目录中文件的请求（例如，`GET /images/hero.jpg`）。

+ 如果 path 表示的文件存在，则调用 serve_file。读取文件内容并写入客户端套接字。
	+ 确保设置了正确的 `Content-Length` HTTP 头信息。该标头的值应为 HTTP 响应正文的大小（以字节为单位）。例如，`Content-Length: 7810`。可以使用 `snprintf` 将整数转换为字符串。
	+ 这项任务必须使用`read`和`write`系统调用。任何使用 `fread` 或 `fwrite` 的实现都将不得分。这完全是出于教学方面的考虑；我们希望你能接受这样一个事实，即低级 I/O 可能会也可能不会对请求的所有字节执行整个操作。
+ 否则，向客户端提供 `404 Not Found` 响应（HTTP 主体是可选的）。在 HTTP 请求过程中可能会出现很多问题，但我们只希望你能支持 `404 Not Found` 错误信息以处理不存在的文件。完成这部分后，对`index.html`进行`curl`时应输出文件 `index.html` 的内容。

### 针对目录的请求

实现 `handle_files_request`，以处理文件和目录的 HTTP GET 请求。

+ 现在需要确定 `handle_files_request` 中的 `path` 指的是文件还是目录。`stat` 系统调用和 `S_ISDIR` 或 `S_ISREG` 宏在这方面很有用。在确定 `path` 是文件还是目录后，需要相应地调用 `serve_file` 或 `serve_directory`。
+ 如果目录中包含 `index.html` 文件，则响应 `200 OK` 和 `index.html` 文件的全部内容。针对目录的请求不一定以"/"结尾。
	+ `libhttp.c` 中的` http_format_index` 函数可能会有用。
+ 如果目录中不包含 `index.html` 文件，则响应一个 HTML 页面，其中包含指向目录所有直接子目录的链接（类似于 `ls -1`），以及指向父目录的链接。
	+ `libhttp.c` 中的 `http_format_href` 函数可能很有用。
	+ 要列出一个目录的内容，可以使用 `opendir` 和 `readdir` 函数。

+ 如果目录不存在，则向客户端发送 404 Not Found 响应。

+ 你不必担心链接中会出现多余的斜线（如 `//files///a.jpg` 就完全没问题）。文件系统和网络浏览器都能容忍这种情况。

+ 除文件和目录外，你不需要处理文件系统对象（例如，你不需要处理符号链接、管道或特殊文件）。

+ 请记住，在从 `handle_files_request` 函数返回之前关闭客户端Socket。
+ 尽可能使用辅助函数重用类似代码。这会让你的代码更容易调试！

完成这部分后，对根目录 / 进行"curl"， 应输出文件 index.html 的内容。至此，你应该能够在gradescope上通过所有有关"Basic Server"的测试。

## 代理

为了完成代理任务，你需要实现 `handle_proxy_request`。`handle_proxy_request`函数将 HTTP 请求代理到另一个 HTTP 服务器。我们已经为你处理了用于设置连接的代码。你应该阅读并理解它，但不需要修改它。

下面是已经实现的内容。

+ 我们使用 `--proxy` 命令行参数的值，其中包含上游 HTTP 服务器的地址和端口号。这两个值存储在全局变量 `server_proxy_hostname` 和 `server_proxy_port` 中。
+ 我们对服务器代理主机名进行 DNS 查询，这将会查询到主机名的 IP 地址（详情见 `gethostbyname`））。
+ 我们创建一个网络套接字，并将其连接到从 DNS 获取的 IP 地址。详见`socket`和`connect`函数。
+ `htons` 用于设置套接字的端口号（请注意：x86 内存中的整数是小端的，而网络设备则是大端的）。还要注意的是，HTTP 是一种 `SOCK_STREAM` 协议（即TCP协议）。

以下是你需要注意的事项。

+ 在两个套接字（HTTP 客户端 fd 和目标 HTTP 服务器 fd）上等待新数据。数据到达后，应立即将其读入缓冲区，然后写入另一个套接字。你应当在 HTTP 客户端和目标 HTTP 服务器之间保持双向通信。你的代理必须支持多个请求/响应。
	+ 这比写入文件或从 stdin 读取数据更加棘手，因为你不知道双向流的哪一方会先写入数据，也不知道它们在收到响应后是否会写入更多数据。在代理模式下，你会发现在同一连接中会发送多个 HTTP 请求/响应，而你的 HTTP 服务器则不同，它只需要在每个连接中支持一个请求/响应。
	+ 你应该使用 `pthreads` 来完成这项任务。考虑使用两个线程来实现双向通信，一个从 A 到 B，另一个从 B 到 A。
	+ 请勿使用 `select`、`fcntl` 或类似函数。在前几个学期，我们曾经推荐过这种方法，但我们发现这种方法太混乱了。
+ 如果任一套接字关闭，通信将无法继续，因此应关闭两个套接字以终止连接。

完成这部分后，你应当在gradescope上通过所有有关"Proxy"的测试。

## 服务器杂烩

在本节中，你将实现多个不同的服务器。有了条件编译预处理器指令（即 `#ifdef` 指令），我们只需改变在每个不同服务器中调用请求处理程序的方式。

### 基于Fork的服务器

为了实现基于Fork的服务器，你无需编写太多新代码。

+ 子进程应使用客户端套接字 fd 调用 `request_handler`。提供响应后，子进程将终止。
+ 父进程将继续监听并接受传入连接。它不会等待子进程。
+ 请记住，父进程和子进程都要适当关闭套接字。

### 多线程服务器

实现多线程服务器。

+ 创建一个新的 `pthread`，向客户端发送正确的响应。
+ 原线程将继续监听并接受传入连接。它不会"join"新线程。

### 基于线程池的服务器

实现基于线程池的服务器。

+ 线程池应能同时为`--num-threads`个客户端提供服务。请注意，我们通常在程序中使用`--num-threads + 1`个线程。原始线程负责在 `while` 循环中接受客户端连接，并将相关请求分派给线程池中的线程处理。
+ 首先查看`wq.h` 中的函数。
	+ 原始线程（即你启动 `httpserver` 程序的线程）需要调用`wq_push`函数，将从 `accept` 接收到的客户端套接字文件描述符放到 `wq_t work_queue`中。`wq_t work_queue`在 `httpserver.c` 顶部声明并在` wq.h` 中定义。
	+ 然后，线程池中的线程应使用 `wq_pop` 获取下一个要处理的客户端套接字文件描述符。
+ 你需要让服务器产生 `--num-threads` 新线程，这些线程将循环执行以下操作：
	+ 阻塞调用 `wq_pop`，获取下一个客户端套接字文件描述符。
	+ 成功弹出待服务客户端套接字 fd 后，调用相应的请求处理程序来处理客户端请求。

## 性能测试

测试并讨论`httpserver`、`forkserver`、`threadserver` 和`poolserver` 的性能。我们将使用 Apache HTTP 服务器基准测试工具（简称 ab）对每种服务器类型进行负载测试。如果你使用的不是教学机，可以用 `sudo apt-get install -y apache2-utils` 安装 `ab`。

1. 运行 `./httpserver --files www/`。
2. 在另一个终端窗口中，运行 `ab -n 500 -c 10 http://localhost:8000/`

	该命令以 10 的并发级别发出 500 个请求（即一次发送 10 个请求）。阅读 `man ab` 了解该工具的更多信息。你可以在终端或首选搜索引擎中键入 "man ab"。

	请注意 ab 如何输出每次请求的平均时间。注意这个值，并评论当我们改变服务器处理请求的方式时，它是如何变化的。

3. 使用 ab 对 `forkserver`、`threadserver` 和 `poolserver` 进行负载测试。调整 `n` 和 `c` 变量以及 `poolserver` 中线程池的大小。

询问你自己下列问题，我们不对这些问题的答案进行测试。

1. 在 `httpserver`上运行`ab`。当 `n` 和 `c` 变大时，会发生什么情况？
2. 在 `forkserver`上运行`ab`。当 `n` 和 `c` 变大时，会发生什么情况？将这些结果与上一问题的答案进行比较。
3. 在`threadserver`上运行`ab`。当 `n` 和 `c`  变大时，会发生什么情况？将这些结果与上一问题的答案进行比较。
4. 在`poolserver`上运行 `ab`。当 `n` 和 `c` 变大时，会发生什么情况？将这些结果与前面问题的答案进行比较。

## 提交
你的代码不应包含无关的或调试打印语句，因为这将干扰autograder。要提交并推送到gradescope，请将更改commit并且push到你的git repo。然后将其提交到Gradescope上的**http**作业。