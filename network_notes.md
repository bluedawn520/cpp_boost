[TOC]

---



## Socket原理

![image-20230528221651165](network_pic\image-20230528221651165.png)

> **从网卡到内核缓冲区**的数据传递方式通常采用 **DMA** 技术。
>
> - 网卡接收数据包并存储在网卡的接收缓冲区中；
> - 网卡的网络设备驱动程序检测到接收缓冲区有数据时，会通过 **DMA引擎 **将数据从 网卡的接收缓冲区 直接传输到 内存中的内核缓冲区 中，而不需要CPU的干预；
> - DMA引擎 负责处理数据的传输和拷贝。 从网卡读数据，并将其直接写入内核缓冲区中，或通过内存控制器将数据写入内存中的指定位置；
> - 一旦数据传输完成，DMA引擎 会触发中断 通知网络设备驱动程序，表示数据已成功传输到内存中了。
> - 网络驱动程序可以进一步处理数据。

> 1. 网卡将数据报 以**DMA方式**传输到 内存RingBuffer中，然后 **向CPU发出硬中断**；
> 2. CPU响应中断请求，调用 **网卡启动时注册的中断处理函数**；
> 3. 中断处理函数发起 **软中断请求**；
> 4. 内核线程`ksoftirqd`线程发现有软中断请求到来，**先关闭硬中断**，然后调用**驱动**的`poll`函数收包；
> 5. 协议栈将封装后的数据包（数据包封装在**sbk**数据结构中）放到对应的**socket缓存队列**中；
> 6. 调用`socket waitqueue` 的**回调函数链**，**激活**等待的进程。

> socket wait queue：是一种用于**管理**等待连接或数据的**进程**的机制。
>
> - 当一个进程在等待某个socket事件时，该进程会将自己添加到socket的等待队列中，然后睡眠等待，暂时释放CPU资源。等待队列负责管理这些等待的进程，并在合适的时机将其唤醒，以便处理相应的事件。
> - **等待队列头**：是一个数据结构，用于管理等待队列中的进程。

![image-20230528221725421](network_pic\image-20230528221725421.png)

> `NET_TX_SOFTIRQ`（**网络传输软中断**）

> 1. 使用系统调用发送数据；
> 2. 拷贝数据到内核缓冲区；
> 3. 内核协议栈将数据进行处理，封装成skb数据结构；
> 4. 然后**内核协议栈** 将其发送到 **驱动的RingBuffer**；
> 5. 网卡进行发送数据；
> 6. 网卡发送完数据后，向CPU发出一个硬件中断；
> 7. 在处理中断处理程序时，其会发出一个 网络传输软中断**NET_TX_SOFTIRQ**；
> 8. 然后内核线程**ksoftirqd**会将网卡的**RingBuffer**进行清理；



![image-20230528221837078](network_pic\image-20230528221837078.png)



![image-20230528224833188](network_pic\image-20230528224833188.png)

> 若在**shutdown**中指定参数是`SHUT_RDWR`的话，socket资源并没有释放;需要调用`close`进行释放资源。

> **RST（Reset）报文**是TCP（传输控制协议）中的一种控制报文，用于**中断或重置TCP连接**。当一方收到不正常的TCP数据流时，可以发送RST报文来中断连接。



![image-20230528224922043](network_pic\image-20230528224922043.png)

![image-20230528225238014](network_pic\image-20230528225238014.png)

> **半关闭**： `shutdown` 中参数指定为 `shut_wr` 或 `shut_rd`。
>
> **`SHUT_WR`**: 关闭写方向；A发送B一个`FIN`包，A不再发送数据给B，但B可以发送数据给A，且A可以接收报文；
>
> **`SHUT_RD`**: 关闭读方向；A不再读数据，若B向A发送数据，则A会向B发送`RST`。
>
> **`SHUT_RDWR`**: 即关闭写方向，也关闭读方向。**但此时Socket资源没有释放**。



![image-20230528225341577](network_pic\image-20230528225341577.png)

![image-20230528225501583](network_pic\image-20230528225501583.png)

![image-20230528225617140](network_pic\image-20230528225617140.png)

> 心跳机制：为了保证在一段事件内若TCP连接没有任何通信，则进行尝试断开。
>
> Linux查看: `sysctl -a` 

> 设置心跳：
>
> - 直接调用 `sysctl` 的 `keepalive` 时间间隔；
> - 在服务端对每个 `socket opt` 配置心跳；
> - 在应用层设置心跳；
>

> 为什么要在应用层设置心跳：
>
> - `tcp keepalive` 报文 **可能被设备特意过滤或屏蔽**,如运营商设备；
> - `tcp keepalive` 由**内核**帮忙回复，**无法检测应用层状态**；如进程阻塞、死锁、TCP缓冲区满等情况；
> - 应用层心跳包 **可以顺便携带业务数据**；
> - `tcp keepalive` 不方便判断连接被关闭的原因；







## I/O多路复用模型

![image-20230528222147766](network_pic\image-20230528222147766.png)

![image-20230528222311705](network_pic\image-20230528222311705.png)

![image-20230528222439464](network_pic\image-20230528222439464.png)

![image-20230528222523694](network_pic\image-20230528222523694.png)

