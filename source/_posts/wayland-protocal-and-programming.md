---
title: wayland protocal and programming
date: 2012-10-01 21:30:29
tags:
---

Written in 2012 before wayland 1.0 released, most of them are outdated as in May 2017.

wayland是异步的面向对象的协议，所有请求都是在某个对象上的方法调用。每个对象实现一个接口，描述一系列操作。请求包含对象id，唯一标识服务器上的一个对象，还包括opcode，指出在对象上调用的方法。server向client发"事件"，每个事件都依附于一个对象。事件可以是出错信息，也可以是其他通知，事件包含对象id和opcode，client可以通过这些信息判断事件的类型。事件可以是对请求的响应(就像http的request和response一样，构成一个来回)，也可以是server自发的。

<!--more-->
# 1 简介

关于wayland的架构，请查看 wayland.freedesktop.org.

# 2 协议分析

wayland 协议采用xml描述，结构化较好，在研究wayland代码之前看一下这个xml文件有助于理清思路。先了解wayland协议中的几个概念。 
客户端(client): 需要在屏幕上绘图的普通程序
服务器(server, compositor): wayland协议的核心程序，负责多个客户端绘图的合成，并输出到显示器，下文中compositor和server含义相同。
请求(request): 由客户端发起的wayland通讯
事件(event): 服务器发起，客户端选择性接收的异步通知，本文有时用“通知”代替
枚举(enum)：定义了一些常量
接口(interface)：请求，事件，和枚举值的集合
表面(surface): 显示在屏幕上的一副图画，具有位置，大小，和像素内容
wayland架构中，有多种协议存在，client和server之间有协议，EGL和server之间有协议，server和shell之间也有协议，等等。这里只列出几个常用的，它们之间的关系如图：
 
![component protocols](/images/post/wayland-component-protocols.png)

## 2.1 client和server间的核心协议

wayland核心协议wayland.xml，位于wayland工程的protocol目录，协议名为"wayland".

### 2.1.1 wl\_display接口

实现该接口的对象(下文省略“实现该接口的对象”)为核心的全局单件对象，wayland协议内部使用。 
请求： 
bind: wl\_display\_bind(), 把client创建的对象绑定到服务器。
sync: wl\_display\_sync(), 要求服务器进行同步，返回wl\_callback，在wl\_callback对象上调用“done"请求。因为服务器是顺序执行各种请求的，sync保证以前的请求已经处理掉。
事件： 
error：严重的不可恢复的错误发生
global: 由服务器创建，并通知给客户端的全局对象。global在client初始连接时由server发布。比如：设备热插拔，断开链接，配置变更等。客户端可以使用"bind"请求绑定到一个global对象，随后获得一个global对象的句柄，客户端可以在这个句柄上调用相应请求
global\_remove: global 对象被移除的通知
delete\_id： 服务器确认删除了对象 id，client现在可以重用它了

### 2.1.2 wl\_callback接口

对callback的一个抽象，这个接口只有一个事件：
done: 完成事件

### 2.1.3 wl\_compositor接口

单件compositor对象，负责把多个surface合成为一个可以在显示器上显示的输出格式。 
请求:
create\_surface: 客户端请求compositor创建一个surface
create\_region： 客户端请求compositor创建一个region

### 2.1.4 wl\_shm\_pool接口

wl\_shm\_pool 对象封装了compositor和client之间共享的内存池。 通过 wl\_shm\_pool 对象，client 可以分配共享内存的wl\_buffer对象。这些对象共享底层的映射内存。 重用映射内存可以避免不断初始化和销毁内存的开销，在改变surface大小和小缓冲区操作比较多的时候很有用。  
请求：
create\_buffer： 从共享内存池里创建wl_buffer, 它从内存池的某个字节偏移开始，有高，宽，步长(stride，每行的字节数)等属性。
destroy： 销毁共享内存池

### 2.1.5 wl\_shm接口

共享内存缓冲区的管理。
请求： 
create\_pool：创建共享内存池，而共享内存池可以创建共享内存的wl\_buffer对象。这个请求带一个文件描述副fd参数，把该文件的内存映射做为共享内存池的备份内存。
事件:
format: 共享内存是什么格式的事件，允许两种格式， argb8888 和 xrgb8888

### 2.1.6 wl\_buffer接口

wl\_buffer提供了wl\_surface的内容，通过wl\_drm， wl\_shm 或类似的工厂接口创建。缓冲区具有高度和宽度，可以绑定到wl\_surface，但是client创建和更新内容的方式由工厂接口定义。  
请求：
destroy: 销毁缓冲区，使对象id无效
事件:
release: compositor释放缓冲区的事件，表明compositor不再使用该缓冲区

### 2.1.7 wl\_data\_offer接口

请求：
accept: 接受一种mime-type，说明client接受一种给定的mime-type，不接受用NULL表示。在drag and drop场景中作为反馈。
receive: 接收 mime_type
destroy: 销毁?
事件：
offer: 创建 wl\_data\_offer 对象后立即发出的事件，每个 mime_type 对应一个offer事件
2.1.8 wl\_data\_source接口

数据源。
请求：
offer: 把一种 mime-type 添加到 mime-type 集合中，告诉目的地，可以多次调用提供多种 mime-type 
destroy: 销毁数据源
事件：
target: 目标接受了 mime-type. 在目标接受了 pointer_focus 或者 motion 时发出事件，如果目标不接受 mime-type, 类型为 NULL
send: 请求另外一个client的数据，通过传输的文件描述符fd发送指定的 mime-type，然后关闭 fd
cancelled: 另外一个选择激活后，现有的选择被取消

