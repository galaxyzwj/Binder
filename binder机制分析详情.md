## 1.RPC(远程过程调用)
Binder机制的目的就是为了实现远程过程调用，即进程A去调用进程B的某个函数，它是在进程间通信(IPC)的基础上实现的. 

RPC的一个应用场景如下:  

A进程想去打开LED，它会调用led_open,然后调用led_ioctl,但是如果A进程并没有权限去打开驱动程序呢？  
 - 假设此时有一个进程B有权限去操作LED驱动程序，那么进程A可以通过如下方式来操作Led驱动程序：  

![image](http://note.youdao.com/yws/public/resource/7797550601ea4ad1574551a6d9bd8ae6/xmlnote/E714C475BADA400B9DEF7C79D1089AE4/5385)

整个过程就好像A程序直接来操纵LED一样，这就是所谓的RPC`(在IPC的基础上做了一些数据封装打包，IPC是基础，它负责数据传输)`。  
这个过程涉及到了IPC(进程间通信)的三大要素，
 - 源       ===> 进程A
 - 目的     ===> 进程B
 - 数据     ===>打包的数据，双方约定好了的数据格式的buffer

## 2.Binder机制实现的RPC
Binder机制采用的是`CS架构`，提供服务的进程称为`server进程`,访问服务的进程称为client进程，server进程和client进程的通信需要依靠内核中的binder驱动来进行。同时binder系统提供了一个上下文的管理者`servicemanager`，server进程可以向servicemanager注册服务，然后client进程可以通过向servicemanager查询服务来获取server进程注册的服务。  
回到上面的例子，A进程想操作LED，它可以通过将B进程的某个函数(事先头文件中约定好)的代号通过IPC发送给B进程，通过B进程来间接的操作LED，==但是如果A进程不知道可以通过哪个进程来间接的操作LED呢，它应该将打包封装好的数据发送给谁呢？这里就引入了Binder系统的大管家servicemanager==。首先B进程向servicemanager注册LED服务，然后我们的A进程就可以通过向servicemanager查询LED服务，就会得到一个handle，这个handle就是指向进程B的，这样进程A就知道把数据包(约定好的数据格式的buffer)发送给哪个进程就可以间接的操作LED了。在这个例子中B进程就是server进程，A进程就是client进程。

总结一下：  
在Binder系统中主要涉及到一下四个东西：  
- a.client进程  
    1.调用server进程哪一个函数，server里的函数编号  
    2.传什么参数过去，参数放在IPC的buffer里面  
    3.返回值，确定是否成功，同上，双方约定好格式即可
- b.server进程
- c.servicemanager进程
- d.client、server、servicemanager三者之间的通信


![image](http://note.youdao.com/yws/public/resource/7797550601ea4ad1574551a6d9bd8ae6/xmlnote/7C4DBD9F0EA846BBA10B1DB58B3E6DAC/5737)  



## 3.Binder系统的源代码简单分析
在Android源码里面有一些C语言写的binder应用程序：   
>framework/native/cmds/servicemanager/servicemanager.c   
>framework/native/cmds/servicemanager/binder.c   
>framework/native/cmds/servicemanager/binder.h   
>framework/native/cmds/servicemanager/bctest.c   

### 3.1 servicemanager.c
framework/native/cmds/servicemanager/servicemanager.c  
```cpp
a.  bs = binder_open(128*1024);   // 打开binder驱动
b.  binder_become_context_manager(bs) // 告诉binder驱动，它是servicemanager
c.  binder_loop(bs, svcmgr_handler); 
        c.1 res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr); // 读数据
        c.2 res = binder_parse(bs,0,readbuf,bwr.read_consumed,func); 
            // 解析
            // 处理:svcmgr_handler
                        SVC_MGR_GET_SERVICE/SVC_MGR_CHECK_SERVICE  // 获取服务
                        SVC_MGR_ADD_SERVICE                        // 注册服务
            // 回复
```    
### 3.2 bctest.c
framework/native/cmds/servicemanager/bctest.c   
注册服务的过程：
``` cpp
a. binder_open
b. svcmgr_publish(bs, svcmgr, argv[1], &token);  
        binder_call(bs, &msg, &reply, 0, SVC_MGR_ADD_SERVICE)    
                        // 含有服务的名字
                              //它会含有servicemanager回复的数据
                                      // 0表示servicemanager
                                         //code
                                             表示要调用servicemanager中的“add service函数”
```   
获取服务的过程：
```cpp
a. binder_open
b. handle = svcmgr_lookup(bs, svcmgr, "alt_svc_mgr");
        binder_call(bs, &msg, &reply, 0, SVC_MGR_ADD_SERVICE)    
                        // 含有服务的名字
                              //它会含有servicemanager回复的数据,表示提供服务的进程
                                      // 0表示servicemanager
                                         //code
                                             表示要调用servicemanager中的“get service函数”

```   
### 3.3 binder.c (封装好的C函数)  
binder_call分析 （实现一个远程调用）  
```cpp
int binder_call(struct binder_state *bs,
                struct binder_io *msg, struct binder_io *reply,
                uint32_t target, uint32_t code)

```
- 向谁发送数据==> target   
- 调用哪个函数==> code   
- 提供什么参数==> msg
- 返回值==>       reply   

binder_call如何使用呢？   
1.构造参数，参数放在buffer[xxx]里面, 用结构体 binder_io来描述buffer   
2.res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);向驱动中写入数据  


==**综上所述**==:   
#### 如何写App   
##### 1. client   
- binder_open
- 获得服务的handle
- 构造参数：binder_io
- 调用binder_call, 函数三大参数为 handle(哪个服务进程)， code(哪个函数)， binder_io(参数)
- 分析返回binder_io,取出返回值        

##### 2. server    
- binder_open
- 注册服务
- ioctl读取binder中的数据包，由client发送过来的，handle能识别出来是否接受数据
- 解析数据 binder_write_read结构体
- 根据code，决定调用哪个函数(从binder_io中取出参数作为函数的参数向)
- 把返回值转换为binder_io,发给client














我们可以参照这些程序，基于Android内核，在linux上实现一个Binder机制RPC的程序来理解使用Binder机制实现进程间通信的整个函数调用过程。  