> `EPOLLONESHOT` **只会触发一次**，线程处理完成之后需要**重置**`epolloneshot`才能继续接收到事件。 
>
> 在**epoll**选择使用**`ET`**模式时，可以选择使用`EPOLLONESHOT`选项。
>
> 使用**`EPOLLONESHOT`**之后，
>
> - **非重入**：当文件描述符上的事件被触发时，该文件描述符就会**从epoll的就绪队列中移除**，直至显示地重新注册。
> - **单次触发**：文件描述符上的事件仅触发一次。即使有多个事件同时发生。
> - **线程安全**：可以确保同一时刻只有一个线程处理特定文件描述符上的事件，避免并发竞争和重复处理。
> - **控制并发**：比如在多线程环境中，只允许一个线程处理特定的事件。

![image-20230528222544371](network_pic\image-20230528222544371.png)

> - epoll 是线程安全的。**多个 **线程可以 **同时 **对 **同一个 **epoll实例进行事件的注册、删除和等待。
> - 而对于一个线程中多个epoll实例，则需要开发者自行保证线程的安全性。

> - Linux 中 epoll 并没有直接通过使用 `mmap` 共享空间。而是，epoll 使用 **内核与用户空间之间的内存映射**(mmap) 来传递就绪事件消息的。
> - 当应用程序调用epoll_create函数创建一个epoll实列时，**内核** 会为该实例 **分配一块内存空间**，**并将其映射到应用层程序的地址空间中**。
> - 该内存空间称为 **内核事件表**（eventpoll)，它存储了已注册的文件描述符和对应事件信息。内核并且会在其中记录文件描述符的就绪状态。
> - 当文件描述符的就绪状态发生变化时，**内核 **就会将就绪的文件描述符及相关事件信息写入到**内核事件表**中。(采用了红黑树使得查找效率提高，不再进行遍历)
> - 应用程序通过`epoll_wait`来等待就绪事件。在这个过程中，**内核 **会将内核事件表中的就绪事件信息 **拷贝** 到应用程序的用户空间缓冲区中。

> - epoll 是**异步**的。它允许应用程序在等待事件发生时，不需要阻塞线程，而是通过epoll_wait进行等待，并在有事件发生时得到通知。
> - 异步性的体现：
>   - **非阻塞**：使用epoll时，可以将文件描述符设置为非阻塞模式。在进行事件等待时，如果没有事件发生，epoll_wait会立即返回，而不会阻塞线程。这样可以使线程能够继续执行其他业务，提高并发性和效率。
>   - **事件通知**：epoll通过调用epoll_wait函数等待事件的发生，并在有事件发生时返回，从而使得应用程序可以通过该函数得到事件的通知，从而进行相应的处理。这种事件驱动的模式允许应用程序**异步地处理多个**连接或文件描述符的事件。
>   - **边缘触发**：在ET模式下，epoll只有状态变化时通知应用程序。这意味着应用程序需要处理边缘触发的事件，并在事件就绪时尽快处理，以免错过事件。

![image-20230528222614148](network_pic\image-20230528222614148.png)

> epoll不能监听普通文件，因为磁盘始终是处理就绪状态。





## 异步IO

![image-20230528222718060](network_pic\image-20230528222718060.png)

![image-20230528222742439](network_pic\image-20230528222742439.png)

> `io_uring` （I/O多路复用机制）。
>
> - 提供了一组新的系统调用，以及相应的用户空间API，用于提交和等待异步I/O操作完成。
> - 利用了Linux内核中的 `RingBuffer`来管理I/O请求和完成事件，实现 **零拷贝 **和 **零系统调用** 的特性。
> - 支持两种触发模式：LT,ET。
> - 需要一些必要的依赖和配置：如 liburing库 和 特定的内核选项。



## 网络编程模型

![image-20230528222916691](network_pic\image-20230528222916691.png)

![image-20230528222931311](network_pic\image-20230528222931311.png)

> 使用`SO_REUSEPORT`之后，
>
> - 支持多个进程或线程绑定到同一端口；
> - 当使用端口重用之后，当一个子进程绑定并监听端口时，即便有其他的子进程已经在监听该端口，也不会产生错误。这是因为内核允许多个套接字共享同一个端口，**但每个连接会被分配到不同的被动套接字**(当一个连接请求到达监听套接字时，内核就会在就绪队列中选择一个子进程来处理该连接请求，并创建一个新套接字与客户端进行通信)，从而避免了多个子进程竞争同一监听套接字。从而**可以避免惊群现象**。



## TCP & UDP

![image-20230528224424919](network_pic\image-20230528224424919.png)

![image-20230528224502644](network_pic\image-20230528224502644.png)

> **TLV**是一种常用的数据传输格式，**用于解决TCP粘包问题**。TLV代表Tag-Length-Value，它将数据分为三个部分：
>
> - **Tag**（标记）：用于标识数据的类型或含义，通常是一个**固定**长度的字段，用于**唯一标识**不同类型的数据。
>
> - **Length**（长度）：表示Value字段的长度，通常是一个**固定**长度的字段，用于指示Value字段的字节数。
>
> - **Value**（值）：实际的数据内容，可以是任意长度的字节序列。
>
> 通过使用TLV格式，发送方将数据按照一定规则划分为多个TLV结构，然后将这些TLV结构依次发送给接收方。接收方在接收到数据后，按照TLV格式解析数据，根据Tag和Length字段来区分和提取不同类型的数据。

![image-20230528224615787](network_pic\image-20230528224615787.png)