### 2.1.9 wl\_data\_device接口

请求:
start_drag: 要求compositor代表本client开始一个 drag and drop 操作。source参数是提供数据传输的wl_data_source对象;origin参数是产生drag的wl_surface对象，client必须有激活的，匹配serial(另一个参数)的隐含grab;icon参数是可选的，是光标移动时的图标。最初，左上角的icon surface放置在光标的热点，往后的 surface.attach 请求可以移动相对位置。
set\_selection: 选择区域
事件：
data\_offer: 引入新的 wl_data_offer 对象，用在以后的 data\_device.enter 或 data_device.selection 事件中。这个事件之后，data_offer 对象立刻发出 data_offer.offer事件，描述它提供的 mime-types.
enter: 进入事件
leave: 离开事件
motion：移动事件
drop：放下事件
selection：选择事件，通知client有新的基于这个device和selection的wl\_data\_offer。data\_device.data\_offer 和 data\_offer.offer 事件在本事件之前立即发出，以引入 data offer 对象。selection事件发送给client后，client立即收到键盘焦点，如果client有键盘焦点，那么设置新的selection

### 2.1.10 wl\_data\_device\_manager接口

请求：
create\_data\_source: 创建数据源
get\_data\_device: 获取数据设备

### 2.1.11 wl\_shell接口

请求： 
get\_shell\_surface: 获得shell surface

### 2.1.12 wl\_shell\_surface接口

桌面风格的元数据接口， wl\_surface 实现了这个接口。服务器端，wl\_shell\_surface 对象随着 wl\_surface 对象的销毁而自动销毁。客户端，必须先调用 wl\_shell\_surface\_destroy，然后再销毁 wl\_surface 对象。 
请求:
pong: 客户端回应server发来的ping事件，必须回应，否则server会认为client出问题了
move: 移动
resize: 重置大小
set\_toplevel: 使surface最顶层
set\_transient: 显示surface,和某个现存的surface相联系，这种surface称为transient surface
set\_fullscreen： 使surface全屏。如果client指定了output参数，那么这个surface会全屏绘制到output上；如果client没有指定output参数，那么compositor有权根据某种策略作出决定 - 通常选择具有最大surface区域的output
set\_popup: 弹出式surface
set\_maximized	：最大化surface 
set\_title：设置surface的标题 
set\_class: surface class指出了应用的类型，class是*.desktop文件的文件名
事件：
ping: 服务端ping客户端，确保客户端能够接收事件，发送请求，客户端应回应pong请求：
configure：要求client重置surface大小，参数中的大小仅仅为建议，client可以忽略。client可以忽略所有的configure事件，除了最后一个。
popup_done: 弹出交互已完成。用户点了一个surface，它不属与拥有弹出surface的client，这时候，server发出popup_done事件。说的直白一点，就是用户点击了和弹出窗口不相关的程序的窗口。

### 2.1.13 wl\_surface接口

surface为显示在屏幕上的一副图画，具有位置，大小，和像素内容。 
请求：
destroy: 销毁surface，对象id无效
attach: 设置surface的缓冲区内容，x, y 参数是距离新缓冲区的左上角的位置
damage: 标记surface的某部分被破坏了。attach一个缓冲区后，damage请求用来描述一个新老缓冲区内容不同的区域，它需要重绘。坐标值是新缓冲区的位置。
frame: 请求下一帧显示时的通知回馈，每调用一次，通知只会发一次。这个请求可以用来调整重绘操作，驱动动画。
set_opaque_region: 设置surface的不透明区域，可以优化compositor重绘操作。这个操作不是必须的，但是把透明内容标记为不透明会导致重绘伪影(repainting artifacts)
set\_input\_region: 设置输入区域，接收鼠标，触摸事件
事件：
enter：surface进去output。每当surface创建，移动，重置大小的操作导致某部分进入output的scanout区域时，都会发出这个事件
leave: 和 enter 相对

### 2.1.14 wl\_input\_device接口

一组键盘、鼠标等设备，这个对象在启动时，或者被热插拔时发布，input_device组通常包括指针，键盘焦点和鼠标焦点。
请求：
attach：设置鼠标图像。
事件：
motion：鼠标移动事件
button：鼠标按键事件，包括点击，释放通知，点击的位置由最后的motion或pointer_focus事件决定
axis: 滚动和其他axis通知
key: 键盘事件
pointer_enter: 鼠标进入某个surface上的通知，client应响应这个事件，设置指针图像，因为默认情况是未定义的。
pointer_leave: 鼠标离开
keyboard_enter
keyboard_leave
touch\_down
touch\_up
touch\_motion
touch\_frame
touch\_cancel: compositor 认为这个touch流其实是一个全局的 gesture，那么释放这个事件。

### 2.1.15 wl\_output接口

output是compositor的输出区域，描述compositor的一部分位置信息。compositor工作在自己的坐标系统中，output是该坐标系中的一部分可见的矩形区域。通常output就是显示器显示的那部分compositor空间。wl_output对象在启动时或者屏幕热插拔时发布。
事件：
subpixel:
geometry:
mode

### 2.1.16 wl\_region接口

区域接口
请求：
destroy
add
subtract

## 2.2 EGL 和 compositor 间的DRM协议

wayland client和server都需要使用EGL, client使用EGL创建wayland窗口和Rendering Context，而 server 和 EGL 需要对 DRM 缓冲区进行通讯。这个协议定义在 mesa 代码目录下的 wayland-drm.xml 文件中，协议名为 "drm".

### 2.2.1 wl\_drm接口

wl\_drm由服务端创建，通过display的全局事件发布。
请求：
authenticate: 先调用 drmGetMagic(), 得到 magic 后，以此作为参数调用此函数，接下来会调用 drmAuthMagic(), DRIAuthConnection(). 这个函数必须在 create\_buffer() 之前调用。 
create\_buffer: 根据 DRM buffer 创建 wl_buffer,  DRM surface 必须有能用 flink ioctl 的名字。 
事件：
device：DRM 设备路径的通知，客户端应根据这个DRM设备路径创建本地缓冲区。只有该设备创建的缓冲区才能通过 DRM 对象的 create_buffer 传给server 
format：buffer的格式，xrgb8888, argb8888, rgb565... 
authenticated: authenticate 请求通过后发出的事件 

## 2.3 server(weston) 和 shell (tablet-shell) 间的tablet协议

这个协议定义了shell(可以理解为桌面)和server之间如何交互，概念有点类似于X11的窗口协议ICCCM，但是架构完全不同。协议定义在weston源码目录下的 tablet-shell.xml 文件中，协议名为"tablet". 协议的请求和事件的名字很直观。

### 2.3.1 tablet\_shell 接口

请求：
set\_lockscreen： 
set\_switcher： 
set\_homescreen： 
show\_grid： 
show\_panels： 
create\_client： 
事件：
show\_lockscreen 
show\_switcher 
hide\_switcher 

### 2.3.2 tablet\_client 接口

请求：
destroy 
activate 

## 2.4 协议到代码的转换

wayland代码中有个程序scanner，它会解析协议 xml 文件，产生三个协议文件:
协议实现文件 xxx-protocol.c 
client 头文件 xxx-client-protocol.h
server 头文件 xxx-server-protocol.h
client include xxx-client-protocol.h, 连接 libwayland-client, server include xxx-server-protocol.h，连接libwayland-server, 它们之间的关系下图所示：

![proto code generation](/images/post/proto-code-generation.png)

scaner产生的文件有标准的模板，请求和事件抽象为 wl\_message，接口对应 wl\_interface，接口中的每个请求对应的函数命名为 wl\_someinterface_request , 每个接口有相应的listener，如 wl\_someinterface\_listener, 里面存放一些函数指针，用来处理异步的“事件”。如 wl\_display\_listener, wl\_callback\_listener, wl\_shell\_surface\_listener 等。

# 3 几个客户端代码分析

## 3.1 simple-shm.c

这个例子展示了如何通过共享内存进行绘图，抽丝拨茧，得到如下核心流程:
1. 调用 wl\_display\_connect(...) ，获得 wl\_display 对象的指针. wl\_display 是全局对象，所有其他全局对象都从 wl\_display 而来。
2. 以wl\_display对象指针为参数，调用wl\_display\_add\_global\_listener(...)，注册全局对象监听器，在回调函数中，根据id和接口判断对象类型，调用wl\_display\_bind()，产生wl\_compositor, wl\_output，wl\_input\_device, wl\_shell, wl\_shm, wl\_data\_device\_manager等全局对象。
3. 调用一次 wl\_display\_iterate(...)，从server获取一次状态，得到一些global对象。
4. 如果对其他事件感兴趣，在相应的全局对象上注册监听器。这样做，是因为wayland协议是完全面向对象的，所有的事件都必须依附于对象。举个例子，如果对wl\_input\_device接口中的事件感兴趣，那么通过步骤 2 获得了wl\_input\_device对象指针后，调用 wl\_input\_device\_add\_listener(wl\_input\_device *, &input\_device\_listener, xxx); 在 input\_device\_listener中存放中实现各种事件处理函数的地址.
5. mmap一块地址，调用 wl\_shm\_create\_pool 创建一个内存池，在内存池对象上调用 wl\_shm\_pool\_create\_buffer(...)，得到一个 wl\_buffer
6. 在第2步得到的wl\_compositor对象上调用 wl\_compositor\_create\_surface(...) 得到一个 wl\_surface, 在该 wl\_surface 上调用 wl\_shell\_get\_shell\_surface(..) 获得 wl_shell_surface.
7. 调用 wl\_shell\_surface\_set\_toplevel(...)，设置上一步得到的wl\_shell\_surface为最顶层。
8. 在第5步得到的 wl\_buffer 和 shm 地址上直接绘图
9. 绘图完毕后调用 wl\_surface\_attach(...) 把 wl\_buffer 绑定到第五步得到的 wl\_surface 上，调用 wl\_surface\_damage(...) 告诉server，哪些区域已经更新了。
10. 调用 wl\_surface\_frame(...)获得一个 wl\_callback 对象，在该对象上调用 wl\_callback\_add\_listner(...)注册回调函数，请求下一帧显示时的通知回馈。每一帧都调用一次，在回调函数中继续绘图，这样client的窗口更新区域源源不断通知给server，server通过合成，把client窗口绘制在屏幕上。
11. 进入事件循环，不断调用 wl\_display\_iterate(...) 推动状态前进。
客户端对象产生的过程可以用下图表示：

![client object generation](/images/post/client-object-generation.png)

## 3.2 flower.c

wayland-demos工程里实现了一个libtoytoolkit.a库，实现代码主要是window.c，它实际上是对wayland协议的包装，客户端程序可以利用windows.c代码，简化编写客户端的过程。

1. 调用display\_create()初始化。涵盖上面的步骤1~7
2. 调用window\_create()创建窗口，设置窗口的属性，添加事件监听器。上面的步骤8
3. 调用display\_run()进入事件循环。上面的步骤9。该函数一旦发现epoll事件就绪，就从epoll事件中附带的data数据中取出task，调用run函数，执行相应的代码。那么wayland协议事件是如何转化为task的呢?task的定义很简单，包括两个成员，一个是把task串成链表的wl_list成员，另一个是函数指针run，指向具体的处理函数。 
最典型的一条路径是：
* 在display\_create()中，调用display\_watch\_fd(), 最后一个参数即为task，指向函数handle\_display\_data().
* 在display\_watch\_fd() 把最后一个参数task放入epoll_event的data.ptr指针中，然后调用epoll\_ctl函数，执行 EPOLL_CTL_ADD操作，把epoll\_event事件加入监听epoll列表
* 一个epoll事件就绪后，handle\_display\_data()被调用，而它最终会调用 wl\_display\_iterate()，处理网络连接上的数据

使用Gtk+或者Qt+等更上层的工具库编写wayland应用程序更简单，和X11上的客户端区别不大，因为它们做了很好的封装，不同显示服务器之间的差别被掩盖了。  

## 3.3 simple-egl.c

这个例子展示了如何使用 egl 和 OpenGL ES 2 绘图，创建窗口的流程和simple-shm基本一致。调用 eglCreateWindowSurface(...) 时，我们需要一个本地窗口，这个本地窗口通过 EGL wayland 平台实现库(mesa)中的 wl\_egl\_window\_create(surface, width, height)创建。

EGLSurface创建完成，通过OpenGL ES 2.0 API进行绘图，这部分超出本文范畴，请查阅相关文档。绘图完毕后，需要调用eglSwapBuffers() 把 back buffer 拷贝到 front buffer，进而在屏幕显示，这个过程中，mesa EGL 平台做了很多工作，以后章节会专门分析。

## 3.4 client如何创建窗口并绘制内容 
1. 在wl\_display\_add\_global\_listener的回调函数中，判断接口名称，调用wl\_display\_bind(...)得到wl\_compositor指针
2. 调用wl\_compositor\_create\_surface(...)，得到wl\_surface的指针，该surface即可理解为窗口
3. 使用cairo等工具在surface上绘制内容，如果使用libtoytoolkit，可以调用window\_get\_surface获得cairo\_surface\_t指针，在cairo_surface上绘图即可。
4. 使用wl\_surface\_attach()绑定wl_buffer, 用 wl\_surface\_damage 通知server绘图区域有更新

# 4 weston代码分析

## 4.1 启动

1. 调用 wl\_display\_create(), 创建wl\_display
2. 调用 wl\_display\_get\_event\_loop(), 创建  wl\_event\_loop
3. 调用 wl\_event_loop_add_signal(), 把signal加入 event loop 的监听源
4. 设置 SIGSEGV 的信号处理函数，一旦发生，跳出主循环，程序结束
5. 根据传入的参数或环境变量决定weston使用哪个后端，可能的backend包括 6. wayland-backend.so, x11-backend.so, openwfd-backend.so(TV的一个平台) 或 drm-backend.so
7. 根据传入的参数或环境变量选择shell接口，默认为 desktop-shell.so，通过 -s 参数选择不同的backend
8. 调用backend 的初始化函数 backend_init(), 这个函数在另一个动态库中，使用 dlopen() 打开。对于drm后端来说, backend_init 代码在 compositor-drm.c 中.
9. 调用shell的初始化函数 shell_init(), 这个函数在另一个动态库中，使用 dlopen() 打开.
10. 调用 wl\_display\_add\_socket(), 把unix domain socket描述符加入event loop事件监听源
11. 调用 weston\_compositor\_wake()，copositor开始工作
调用 wl\_display\_run(), 进入主循环, 不断调用 wl\_event\_loop\_dispatch()，派发事件
12. 如果主循环由于某种原因退出，进行一些清理性工作
    * eglUnbindWaylandDisplayWL(). (在mesa for wayland库中)
    * wl\_event\_source\_remove()
    * wl\_display\_destroy()

## 4.2 client如何产生

在server代码中，client 使用 wl\_client 数据结构标识，存放在wl\_display结构中(client和server对wl\_display的定义是不同的, 但它是不透名结构，库的用户看不到)。client是如何生成的呢?
1. server调用 wl\_display\_add\_socket()，把domain socket 加入到监视列表，设置回调函数为 socket\_data()
2. client调用 wl\_display\_connect()后, server端domain socket上有数据，socket\_data() 得到调用 
    * 在domain socket上调用accept, 得到client\_fd
    * 调用 wl\_event\_loop\_add\_fd()，把 client_fd 加入监听列表
    * 调用 wl\_client\_create(display, client\_fd) 创建 wl\_client，最终会调用wl\_client\_add\_object(...)创建一个 wl\_display对象
3. 至此，client得到wl\_display对象
4. client执行后续操作参考

## 4.3 client和server的交互

这里，我们把client和server调用流程的片段整合成一张交互图，就很容易理解wayland的整个协议流程了。如前所述，client在server中的状态以 wl\_client 结构表示，client启动之初调用的第一个函数是 wl\_display\_connect，得到全局唯一的 wl\_display 对象，后续操作在 wl\_display 对象上进行。产生其他全局对象的调用方法是 wl\_display\_bind(...)。那么为了确保 wl\_display\_bind(...)能够顺利地绑定一个全局对象，server端必须注册了接收wl\_display\_bind请求的回调函数。


![client server interaction](/images/post/client-server-interaction.png)

举个例子，假设server要接收 compositor 全局对象的绑定请求，server必须在初始化时做一些准备工作，调用 wl\_display\_add\_global(display, &wl\_compositor\_interface, ..., compositor\_bind)，注册一个 compositor 接口类型的回调函数compositor\_bind. 一旦client调用 wl\_display\_bind()请求绑定一个 wl\_compositor 对象，那么 server 会调用compositor\_bind(...)函数，在这个函数里调用 wl\_client\_add\_object(...)， 创建 wl\_compositor 对象，并返回给client.

server端每调用一个 wl\_client\_add\_object(&wl\_some\_interface)，就意味着client可以得到wl\_some\_interface对象，随后 client 可以在 wl\_some\_interface 对象上调用相应函数。wayland 是一个面向对象的协议，通过这种调用方式和实现机制，保证了任何操作都在对象上进行。

## 4.4 compositor坐标系

compositor程序维护一张场景图，场景图由一个或多个output组成，每个output对应一个connector。compositor把所有的output宽度计算出来，从左到右一字排开，称为compositor坐标系。 渲染的时候，compositor计算出需要重绘的区域，得到compositor坐标，然后把compositor坐标转化成窗口本地坐标，再把这些窗口本地坐标规约化，成为[-1, 1]区间的模型坐标，同时把纹理坐标规约成[0,1]，最后把点的位置和纹理坐标送入GPU，调用glDrawElements开始渲染。

## 4.5 动画

TODO

## 4.6 weston和tablet-shell

TODO

# 5 回顾

前面章节分析了wayland协议及相关代码，现在回过头来，理清几个概念，以便更好地理解wayland架构。

## 5.1 缓冲区

client和server共享绘图缓冲区的信息，避免内存拷贝，优化性能。client可以使用两种类型的缓冲区进行绘图，一种是wl\_shm，基于Linux的内存共享机制，另一种是wl\_drm，使用GBM共享GPU存储。wl\_shm用的是系统内存，wl_drm用的是VRAM. 

无论哪种缓冲区，client 和 server 间均无需拷贝绘图内容，但是  server 需要把 wl\_shm 内存拷贝到GPU (glTexImage2D, glTexSubImage2D)，而wl_drm不存在任何拷贝，因为本来就在GPU中，所以wl_drm绘图较快，没有内存拷贝，client和server均在GPU存储上操作，性能好。要使用wl\_drm，GPU需支持DRM, Mesa需要实现EGL wayland 平台和 EGL DRM平台。

## 5.2 wayland, egl 和 gles2

上文把分析的重点放在wayland本身的代码上，但是通过介绍，我们发现要完美地运行wayland，还必须其他软件库的支持，其中最重要的就是Mesa，它提供了EGL， GLES2的实现，同时也包括了平台中立的内存管理库GBM. wayland协议本身并不复杂，之所以得到业界的重视，一个很重要的原因在于它得到了绝大多数主流软件库的支持，包括Mesa, Qt, Gtk, Clutter, EFL等。关于wayland, egl, gles2之间的关系，Pekka Paalanen 有篇很好的文章(http://ppaalanen.blogspot.co.uk/2012/03/what-does-egl-do-in-wayland-stack.html)，这里借用一下他的图片。

![EGL mesa wayland arch](/images/post/EGL-Mesa-Wayland-arch.png)

wayland需要eglCreateImageKHR 扩展把 wl\_buffe r转化为 EGLImage，然后用 glEGLImageTargetTexture2DOES 把EGLImage 转化为texture，这是通过Mesa EGL 实现的。

Typically, hardware enabling includes modesetting/display and EGL/GLES2. On top of that, Wayland needs a way to share buffers efficiently between processes. There are two sides to that, the client side and the server side.

On the client side we've defined a Wayland EGL platform. In the EGL model, that consists of the native types (EGLNativeDisplayType, EGLNativeWindowType and EGLNativePixmapType) and a way to create those types. In other words, it's the glue code that binds the EGL stack and its buffer sharing mechanism to the generic Wayland API. The EGL stack is expected to provide an implementation of the Wayland EGL platform. The full API is in the wayland-egl.h header. The open source implementation in the mesa EGL stack is in wayland-egl.c andplatform_wayland.c.

Under the hood, the EGL stack is expected to define a vendor-specific protocol extension that lets the client side EGL stack communicate buffer details with the compositor in order to share buffers. The point of the wayland-egl.h API is to abstract that away and just let the client create an EGLSurface for a Wayland surface and start rendering. The open source stack uses the drm Wayland extension, which lets the client discover the drm device to use and authenticate and then share drm (GEM) buffers with the compositor.

The server side of Wayland is the compositor and core UX for the vertical, typically integrating task switcher, app launcher, lock screen in one monolithic application. The server runs on top of a modesetting API (kernel modesetting, OpenWF Display or similar) and composites the final UI using a mix of EGL/GLES2 compositor and hardware overlays if available. Enabling modesetting, EGL/GLES2 and overlays is something that should be part of standard hardware bringup. The extra requirement for Wayland enabling is the EGL_WL_bind_wayland_display extension that lets the compositor create an EGLImage from a generic Wayland shared buffer. It's similar to the EGL_KHR_image_pixmap extension to create an EGLImage from an X pixmap.

The extension has a setup step where you have to bind the EGL display to a Wayland display. Then as the compositor receives generic Wayland buffers from the clients (typically when the client calls eglSwapBuffers), it will be able to pass the struct wl_buffer pointer to eglCreateImageKHR as the EGLClientBuffer argument and with EGL_WAYLAND_BUFFER_WL as the target. This will create an EGLImage, which can then be used by the compositor as a texture or passed to the modesetting code to use as an overlay plane. Again, this is implemented by the vendor specific protocol extension, which on the server side will receive the driver specific details about the shared buffer and turn that into an EGL image when the user calls eglCreateImageKHR.


## 5.3 egl和gles2的buffer

说到底，compositor 最重要的工作就是 buffer 的管理，render 部分由客户端自己完成了(cairo, cairo-gl, cairo-gles, gles, clutter). 在 gles 和 egl 中，有很多种类型的buffer，让人头晕，那么这些这些缓冲区都在哪里，看看下面这张图，有助于理清思路：

![gles egl buffer](/images/post/gles-egl-buffer-residence-and-init-sequence.png)

使用gles2绘图的client的初始化过程大致是这样的：
* client 先要通过本地窗口系统获得一个窗口，把它作为参数传入 eglCreateWindowSurface()，获得 EGLSurface, 类型为 window surface，位于GPU的VRAM中。 
* client 调用 eglMakeCurrent()，把 EGLSurface 和 EGLDisplay（通过 eglGetDisplay()得到，图中未画）绑定到一个 rendering context（通过 eglCreateContext()创建，图中未画）. 
* client 使用 gles2 函数绘图，所有的操作都作用在egl 提供的 rendering context上，最后调用 glDrawElements() 或 glDrawArrays()告诉GPU开始绘图。OpenGL ES 2 只支持double buffer, 所有所有绘图操作都作用于 back bufffer上。 
* client 调用 eglSwapBuffers()，切换 back buffer 和 front buffer 的内容，绘图内容呈现在显示器上。 

gles2 中的几个copy像素的函数：

* glTexImage2D()：把系统内存中存放像素的缓冲区拷贝到texture，System RAM => VRAM 
* glReadPixels()： 把color buffer的内容(来自back buffer)拷贝到系统内存, VRAM => System RAM 
* glCopyTexImage2D()： 把color buffer的内容拷贝到texture，VRAM => VRAM 

# 6 其他相关知识

了解了wayland框架，我们打算自己实现一个不依赖 X 的 compositor, 这就需要熟悉 linux graphics stack 中的其他组件，我们来看一下DRM, KMS 和 GBM。同时，我们也必须熟悉输入框架 evdev 和 设备框架 udev .

## 6.1 DRM, KMS and GBM

DRM(Direct Rendering Manager) 是 DRI (Direct Rendering Infrastructure) 的内核实现部分，实现Linux 2D 和 3D 图形加速。DRI 的提出是为了加速 X Server，所以事实上 X 是 DRM 的主要用户，可以在 X 代码里看到 DRM 的身影。现在 Wayland  使用了 DRM KMS(kernel mode settings)，但不包括和 X 相关的 DRI 用户空间那部分代码. KMS 是 DRM 的一部分，在内核态实现了显示模式(比如分辨率，行刷新率，场刷新率等)的设置。因为模式配置在内核中实现，不需要用户态的 X 服务器，所以控制台和桌面之间的切换就不会闪烁。

DRM 暴露给用户空间的是一系列 ioctl 接口，libdrm 是用户态的 DRM 库，包装了 这些 ioctl 接口。 我们通过几个例子来了解一下怎么操作 DRM.

### 6.1.1 kms-basic, 了解一下概念

    fd = open("/dev/dri/card0", O_RDWR);
    resources = drmModeGetResources(fd);

打开 DRM 设备，根据drm设备描述符获得 drmModeRes. drmModeRes 结构是后续操作的起点，寻找 connector 和 encoder 的过程从这个结构开始。下图展示了DRM的核心数据结构，图中的箭头是图形数据的流向，和代码中寻找 connector 和 encoder 的过程正好相反。 

![drm kms core data strucuture](/images/post/drm-kms-core-data-structures.png)

    for(i=0; i < resources->count_connectors; ++i){ 
		connector = drmModeGetConnector(fd, resources->connectors[i]);
		if(connector != NULL){
			fprintf(stderr, "connector %d found\n", connector->connector_id);
			if(connector->connection == DRM_MODE_CONNECTED
				&& connector->count_modes > 0)
				break;
			drmModeFreeConnector(connector);
		}
		else
			fprintf(stderr, "get a null connector pointer\n");
	}
	if(i == resources->count_connectors){
		fprintf(stderr, "No active connector found.\n");
		goto free_drm_res;
	}
	mode = connector->modes[0];
	fprintf(stderr, "(%dx%d)\n", mode.hdisplay, mode.vdisplay);

遍历 resource->connector 数组，寻找第一个连接 ( DRM_MODE_CONNECTED )的 connector，通过 connector->modes[0] 获得 drmModeModeInfo.

	for(i=0; i < resources->count_encoders; ++i){ 
		encoder = drmModeGetEncoder(fd, resources->encoders[i]);
		if(encoder != NULL){
			fprintf(stderr, "encoder %d found\n", encoder->encoder_id);
			if(encoder->encoder_id == connector->encoder_id);
				break;
			drmModeFreeEncoder(encoder);
		} else
			fprintf(stderr, "get a null encoder pointer\n");
	}
	if(i == resources->count_encoders){
		fprintf(stderr, "No matching encoder with connector, shouldn't happen\n");
		goto free_drm_res;
	}

在所有 encoder (resource->encoders 数组) 中，寻找和已连接的那个 connector 对应的 encoder.

    kms_create(fd, &kms_driver);
    unsigned bo_attribs[] = { 
        KMS_WIDTH,   mode.hdisplay,
        KMS_HEIGHT,  mode.vdisplay,
        KMS_BO_TYPE, KMS_BO_TYPE_SCANOUT_X8R8G8B8,
        KMS_TERMINATE_PROP_LIST
    };
    kms_bo_create(kms_driver, bo_attribs, &kms_bo);
    kms_bo_get_prop(kms_bo, KMS_PITCH, &pitch);

创建kms_driver, 创建bo(buffer object), 获得该bo的pitch(buffer每行字节数) .

    kms_bo_map(kms_bo, &map_buf);
    draw_buffer(map_buf, mode.hdisplay, mode.vdisplay, pitch); 

把bo映射到用户空间, 现在map_buf 存放该内存的起始地址，以后可以在这块buffer中任意绘图.

    orig_crtc = drmModeGetCrtc(fd, encoder->crtc_id); 

调用 drmModeGetCrtc 获得原来的模式设置，以备程序退出时恢复。 

    kms_bo_get_prop(kms_bo, KMS_HANDLE, &bo_handle); 
    drmModeAddFB(fd, mode.hdisplay, mode.vdisplay, 24, 32, pitch, bo_handle, &fb_id);
    drmModeSetCrtc(  
			fd, encoder->crtc_id, fb_id, 
			0, 0, 	/* x, y */ 
			&connector->connector_id, 
			1, 		/* element count of the connectors array above*/
			&mode);

获得 bo 的 handle，传入 drmModeAddFB，得到 Frame Buffer 的 id (fb_id). 调用 drmModeSetCrtc 设置显示模式，输出与 FB 关联的 BO 缓冲区内容(map_buf) 。至此，图形输出到显示器上。值得注意的是，可以设置Crtc 的 x, y值, 输出 FB 的部分内容，称为 scan out buffer.

    getchar()
    ret = drmModeSetCrtc(fd, orig_crtc->crtc_id, orig_crtc->buffer_id,
					orig_crtc->x, orig_crtc->y,
					&connector->connector_id, 1, &orig_crtc->mode);

等待用户按回车，恢复原来的显示模式，退出程序。

### 6.1.2 kms-pageflip, DRM 的double buffer机制

kms-basic 例子在同一个 BO 上，不断绘制图像。如果绘图速度较慢，会看到部分刷新的情况。kms page flip 是一种double buffer 机制，避免了部分刷新带来的闪烁问题。它的用法是用户创建两个Frame Buffer， 每个Frame Buffer 分别关联一个 BO，绘图函数在不同的 BO 上操作，程序记录下当前使用的 frame buffer id, 当下一个 vblank 刷新周期到来时，使用 drmModePageFlip 函数把另外一个 frame buffer 送给 crtc，不断循环，实现两个 frame buffer 之间的切换。这个例子还使用了select实现多路监听。

	...

使用 kms_bo_create 创建两个bo, 使用 drmModeAddFB 创建两个 FB，每个 FB 关联一个 bo.

    struct termios old_tio, new_tio; 
    tcgetattr(STDIN_FILENO,&old_tio);
    new_tio = old_tio;
    new_tio.c_lflag &= (~ICANON & ~ECHO);
    tcsetattr(STDIN_FILENO, TCSANOW, &new_tio);

关闭终端stdin的行缓冲，这样按任意一个键，不用按回车，程序就能接收到。

    while(1){
        struct timeval timeout = { 
            .tv_sec = 3, 
            .tv_usec = 0 
        };
    fd_set fds;

    FD_ZERO(&fds);
    FD_SET(STDIN_FILENO, &fds);
		FD_SET(fd, &fds);
		ret = select(max(STDIN_FILENO, fd) + 1, &fds, NULL, NULL, &timeout);

		if (ret <= 0) {
			continue;
		} else if (FD_ISSET(STDIN_FILENO, &fds)) {
			char c = getchar();
			if(c == 'q' || c == 27)
				break;
		} else {
			/* drm device fd data ready */
			ret = drmHandleEvent(fd, &evctx);
			if (ret != 0) {
				fprintf(stderr, "drmHandleEvent failed: %s\n", strerror(errno));
				break;
			}
		}
	}

这是程序的主循环，使用 select 把 stdin 标准输入加入监听，这样终端有输入，select 就会返回ready，这里按 ESC 或 q 推出程序。使用 drmHandleEvent 处理 DRM 设备描述符上的事件, evctx 是 drmEventContext 结构的实例，evctx.page_flip_handler 指向page flip事件到来后的处理函数。

	void page_flip_handler(int fd, unsigned int frame,
		  unsigned int sec, unsigned int usec, void *data)
	{
		struct flip_context *context;
		unsigned int new_fb_id;
		struct timeval end;
		double t;

		context = data;
		if (context->current_fb_id == context->fb_id[0])
			new_fb_id = context->fb_id[1];
		else
			new_fb_id = context->fb_id[0];
			
		drmModePageFlip(fd, context->crtc_id, new_fb_id,
				DRM_MODE_PAGE_FLIP_EVENT, context);
		context->current_fb_id = new_fb_id;
		context->swap_count++;
		if (context->swap_count == 60) {
			gettimeofday(&end, NULL);
			t = end.tv_sec + end.tv_usec * 1e-6 -
				(context->start.tv_sec + context->start.tv_usec * 1e-6);
			fprintf(stderr, "freq: %.02fHz\n", context->swap_count / t);
			context->swap_count = 0;
			context->start = end;
		}
	} 

回调函数 page_flip_handler 做的事情很简单，取出当前的 fb id, 得到下一个 fb id, 调用 drmModePageFlip，把下一个 fb 设置到 crtc，由此实现两个frame buffer 的切换。另外，程序实现了频率计算，可以看到，这个频率和 vblank 刷新频率相同。

	 ret = drmModeSetCrtc(fd, orig_crtc->crtc_id, orig_crtc->buffer_id,
					orig_crtc->x, orig_crtc->y,
					&connector->connector_id, 1, &orig_crtc->mode); 
	tcsetattr(STDIN_FILENO,TCSANOW,&old_tio); 

主循环退出后，做一些清理工作，包括回复以前的显示模式模式，以及调用 tcsetattr 恢复以前的终端设置。

### 6.1.3 kms-gles2, 使用EGL和GBM

前两个例子绘图用的是 cairo，如果要实现绚烂的动画效果，cairo 是不能胜任的，因为它比较低效，没有使用GPU加速(不包括cairo-gl, cairo-gles 等backend)。kms-gles2这个例子使用 EGL 和 OpenGL ES 2.0 实现 3D 动画。选用 OpenGL ES 2.0 是因为很多嵌入式平台都只支持OpenGL ES，而不支持 OpenGL。

Open GL ES 只负责渲染，不能和本地窗口系统进行交互，这部分是由 EGL 完成的。EGL是一个平台无关的库，为 OpenGL ES 提供和窗口系统的连接，包括一个渲染上下文(Rendering Context)和一个 EGLSurface。上两个例子中，被渲染的 buffer 是 KMS 函数创建并映射到用户空间的，但是 OpenGL ES 绘图需要的是一个 EGLSurface，怎么把KMS Buffer转化成 EGLSurface 呢? GBM 派上了用场。GBM(Generic Bufffer Manager)是mesa中用来管理内存的平台无关库。gbm_surface 是最近添加的接口，有了这个接口，我们可以用 gbm_surface 创建 egl window surface, 让 egl 负责缓冲区的分配，用户负责缓冲区的渲染(使用 kms pageflip). GBM还提供了锁住和释放缓冲区的函数。gbm_surface_lock_front_buffer() 和  gbm_surface_release_buffer(). 值得注意的是，编译mesa的时候，必须使用 --enable-gbm 开关才能把 GBM 编译进去。

废话不多说，来解释一下关键代码。

	gbm = gbm_create_device(fd); 
	if (gbm == NULL) {
	   fprintf(stderr, "couldn't create gbm device\n");
	   return EXIT_FAILURE;
	}

	display = eglGetDisplay(gbm);
	if(display == EGL_NO_DISPLAY){
		fprintf(stderr, "eglGetDisplay() failed: %d\n", eglGetError());
		return EXIT_FAILURE;
	}

通过 gbm_create_device 创建一个 gbm device，把它传入 eglGetDisplay，得到 EGLDisplay.

	...
	gbm_surface = gbm_surface_create(gbm, mode.hdisplay, mode.vdisplay,
				GBM_BO_FORMAT_XRGB8888,
				GBM_BO_USE_SCANOUT | GBM_BO_USE_RENDERING);
	surface = eglCreateWindowSurface(display, config, gbm_surface, NULL);
	
	if (!eglMakeCurrent(display, surface, surface, context)) {
	   fprintf(stderr, "failed to make context current\n");
	   return EXIT_FAILURE;
	}

其他egl初始化过程省略。调用 gbm_surface_create 创建一个 gbm surface，用它做参数传入 eglCreateWindowSurface，这样我们就得到了 EGLSurface，调用 eglMakeCurrent 把 EGLDisplay， EGLSurface 和 EGLContext 绑定起来。至此，EGL 初始化完成，以后 OpenGL ES 的绘图操作将会渲染到 EGLSurface 相应的 back buffer中，调用 eglSwapBuffers，back buffer 的内容和 front buffer交换，图形呈现在显示屏上。

	gbm_bo = gbm_surface_lock_front_buffer(gbm_surface);
	handle = gbm_bo_get_handle(gbm_bo).u32;
	stride = gbm_bo_get_pitch(gbm_bo);

	drmModeAddFB(fd,
		      mode.hdisplay, mode.vdisplay,
		      24, 32, stride, handle, &fb_id);

	drmModeSetCrtc(fd, encoder->crtc_id, fb_id, 0, 0,
			&connector->connector_id, 1, &mode);



调用 gbm_surface_lock_front_buffer 得到 gbm buffer object，然后调用 gbm_bo_get_handle 得到 gbm bo 的handle，把handle传入 drmModeAddFB 得到Frame Buffer 的 fb_id，最后调用 drmModeSetCrtc 设置显示模式.

OpenGL ES 绘图相关的过程略。

### 6.1.4 kms-gles2-pageflip

前一个例子运行起来可以明显看到画面的闪烁，原因在于定时器刷新周期和显示刷新不匹配，本例将着手解决这一问题，方法介绍过了，使用 kms pageflip。

![egl gbm surface page flip](/images/post/egl-gbm-surface-page-flip.png)

## 6.2 evdev

TODO

## 6.3 udev

TODO
 