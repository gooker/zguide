# 可靠的请求-回复模式

[**高级**-请求-回复]涵盖了ZeroMQ的请求-回复模式的高级用途,并附有工作实例.本章探讨了可靠性的一般问题,并在ZeroMQ的核心请求-回复模式之上建立了一套可靠的消息传递模式.

在本章中,我们主要关注用户空间的请求-回复*模式,这些可重复使用的模式可以帮助你设计自己的ZeroMQ架构.

* *懒惰的海盗* 模式:来自客户端的可靠请求-回复
* *简单的海盗* 模式:使用负载均衡进行可靠的请求-回复.
* *偏执的海盗* 模式:可靠的请求-回复与心跳.
* *Majordomo* 模式:面向服务的可靠队列
* *泰坦尼克号* 模式:基于磁盘/断开连接的可靠队列
* *双星* 模式:主-备份服务器故障切换
* *Freelance* 模式:无borkers的可靠请求-回复

## 什么是"可靠性"？

大多数谈论"可靠性"的人并不真正了解他们的意思.我们只能从失败的角度来定义可靠性.也就是说,如果我们能够处理某一组定义明确的,可以理解的故障,那么对于这些故障来说,我们就是可靠的.没有更多,也没有更少.因此,让我们看看在一个分布式ZeroMQ应用程序中可能出现的故障原因,大致上按概率降序排列.

* 应用程序代码是最糟糕的犯罪者.它可能会崩溃并退出,冻结并停止对输入的响应,对其输入来说运行得太慢,耗尽所有内存,等等.

* 系统代码--比如我们使用ZeroMQ编写的borkers--会因为与应用程序代码相同的原因而死亡.系统代码*应该*比应用程序代码更可靠,但它仍然会崩溃和烧毁,特别是如果它试图为缓慢的客户端排定消息队列,就会耗尽内存.

* 消息队列可能会溢出,通常是在系统代码中,这些代码已经学会粗暴地处理慢速客户端.当一个队列溢出时,它开始丢弃消息.所以我们会得到"丢失"的消息.

* 网络可能发生故障(例如,WiFi被关闭或超出范围).在这种情况下,ZeroMQ会自动重新连接,但在此期间,消息可能会丢失.

* 硬件可能发生故障,并带走所有运行在该机器上的进程.

* 网络可能会以异乎寻常的方式发生故障,例如,交换机上的一些端口可能死亡,网络的这些部分就无法访问.

* 整个数据中心可能被雷电,地震,火灾或更普通的电力或冷却故障所袭击.

要使一个软件系统完全可靠地应对*所有*这些可能的故障,是一项极其困难和昂贵的工作,这超出了本书的范围.

因为上述列表中的前五种情况涵盖了大公司以外的99.9%的现实需求(根据我刚刚进行的一项高度科学的研究,它还告诉我78%的统计数据是当场编造的,此外,永远不要相信我们自己没有伪造的统计数据),这就是我们要研究的内容.如果你是一家大公司,有资金用于后两种情况,请立即联系我的公司! 我的海滨别墅后面有一个大洞,等待着被改造成一个行政游泳池.

## 设计可靠性

因此,简单粗暴地说,可靠性就是"在代码冻结或崩溃时保持事物正常工作",这种情况我们简称为"死亡".然而,我们想要保持正常工作的东西比单纯的消息更复杂.我们需要对ZeroMQ的每个核心消息传递模式进行研究,看看如何让它在代码死亡时也能正常工作(如果可以的话).

让我们逐一来看看.

* 请求-回复:如果服务器死了(在处理一个请求的时候),客户端就会发现,因为它不会得到一个回复.然后,它可以一气之下放弃,等待并稍后再试,找到另一个服务器,如此类推.至于客户端的死亡,我们可以暂时把它当作"别人的问题"来处理.

* Pub-sub:如果客户端死了(已经得到了一些数据),服务器不会知道这件事.Pub-sub不会从客户端向服务器发送任何信息.但是,客户端可以通过请求-回复等方式与服务器进行带外联系,并要求"请重新发送我错过的一切".至于服务器的死亡,这不在这里的范围之内.订阅者也可以自我验证他们是否运行得太慢,如果他们运行得太慢,可以采取行动(例如,警告操作员和死亡).

* 管道:如果一个工人死了(在工作时),通风设备不知道这件事.管道,就像时间的研磨齿轮,只在一个方向上工作.但下游的收集器可以检测到一个任务没有完成,并向通风器发送一个消息,说"嘿,重新发送任务324！" 如果通风器或收集器死了,无论哪个上游客户端最初发送的工作批次都会厌倦等待,并重新发送整个批次.这并不优雅,但系统代码确实不应该经常死掉,这一点很重要.

在这一章中,我们将只关注请求-回复,这是可靠消息传递的低挂果实.

基本的请求-回复模式(一个 REQ 客户端套接字向一个 REP 服务器套接字进行阻塞式发送/接收)在处理最常见的故障类型方面得分很低.如果服务器在处理请求时崩溃了,客户端就会永远挂起.如果网络丢失了请求或回复,客户端就会永远挂起.

请求-回复仍然比TCP好得多,这要归功于ZeroMQ默默地重新连接对等体的能力,以及负载平衡消息等.但对于实际工作来说,它仍然不够好.唯一能让你真正相信基本的请求-回复模式的情况是在同一进程中的两个线程之间,那里没有网络或独立的服务器进程会死亡.

然而,只要做一点额外的工作,这个不起眼的模式就能成为分布式网络中实际工作的良好基础,我们得到了一组可靠的请求-回复(RRR)模式,我喜欢称之为*海盗*模式(我希望你最终会明白这个笑话).

根据我的经验,大概有三种方式来连接客户和服务器.每种方式都需要一种特定的可靠性方法.

* <mark><font color=darkred>多个客户端直接与一个服务器对话.用例:客户需要与之对话的单一知名服务器.我们旨在处理的故障类型:服务器崩溃和重新启动,以及网络断开.</font></mark>

* <mark><font color=darkred>多个客户端与一个将工作分配给多个worker的broker交谈.用例:面向服务的事务处理.我们要处理的故障类型:工作器崩溃和重启,工作器繁忙循环,工作器过载,队列崩溃和重启,以及网络断开连接.</font></mark>

* <mark><font color=darkred>多个客户端与多个服务器对话,没有中间代理.用例:分布式服务,如名称解析.我们旨在处理的故障类型:服务崩溃和重启,服务繁忙循环,服务过载,以及网络断开连接.</font></mark>

这些方法中的每一种都有其权衡之处,而且往往你会将它们混合起来.我们将详细研究这三种方法.

## 客户端的可靠性(懒惰的海盗模式)

我们可以通过对客户端的一些改变来获得非常简单的可靠的请求-回复.我们称之为"懒惰海盗模式"[图].我们不是做一个阻塞的接收,而是.

* 轮询REQ套接字,只有在确定有回复到达时才从它那里接收.
* 如果在超时时间内没有回复,则重新发送请求.
* 如果多次请求后仍无回复,则放弃该事务.

如果你试图在严格的发送/接收方式之外使用REQ套接字,你会得到一个错误(从技术上讲,REQ套接字实现了一个小型的有限状态机来执行发送/接收乒乓,所以错误代码被称为"EFSM").当我们想在海盗模式中使用REQ时,这就略显烦人了,因为我们可能会在得到回复之前发送几个请求.

蛮好的解决方法是在出现错误后关闭并重新打开REQ套接字.

  <summary><mark><font color=darkred>Lazy Pirate client</font></mark></summary>

```c
#include <czmq.h>
#define REQUEST_TIMEOUT 2500 // msecs, (>1000!)
#define REQUEST_RETRIES 3 // Before we abandon
#define SERVER_ENDPOINT"tcp://localhost:5555"

int main()
{
    zsock_t *client = zsock_new_req(SERVER_ENDPOINT);
    printf("I: Connecting to server...\n");
    assert(client);

    int sequence = 0;
    int retries_left = REQUEST_RETRIES;
    printf("Entering while loop...\n");
    while(retries_left) // interrupt needs to be handled
    {
        // We send a request, then we get a reply
        char request[10];
        sprintf(request,"%d", ++sequence);
        zstr_send(client, request);
        int expect_reply = 1;
        while(expect_reply)
        {
            printf("Expecting reply....\n");
            zmq_pollitem_t items [] = {{zsock_resolve(client), 0, ZMQ_POLLIN, 0}};
            printf("After polling\n");
            int rc = zmq_poll(items, 1, REQUEST_TIMEOUT * ZMQ_POLL_MSEC);
            printf("Polling Done.. \n");
            if (rc == -1)
                break; // Interrupted
            
            // Here we process a server reply and exit our loop if the
            // reply is valid. If we didn't get a reply we close the
            // client socket, open it again and resend the request. We
            // try a number times before finally abandoning:

            if (items[0].revents & ZMQ_POLLIN)
            {
                // We got a reply from the server, must match sequence
                char *reply = zstr_recv(client);
                if(!reply)
                    break; // interrupted
                if (atoi(reply) == sequence)
                {
                    printf("I: server replied OK (%s)\n", reply);
                    retries_left=REQUEST_RETRIES;
                    expect_reply = 0;
                }
                else
                {
                    printf("E: malformed reply from server: %s\n", reply);
                }
                free(reply);
            }
            else 
            {
                if(--retries_left == 0)
                {
                    printf("E: Server seems to be offline, abandoning\n");
                    break;
                }
                else
                {
                    printf("W: no response from server, retrying...\n");
                    zsock_destroy(&client);
                    printf("I: reconnecting to server...\n");
                    client = zsock_new_req(SERVER_ENDPOINT);
                    zstr_send(client, request);
                }
            }
        }
        zsock_destroy(&client);
        return 0;
    }
}
```
</details>

与匹配的服务器一起运行这个程序.

  <summary><mark><font color=darkred>Lazy Pirate server</font></mark></summary>
  
```c
//  Lazy Pirate server
//  Binds REQ socket to tcp://*:5555
//  Like hwserver except:
//   - echoes request as-is
//   - randomly runs slowly, or exits to simulate a crash.

#include"zhelpers.h"
#include <unistd.h>

int main (void)
{
    srandom ((unsigned) time (NULL));

    void *context = zmq_ctx_new ();
    void *server = zmq_socket (context, ZMQ_REP);
    zmq_bind (server,"tcp://*:5555");

    int cycles = 0;
    while (1) {
        char *request = s_recv (server);
        cycles++;

        //  Simulate various problems, after a few cycles
        if (cycles > 3 && randof (3) == 0) {
            printf ("I: simulating a crash\n");
            break;
        }
        else
        if (cycles > 3 && randof (3) == 0) {
            printf ("I: simulating CPU overload\n");
            sleep (2);
        }
        printf ("I: normal request (%s)\n", request);
        sleep (1);              //  Do some heavy work
        s_send (server, request);
        free (request);
    }
    zmq_close (server);
    zmq_ctx_destroy (context);
    return 0;
}
```
</details>

```[[type="textdiagram" title="The Lazy Pirate Pattern"]]
#-----------#   #-----------#   #-----------#
|  Client   |   |  Client   |   |  Client   |
+-----------+   +-----------+   +-----------+
|   Retry   |   |   Retry   |   |   Retry   |
+-----------+   +-----------+   +-----------+
|    REQ    |   |    REQ    |   |    REQ    |
'-----------'   '-----------'   '-----------'
      ^               ^               ^
      |               |               |
      '---------------+---------------'
                      |
                      v
               .-------------.
               |     REP     |
               +-------------+
               |             |
               |    Server   |
               |             |
               #-------------#
```

要运行这个测试案例,在两个控制台窗口中启动客户端和服务器.服务器会在几条信息后随机出现错误行为.你可以检查客户端的响应.下面是服务器的典型输出.

```
I: normal request (1)
I: normal request (2)
I: normal request (3)
I: simulating CPU overload
I: normal request (4)
I: simulating a crash
```

下面是客户端的响应.


```
I: connecting to server...
I: server replied OK (1)
I: server replied OK (2)
I: server replied OK (3)
W: no response from server, retrying...
I: connecting to server...
W: no response from server, retrying...
I: connecting to server...
E: server seems to be offline, abandoning
```

客户端对每个消息进行排序,并检查回复是否完全按顺序返回:没有请求或回复丢失,也没有回复超过一次,或不按顺序返回.运行几次测试,直到你确信这个机制确实有效.在生产应用中你不需要序列号；它们只是帮助我们相信我们的设计.

客户端使用的是REQ套接字,并进行强行关闭/重新打开,因为REQ套接字规定了严格的发送/接收周期.你可能很想用DEALER来代替,但这不是一个好的决定.首先,这意味着模仿REQ对信封的秘密处理(如果你忘记了那是什么,那就说明你不想去做).第二,这意味着可能会收到你没有想到的回复.

当我们有一组客户端与一个服务器对话时,只在客户端处理故障是可行的.它可以处理服务器崩溃,但只有在恢复意味着重新启动同一台服务器的情况下.如果有一个永久性的错误,比如服务器硬件上的电源坏了,这种方法就行不通.因为在任何架构中,服务器中的应用程序代码通常是最大的故障源,依赖单一服务器并不是一个好主意.

所以,利与弊.

* 优点:简单易懂,易于实现.
* 优点:很容易与现有的客户和服务器应用代码一起工作.
* 优点:ZeroMQ会自动重试实际的重新连接,直到它成功.
* 缺点:没有故障转移到备份或备用服务器.

## 基本可靠队列(简单海盗模式)

我们的第二种方法是用一个队列broker来扩展懒惰海盗模式,让我们能够透明地与多个服务器对话,我们可以更准确地称之为"worker".我们将分阶段进行开发,从一个最小的工作模式开始,即简单海盗模式.

在所有这些海盗模式中,worker是无状态的.如果应用程序需要一些共享状态,比如共享数据库,我们在设计消息传递框架时并不了解这些状态.拥有一个队列broker意味着worker可以来来去去,而客户端对此一无所知.如果一个worker死亡,另一个就会接手.这是一个很好的,简单的拓扑结构,只有一个真正的弱点,即中央队列本身,它可能成为一个管理问题,也是一个单点故障.

```[[type="textdiagram" title="The Simple Pirate Pattern"]]
#-----------#   #-----------#   #-----------#
|  Client   |   |  Client   |   |  Client   |
+-----------+   +-----------+   +-----------+
|   Retry   |   |   Retry   |   |   Retry   |
+-----------+   +-----------+   +-----------+
|    REQ    |   |    REQ    |   |    REQ    |
'-----+-----'   '-----+-----'   '-----+-----'
      |               |               |
      '---------------+---------------'
                      |
                      v
                .-----------.
                |  ROUTER   |
                +-----------+
                |   Load    |
                |  balancer |
                +-----------+
                |  ROUTER   |
                '-----------'
                      ^
                      |
      .---------------+---------------.
      |               |               |
.-----+-----.   .-----+-----.   .-----+-----.
|    REQ    |   |    REQ    |   |    REQ    |
+-----------+   +-----------+   +-----------+
|   Worker  |   |   Worker  |   |   Worker  |
#-----------#   #-----------#   #-----------#
```

队列broker的基础是来自[#高级-请求-回复]的负载平衡代理.我们在处理死亡或阻塞的worker时,最*少需要做什么？事实证明,这是很难得的.我们在客户端已经有一个重试机制.因此,使用负载平衡模式将非常有效.这符合ZeroMQ的理念,即我们可以通过在中间插入天真的代理来扩展点对点模式,比如请求-回复[图].

我们不需要一个特殊的客户端；我们仍然在使用Lazy Pirate客户端.这里是队列,它与负载平衡代理的主要任务相同.


  <summary><mark><font color=darkred>Simple Pirate broker</font></mark></summary>

```c
//  Simple Pirate broker
//  This is identical to load-balancing pattern, with no reliability
//  mechanisms. It depends on the client for recovery. Runs forever.

#include"czmq.h"
#define WORKER_READY  "\001"      //  Signals worker is ready

int main (void)
{
    zctx_t *ctx = zctx_new ();
    void *frontend = zsocket_new (ctx, ZMQ_ROUTER);
    void *backend = zsocket_new (ctx, ZMQ_ROUTER);
    zsocket_bind (frontend,"tcp://*:5555");    //  For clients
    zsocket_bind (backend, "tcp://*:5556");    //  For workers

    //  Queue of available workers
    zlist_t *workers = zlist_new ();
    
    //  The body of this example is exactly the same as lbbroker2.
    //  .skip
    while (true) {
        zmq_pollitem_t items [] = {
            { backend,  0, ZMQ_POLLIN, 0 },
            { frontend, 0, ZMQ_POLLIN, 0 }
        };
        //  Poll frontend only if we have available workers
        int rc = zmq_poll (items, zlist_size (workers)? 2: 1, -1);
        if (rc == -1)
            break;              //  Interrupted

        //  Handle worker activity on backend
        if (items [0].revents & ZMQ_POLLIN) {
            //  Use worker identity for load-balancing
            zmsg_t *msg = zmsg_recv (backend);
            if (!msg)
                break;          //  Interrupted
            zframe_t *identity = zmsg_unwrap (msg);
            zlist_append (workers, identity);

            //  Forward message to client if it's not a READY
            zframe_t *frame = zmsg_first (msg);
            if (memcmp (zframe_data (frame), WORKER_READY, 1) == 0)
                zmsg_destroy (&msg);
            else
                zmsg_send (&msg, frontend);
        }
        if (items [1].revents & ZMQ_POLLIN) {
            //  Get client request, route to first available worker
            zmsg_t *msg = zmsg_recv (frontend);
            if (msg) {
                zmsg_wrap (msg, (zframe_t *) zlist_pop (workers));
                zmsg_send (&msg, backend);
            }
        }
    }
    //  When we're done, clean up properly
    while (zlist_size (workers)) {
        zframe_t *frame = (zframe_t *) zlist_pop (workers);
        zframe_destroy (&frame);
    }
    zlist_destroy (&workers);
    zctx_destroy (&ctx);
    return 0;
    //  .until
}
```
</details>

这里是工作器,它接收懒惰海盗服务器并使其适应负载均衡模式(使用REQ"就绪"信号).


  <summary><mark><font color=darkred>Simple Pirate worker</font></mark></summary>

```c
//  Simple Pirate worker
//  Connects REQ socket to tcp://*:5556
//  Implements worker part of load-balancing

#include"czmq.h"
#define WORKER_READY  "\001"      //  Signals worker is ready

int main (void)
{
    zctx_t *ctx = zctx_new ();
    void *worker = zsocket_new (ctx, ZMQ_REQ);

    //  Set random identity to make tracing easier
    srandom ((unsigned) time (NULL));
    char identity [10];
    sprintf (identity,"%04X-%04X", randof (0x10000), randof (0x10000));
    zmq_setsockopt (worker, ZMQ_IDENTITY, identity, strlen (identity));
    zsocket_connect (worker,"tcp://localhost:5556");

    //  Tell broker we're ready for work
    printf ("I: (%s) worker ready\n", identity);
    zframe_t *frame = zframe_new (WORKER_READY, 1);
    zframe_send (&frame, worker, 0);

    int cycles = 0;
    while (true) {
        zmsg_t *msg = zmsg_recv (worker);
        if (!msg)
            break;              //  Interrupted

        //  Simulate various problems, after a few cycles
        cycles++;
        if (cycles > 3 && randof (5) == 0) {
            printf ("I: (%s) simulating a crash\n", identity);
            zmsg_destroy (&msg);
            break;
        }
        else
        if (cycles > 3 && randof (5) == 0) {
            printf ("I: (%s) simulating CPU overload\n", identity);
            sleep (3);
            if (zctx_interrupted)
                break;
        }
        printf ("I: (%s) normal reply\n", identity);
        sleep (1);              //  Do some heavy work
        zmsg_send (&msg, worker);
    }
    zctx_destroy (&ctx);
    return 0;
}
```
</details>

要测试这一点,可以按任何顺序启动一些工作器,一个懒惰海盗客户端和队列.你会看到,工人最终都崩溃了,客户端重试,然后放弃了.队列永远不会停止,你可以无休止地重新启动工人和客户端.这种模式适用于任何数量的客户和worker.

## 稳健可靠的排队(偏执狂海盗模式)

```[[type="textdiagram" title="The Paranoid Pirate Pattern"]]
#-----------#   #-----------#   #-----------#
|  Client   |   |  Client   |   |  Client   |
+-----------+   +-----------+   +-----------+
|   Retry   |   |   Retry   |   |   Retry   |
+-----------+   +-----------+   +-----------+
|    REQ    |   |    REQ    |   |    REQ    |
'-----+-----'   '-----+-----'   '-----+-----'
      |               |               |
      '---------------+---------------'
                      |
                      v
                .-----------.
                |  ROUTER   |
                +-----------+
                |   Queue   |
                +-----------+
                | Heartbeat |
                +-----------+
                |  ROUTER   |
                '-----------'
                      ^
                      |
      .---------------+---------------.
      |               |               |
.-----+-----.   .-----+-----.   .-----+-----.
|  DEALER   |   |  DEALER   |   |  DEALER   |
+-----------+   +-----------+   +-----------+
| Heartbeat |   | Heartbeat |   | Heartbeat |
+-----------+   +-----------+   +-----------+
|   Worker  |   |   Worker  |   |   Worker  |
#-----------#   #-----------#   #-----------#
```

简单的海盗队列模式效果相当好,特别是因为它只是两个现有模式的组合.不过,它确实有一些弱点.

* 它在面对队列崩溃和重启时并不健壮.客户端会恢复,但worker不会.虽然ZeroMQ会自动重新连接worker的套接字,但就新启动的队列而言,worker还没有发出准备好的信号,所以并不存在.为了解决这个问题,我们必须进行从队列到worker的心跳,这样worker就可以检测到队列何时消失了.

* 队列不会检测到worker的失败,所以如果一个worker在空闲时死亡,队列无法将其从worker队列中移除,直到队列向它发出请求.客户端的等待和重试都是徒劳的.这不是一个严重的问题,但也不是很好.为了使其正常工作,我们从worker到队列进行心跳,以便队列可以在任何阶段检测到丢失的worker.

我们将在一个适当的迂腐的偏执狂海盗模式中解决这些问题.

我们之前为worker使用了一个REQ套接字.对于偏执狂海盗的工作器,我们将改用DEALER套接字[图].这样做的好处是,我们可以在任何时候发送和接收消息,而不是像REQ那样锁死的发送/接收.DEALER的缺点是我们必须做我们自己的信封管理(重新阅读[#高级-请求-回复]了解这个概念的背景).

我们仍然在使用Lazy Pirate的客户端.这里是偏执狂海盗的队列broker.


  <summary><mark><font color=darkred>Paranoid Pirate worker</font></mark></summary>

```c
//  Paranoid Pirate queue

#include"czmq.h"
#define HEARTBEAT_LIVENESS  3       //  3-5 is reasonable
#define HEARTBEAT_INTERVAL  1000    //  msecs

//  Paranoid Pirate Protocol constants
#define PPP_READY      "\001"      //  Signals worker is ready
#define PPP_HEARTBEAT  "\002"      //  Signals worker heartbeat

//  .split worker class structure
//  Here we define the worker class; a structure and a set of functions that
//  act as constructor, destructor, and methods on worker objects:

typedef struct {
    zframe_t *identity;         //  Identity of worker
    char *id_string;            //  Printable identity
    int64_t expiry;             //  Expires at this time
} worker_t;

//  Construct new worker
static worker_t *
s_worker_new (zframe_t *identity)
{
    worker_t *self = (worker_t *) zmalloc (sizeof (worker_t));
    self->identity = identity;
    self->id_string = zframe_strhex (identity);
    self->expiry = zclock_time ()
                 + HEARTBEAT_INTERVAL * HEARTBEAT_LIVENESS;
    return self;
}

//  Destroy specified worker object, including identity frame.
static void
s_worker_destroy (worker_t **self_p)
{
    assert (self_p);
    if (*self_p) {
        worker_t *self = *self_p;
        zframe_destroy (&self->identity);
        free (self->id_string);
        free (self);
        *self_p = NULL;
    }
}

//  .split worker ready method
//  The ready method puts a worker to the end of the ready list:

static void
s_worker_ready (worker_t *self, zlist_t *workers)
{
    worker_t *worker = (worker_t *) zlist_first (workers);
    while (worker) {
        if (streq (self->id_string, worker->id_string)) {
            zlist_remove (workers, worker);
            s_worker_destroy (&worker);
            break;
        }
        worker = (worker_t *) zlist_next (workers);
    }
    zlist_append (workers, self);
}

//  .split get next available worker
//  The next method returns the next available worker identity:

static zframe_t *
s_workers_next (zlist_t *workers)
{
    worker_t *worker = zlist_pop (workers);
    assert (worker);
    zframe_t *frame = worker->identity;
    worker->identity = NULL;
    s_worker_destroy (&worker);
    return frame;
}

//  .split purge expired workers
//  The purge method looks for and kills expired workers. We hold workers
//  from oldest to most recent, so we stop at the first alive worker:

static void
s_workers_purge (zlist_t *workers)
{
    worker_t *worker = (worker_t *) zlist_first (workers);
    while (worker) {
        if (zclock_time () < worker->expiry)
            break;              //  Worker is alive, we're done here

        zlist_remove (workers, worker);
        s_worker_destroy (&worker);
        worker = (worker_t *) zlist_first (workers);
    }
}

//  .split main task
//  The main task is a load-balancer with heartbeating on workers so we
//  can detect crashed or blocked worker tasks:

int main (void)
{
    zctx_t *ctx = zctx_new ();
    void *frontend = zsocket_new (ctx, ZMQ_ROUTER);
    void *backend = zsocket_new (ctx, ZMQ_ROUTER);
    zsocket_bind (frontend,"tcp://*:5555");    //  For clients
    zsocket_bind (backend, "tcp://*:5556");    //  For workers

    //  List of available workers
    zlist_t *workers = zlist_new ();

    //  Send out heartbeats at regular intervals
    uint64_t heartbeat_at = zclock_time () + HEARTBEAT_INTERVAL;

    while (true) {
        zmq_pollitem_t items [] = {
            { backend,  0, ZMQ_POLLIN, 0 },
            { frontend, 0, ZMQ_POLLIN, 0 }
        };
        //  Poll frontend only if we have available workers
        int rc = zmq_poll (items, zlist_size (workers)? 2: 1,
            HEARTBEAT_INTERVAL * ZMQ_POLL_MSEC);
        if (rc == -1)
            break;              //  Interrupted

        //  Handle worker activity on backend
        if (items [0].revents & ZMQ_POLLIN) {
            //  Use worker identity for load-balancing
            zmsg_t *msg = zmsg_recv (backend);
            if (!msg)
                break;          //  Interrupted

            //  Any sign of life from worker means it's ready
            zframe_t *identity = zmsg_unwrap (msg);
            worker_t *worker = s_worker_new (identity);
            s_worker_ready (worker, workers);

            //  Validate control message, or return reply to client
            if (zmsg_size (msg) == 1) {
                zframe_t *frame = zmsg_first (msg);
                if (memcmp (zframe_data (frame), PPP_READY, 1)
                &&  memcmp (zframe_data (frame), PPP_HEARTBEAT, 1)) {
                    printf ("E: invalid message from worker");
                    zmsg_dump (msg);
                }
                zmsg_destroy (&msg);
            }
            else
                zmsg_send (&msg, frontend);
        }
        if (items [1].revents & ZMQ_POLLIN) {
            //  Now get next client request, route to next worker
            zmsg_t *msg = zmsg_recv (frontend);
            if (!msg)
                break;          //  Interrupted
            zframe_t *identity = s_workers_next (workers); 
            zmsg_prepend (msg, &identity);
            zmsg_send (&msg, backend);
        }
        //  .split handle heartbeating
        //  We handle heartbeating after any socket activity. First, we send
        //  heartbeats to any idle workers if it's time. Then, we purge any
        //  dead workers:
        if (zclock_time () >= heartbeat_at) {
            worker_t *worker = (worker_t *) zlist_first (workers);
            while (worker) {
                zframe_send (&worker->identity, backend,
                             ZFRAME_REUSE + ZFRAME_MORE);
                zframe_t *frame = zframe_new (PPP_HEARTBEAT, 1);
                zframe_send (&frame, backend, 0);
                worker = (worker_t *) zlist_next (workers);
            }
            heartbeat_at = zclock_time () + HEARTBEAT_INTERVAL;
        }
        s_workers_purge (workers);
    }
    //  When we're done, clean up properly
    while (zlist_size (workers)) {
        worker_t *worker = (worker_t *) zlist_pop (workers);
        s_worker_destroy (&worker);
    }
    zlist_destroy (&workers);
    zctx_destroy (&ctx);
    return 0;
}
```
</details>

这里是工作器,它接收懒惰海盗服务器并使其适应负载均衡模式(使用REQ"就绪"信号).

该队列用worker的心跳来扩展负载平衡模式.跳心是那些"简单"的事情之一,但却很难做到正确.我稍后会解释更多关于这个问题.

下面是偏执狂海盗的工作器.


  <summary><mark><font color=darkred>Paranoid Pirate worker</font></mark></summary>

```c
//  Paranoid Pirate worker

#include"czmq.h"
#define HEARTBEAT_LIVENESS  3       //  3-5 is reasonable
#define HEARTBEAT_INTERVAL  1000    //  msecs
#define INTERVAL_INIT       1000    //  Initial reconnect
#define INTERVAL_MAX       32000    //  After exponential backoff

//  Paranoid Pirate Protocol constants
#define PPP_READY      "\001"      //  Signals worker is ready
#define PPP_HEARTBEAT  "\002"      //  Signals worker heartbeat

//  Helper function that returns a new configured socket
//  connected to the Paranoid Pirate queue

static void *
s_worker_socket (zctx_t *ctx) {
    void *worker = zsocket_new (ctx, ZMQ_DEALER);
    zsocket_connect (worker,"tcp://localhost:5556");

    //  Tell queue we're ready for work
    printf ("I: worker ready\n");
    zframe_t *frame = zframe_new (PPP_READY, 1);
    zframe_send (&frame, worker, 0);

    return worker;
}

//  .split main task
//  We have a single task that implements the worker side of the
//  Paranoid Pirate Protocol (PPP). The interesting parts here are
//  the heartbeating, which lets the worker detect if the queue has
//  died, and vice versa:

int main (void)
{
    zctx_t *ctx = zctx_new ();
    void *worker = s_worker_socket (ctx);

    //  If liveness hits zero, queue is considered disconnected
    size_t liveness = HEARTBEAT_LIVENESS;
    size_t interval = INTERVAL_INIT;

    //  Send out heartbeats at regular intervals
    uint64_t heartbeat_at = zclock_time () + HEARTBEAT_INTERVAL;

    srandom ((unsigned) time (NULL));
    int cycles = 0;
    while (true) {
        zmq_pollitem_t items [] = { { worker,  0, ZMQ_POLLIN, 0 } };
        int rc = zmq_poll (items, 1, HEARTBEAT_INTERVAL * ZMQ_POLL_MSEC);
        if (rc == -1)
            break;              //  Interrupted

        if (items [0].revents & ZMQ_POLLIN) {
            //  Get message
            //  - 3-part envelope + content -> request
            //  - 1-part HEARTBEAT -> heartbeat
            zmsg_t *msg = zmsg_recv (worker);
            if (!msg)
                break;          //  Interrupted

            //  .split simulating problems
            //  To test the robustness of the queue implementation we 
            //  simulate various typical problems, such as the worker
            //  crashing or running very slowly. We do this after a few
            //  cycles so that the architecture can get up and running
            //  first:
            if (zmsg_size (msg) == 3) {
                cycles++;
                if (cycles > 3 && randof (5) == 0) {
                    printf ("I: simulating a crash\n");
                    zmsg_destroy (&msg);
                    break;
                }
                else
                if (cycles > 3 && randof (5) == 0) {
                    printf ("I: simulating CPU overload\n");
                    sleep (3);
                    if (zctx_interrupted)
                        break;
                }
                printf ("I: normal reply\n");
                zmsg_send (&msg, worker);
                liveness = HEARTBEAT_LIVENESS;
                sleep (1);              //  Do some heavy work
                if (zctx_interrupted)
                    break;
            }
            else
            //  .split handle heartbeats
            //  When we get a heartbeat message from the queue, it means the
            //  queue was (recently) alive, so we must reset our liveness
            //  indicator:
            if (zmsg_size (msg) == 1) {
                zframe_t *frame = zmsg_first (msg);
                if (memcmp (zframe_data (frame), PPP_HEARTBEAT, 1) == 0)
                    liveness = HEARTBEAT_LIVENESS;
                else {
                    printf ("E: invalid message\n");
                    zmsg_dump (msg);
                }
                zmsg_destroy (&msg);
            }
            else {
                printf ("E: invalid message\n");
                zmsg_dump (msg);
            }
            interval = INTERVAL_INIT;
        }
        else
        //  .split detecting a dead queue
        //  If the queue hasn't sent us heartbeats in a while, destroy the
        //  socket and reconnect. This is the simplest most brutal way of
        //  discarding any messages we might have sent in the meantime:
        if (--liveness == 0) {
            printf ("W: heartbeat failure, can't reach queue\n");
            printf ("W: reconnecting in %zd msec...\n", interval);
            zclock_sleep (interval);

            if (interval < INTERVAL_MAX)
                interval *= 2;
            zsocket_destroy (ctx, worker);
            worker = s_worker_socket (ctx);
            liveness = HEARTBEAT_LIVENESS;
        }
        //  Send heartbeat to queue if it's time
        if (zclock_time () > heartbeat_at) {
            heartbeat_at = zclock_time () + HEARTBEAT_INTERVAL;
            printf ("I: worker heartbeat\n");
            zframe_t *frame = zframe_new (PPP_HEARTBEAT, 1);
            zframe_send (&frame, worker, 0);
        }
    }
    zctx_destroy (&ctx);
    return 0;
}
```
</details>

关于这个例子的一些评论.

* 该代码包括模拟失败,和以前一样.这使得它(a)非常难以调试,(b)重复使用很危险.当你想调试的时候,请禁用故障模拟.

* worker使用的重新连接策略与我们为懒惰海盗客户端设计的策略类似,但有两个主要区别.(a)它做了一个指数级的回退,(b)它无限期地重试(而客户端在报告失败之前会重试几次).

试试客户端,队列和worker,比如用这样的脚本.

```
ppqueue &
for i in 1 2 3 4; do
    ppworker &
    sleep 1
done
lpclient &
```

你应该看到worker在模拟崩溃时一个接一个地死去,而客户端最终也放弃了.你可以停止并重新启动队列,客户端和worker都会重新连接并继续工作.而且,无论你对队列和worker做了什么,客户端都不会得到一个失序的回复:整个链条要么工作,要么客户端放弃.

## Heartbeating

Heartbeating解决了知道一个对等体是活着还是死了的问题.这不是ZeroMQ特有的问题.TCP有一个很长的超时(30分钟左右),这意味着可能无法知道一个对等体是否已经死亡,被断开连接,或者在周末带着一箱伏特加,一个红发女郎和一个大额支出账户去了布拉格.

要把心跳搞好并不容易.在写偏执狂海盗的例子时,花了大约五个小时才使心跳正常.其余的请求-回复链可能需要10分钟.创造"假失败"特别容易,即当对等体决定他们被断开连接时,因为心跳没有被正确发送.

我们来看看人们用ZeroMQ进行心跳时的三个主要答案.

### 甩掉它

最常见的方法是根本不做心跳,并希望得到最好的结果.许多甚至是大多数ZeroMQ应用程序都是这样做的.ZeroMQ通过在许多情况下隐藏对等体来鼓励这种做法.这种方法会导致什么问题？

* 当我们在跟踪对等体的应用程序中使用ROUTER套接字时,当对等体断开连接和重新连接时,应用程序将泄漏内存(应用程序为每个对等体持有的资源),并且变得越来越慢.

* 当我们使用基于SUB-或DEALER的数据接收者时,我们无法区分好的沉默(没有数据)和坏的沉默(另一端死亡).当接收者知道对方死了,它可以例如切换到一个备份路由.

* 如果我们使用的TCP连接长时间保持沉默,那么在某些网络中,它就会直接死亡.发送一些信息(从技术上讲,"keep-alive"多于心跳),将使网络保持活力.

### 单向心跳

第二个选择是每隔一秒左右从每个节点向其对等体发送一个心跳信息.当一个节点在某个超时时间内(通常是几秒钟)没有听到另一个节点的消息时,它将把这个对等体视为死亡.听起来不错,对吗？不幸的是,不是.这在某些情况下是可行的,但在其他情况下有令人讨厌的边缘情况.

对于pub-sub,这确实有效,而且是你可以使用的唯一模式.SUB套接字不能回话给PUB套接字,但是PUB套接字可以很高兴的向他们的订阅者发送"我还活着"的消息.

作为一种优化,你可以只在没有真正的数据需要发送时才发送心跳.此外,如果网络活动是一个问题(例如,在移动网络上,活动会耗尽电池),你可以渐渐地发送心跳,速度越来越慢.只要接收者能检测到故障(活动的急剧停止),那就没问题.

下面是这种设计的典型问题.

* 当我们发送大量数据时,它可能不准确,因为心跳会在这些数据后面延迟.如果心跳被延迟了,你可能会得到错误的超时和由于网络拥堵而导致的断线.因此,总是把*任何*传入的数据当作心跳,无论发送方是否优化出心跳.

* 虽然pub-sub模式会放弃消失的接收者的消息,但PUSH和DEALER套接字会排队.因此,如果你向一个已经死亡的对等体发送心跳,而它又回来了,它将得到你发送的所有心跳,这可能是成千上万的.哇哦,哇哦!

* 这个设计假设心跳超时在整个网络中是一样的.但这并不准确.有些对等体需要非常积极的心跳,以便迅速发现故障.而有些则希望心跳非常轻松,以便让沉睡的网络躺下,节省电力.

### 乒乓式心跳

第三种选择是使用乒乓对话.一个对等体向另一个对等体发送一个ping命令,另一个对等体用一个pong命令进行回复.两个命令都没有任何有效载荷.ping和pong是不相关的.因为在一些网络中,"客户"和"服务器"的角色是任意的,我们通常规定任何一个对等体实际上都可以发送一个ping,并期望得到一个pong的响应.然而,由于超时取决于动态客户端最熟悉的网络拓扑结构,通常是客户端ping服务器.

这适用于所有基于ROUTER的经纪商.我们在第二个模型中使用的同样的优化使这个工作变得更好:将任何传入的数据视为pong,并且只在不发送数据时发送ping.

### 偏执狂海盗的心跳

对于偏执狂海盗,我们选择了第二种方法.这可能不是最简单的选择:如果今天设计这个,我可能会尝试用ping-pong的方法来代替.然而,其原理是相似的.心跳信息在两个方向上都是异步流动的,任何一个对等体都可以决定另一个是"死"的,并停止与它对话.

在Worker中,我们是这样处理来自队列的心跳的.

* 我们计算一个*有效期*,也就是在决定队列死亡之前,我们还能错过多少次心跳.它从3开始,每错过一次心跳我们就递减一次.
* 在{{zmq_poll}}循环中,我们每次等待一秒钟,这就是我们的心跳间隔.
* 如果在这段时间内有任何来自队列的消息,我们就将我们的有效性重置为3.
* 如果在这段时间内没有消息,我们就倒数我们的有效期.
* 如果有效性达到零,我们认为队列已经死亡.
* 如果队列死了,我们销毁我们的套接字,创建一个新的套接字,然后重新连接.
* 为了避免打开和关闭过多的套接字,我们在重新连接之前要等待一定的时间间隔,而且每次都要把时间间隔延长一倍,直到达到32秒.

而这就是我们处理心跳的方式,*到*队列.

* 我们计算何时发送下一次心跳；这是一个单一的变量,因为我们是与一个对等体,即队列对话.
* 在{{zmq_poll}}循环中,只要我们通过这个时间,我们就向队列发送心跳.

下面是worker的基本心跳代码.

  <summary><mark><font color=darkred>查看</font></mark></summary>

```c++
#define HEARTBEAT_LIVENESS  3       *  3-5 is reasonable
#define HEARTBEAT_INTERVAL  1000    *  msecs
#define INTERVAL_INIT       1000    *  Initial reconnect
#define INTERVAL_MAX       32000    *  After exponential backoff

...
*  If liveness hits zero, queue is considered disconnected
size_t liveness = HEARTBEAT_LIVENESS;
size_t interval = INTERVAL_INIT;

*  Send out heartbeats at regular intervals
uint64_t heartbeat_at = zclock_time () + HEARTBEAT_INTERVAL;

while (true) {
    zmq_pollitem_t items [] = { { worker,  0, ZMQ_POLLIN, 0 } };
    int rc = zmq_poll (items, 1, HEARTBEAT_INTERVAL * ZMQ_POLL_MSEC);

    if (items [0].revents & ZMQ_POLLIN) {
        *  Receive any message from queue
        liveness = HEARTBEAT_LIVENESS;
        interval = INTERVAL_INIT;
    }
    else
    if (--liveness == 0) {
        zclock_sleep (interval);
        if (interval < INTERVAL_MAX)
            interval *= 2;
        zsocket_destroy (ctx, worker);
        ...
        liveness = HEARTBEAT_LIVENESS;
    }
    *  Send heartbeat to queue if it's time
    if (zclock_time () > heartbeat_at) {
        heartbeat_at = zclock_time () + HEARTBEAT_INTERVAL;
        *  Send heartbeat message to queue
    }
}
```
</details>

队列也是如此,但为每个worker管理了一个过期时间.

这里有一些关于你自己心跳的实施的提示.

* 使用{zmq_poll}}或反应器作为应用程序的主要任务的核心.

* 从建立对等体之间的心跳开始,通过模拟失败来测试它,然后*建立其余的消息流.之后再添加心跳则要麻烦得多.

* 使用简单的跟踪,即打印到控制台,以使其正常工作.为了帮助你追踪对等体之间的消息流,使用一个转储方法,如zmsg提供的,并对你的消息进行增量编号,这样你就可以看到是否有空隙.

* 在实际应用中,心跳必须是可配置的,通常是与对等体协商的.有些对等体需要积极的心跳,低至10msecs.另一些对等体则距离较远,希望心跳时间高达 30 秒.

* 如果你对不同的对等体有不同的心跳间隔,你的投票超时应该是其中最低的(最短的时间).不要使用无限的超时.

* 在你用来发送消息的同一个套接字上进行心跳,这样你的心跳也可以作为一个*keep-alive*来阻止网络连接变质(有些防火墙会对无声连接不友好).

## 契约和协议

如果你注意到了,你会意识到偏执狂海盗与简单海盗是不能互通的,因为有心跳.但我们如何定义"互操作性"呢？为了保证互操作性,我们需要一种契约,一种让不同的团队在不同的时间和地点编写的代码能保证一起工作的协议.我们把这称为"协议".

在没有规范的情况下进行实验是很有趣的,但对于真正的应用来说,这不是一个合理的基础.如果我们想用另一种语言写一个worker,会发生什么？我们必须阅读代码来了解事情是如何进行的吗？如果我们因为某些原因想改变协议怎么办？即使是一个简单的协议,如果它是成功的,也会不断发展,变得更加复杂.

缺乏合同是一个一次性应用的肯定标志.因此,让我们为这个协议写一个合同.我们怎么做呢？

在[http://rfc.zeromq.org rfc.zeromq.org]有一个wiki,我们特别将其作为公共ZeroMQ合约的家园.
要创建一个新的规范,如果需要的话,在wiki上注册,然后按照说明操作.这是相当简单的,尽管写技术文本不是每个人都喜欢的.

我花了大约十五分钟来起草新的[http://rfc.zeromq.org/spec:6 Pirate Pattern Protocol].它不是一个大的规范,但它确实捕捉到了足够的信息,可以作为争论的基础("你的队列不兼容PPP；请修复它！").

把PPP变成一个真正的协议将需要更多的工作.

* 在READY命令中应该有一个协议版本号,这样就有可能区分不同版本的PPP.

* 现在,READY和HEARTBEAT并没有完全区别于请求和回复.为了使它们与众不同,我们需要一个包括"消息类型"部分的消息结构.

## 面向服务的可靠排队(Majordomo模式)

```[[type="textdiagram" title="The Majordomo Pattern"]]
#-----------#   #-----------#   #-----------#
|  Client   |   |  Client   |   |  Client   |
'-----+-----'   '-----+-----'   '-----+-----'
      |               |               |
      '---------------+---------------'
"Give me coffee"      |        "Give me tea"
                      v
                .-----------.
                |  Broker   |
                '-----------'
                      ^
                      |
      .---------------+---------------.
      |               |               |
.-----+-----.   .-----+-----.   .-----+-----.
| "Water"  |   |  "Tea"   |   |"Coffee"  |
+-----------+   +-----------+   +-----------+
|  Worker   |   |  Worker   |   |  Worker   |
#-----------#   #-----------#   #-----------#
```


进步的好处是,当律师和委员会不参与时,它发生得非常快.[http://rfc.zeromq.org/spec:7 一页纸的MDP规范]将PPP变成了更坚实的东西[图].这就是我们应该如何设计复杂的架构:从写下合同开始,然后才**写软件来实现它们.

Majordomo协议(MDP)以一种有趣的方式对PPP进行了扩展和改进:它在客户端发送的请求中增加了一个"服务名称",并要求worker注册特定的服务.添加服务名称将我们的偏执狂海盗队列变成了一个面向服务的borkers.MDP的好处是,它来自于工作代码,一个更简单的祖先协议(PPP),以及一套精确的改进,每个都解决了一个明确的问题.这使得它很容易起草.

为了实现Majordomo,我们需要为客户端和worker编写一个框架.要求每个应用开发者阅读规范并使其工作,这实在是不理智的,因为他们可以使用一个更简单的API来为他们做工作.

因此,虽然我们的第一个合同(MDP本身)定义了我们的分布式架构的各个部分是如何相互交谈的,但我们的第二个合同定义了用户应用程序如何与我们将要设计的技术框架交谈.

Majordomo有两个部分,一个是客户端,一个是worker端.因为我们将同时编写客户端和工作端应用程序,所以我们将需要两个API.下面是一个客户端API的草图,使用一个简单的面向对象的方法.

```[[type="fragment" name="mdclient"]]
mdcli_t *mdcli_new     (char *broker);
void     mdcli_destroy (mdcli_t **self_p);
zmsg_t  *mdcli_send    (mdcli_t *self, char *service, zmsg_t **request_p);
```

就是这样.我们向borkers打开一个会话,发送一个请求信息,得到一个回复信息,并最终关闭连接.这里有一个Worker API的草图.

```[[type="fragment" name="mdworker"]]
mdwrk_t *mdwrk_new (char *broker,char *service).
void mdwrk_destroy (mdwrk_t **self_p).
zmsg_t *mdwrk_recv (mdwrk_t *self, zmsg_t *reply);
```

这或多或少是对称的,但worker的对话有点不同.worker第一次做recv()时,会传递一个空回复.此后,它传递当前的回复,并获得一个新的请求.

客户端和workerAPI的构建相当简单,因为它们在很大程度上是基于我们已经开发的Paranoid Pirate代码.下面是客户端的API.

```c
//  mdcliapi class - Majordomo Protocol Client API
//  Implements the MDP/Worker spec at http://rfc.zeromq.org/spec:7.

#include"mdcliapi.h"

//  Structure of our class
//  We access these properties only via class methods

struct _mdcli_t {
    zctx_t *ctx;                //  Our context
    char *broker;
    void *client;               //  Socket to broker
    int verbose;                //  Print activity to stdout
    int timeout;                //  Request timeout
    int retries;                //  Request retries
};

//  Connect or reconnect to broker

void s_mdcli_connect_to_broker (mdcli_t *self)
{
    if (self->client)
        zsocket_destroy (self->ctx, self->client);
    self->client = zsocket_new (self->ctx, ZMQ_REQ);
    zmq_connect (self->client, self->broker);
    if (self->verbose)
        zclock_log ("I: connecting to broker at %s...", self->broker);
}

//  .split constructor and destructor
//  Here we have the constructor and destructor for our class:

//  Constructor

mdcli_t *
mdcli_new (char *broker, int verbose)
{
    assert (broker);

    mdcli_t *self = (mdcli_t *) zmalloc (sizeof (mdcli_t));
    self->ctx = zctx_new ();
    self->broker = strdup (broker);
    self->verbose = verbose;
    self->timeout = 2500;           //  msecs
    self->retries = 3;              //  Before we abandon

    s_mdcli_connect_to_broker (self);
    return self;
}

//  Destructor

void
mdcli_destroy (mdcli_t **self_p)
{
    assert (self_p);
    if (*self_p) {
        mdcli_t *self = *self_p;
        zctx_destroy (&self->ctx);
        free (self->broker);
        free (self);
        *self_p = NULL;
    }
}

//  .split configure retry behavior
//  These are the class methods. We can set the request timeout and number
//  of retry attempts before sending requests:

//  Set request timeout

void
mdcli_set_timeout (mdcli_t *self, int timeout)
{
    assert (self);
    self->timeout = timeout;
}

//  Set request retries

void
mdcli_set_retries (mdcli_t *self, int retries)
{
    assert (self);
    self->retries = retries;
}

//  .split send request and wait for reply
//  Here is the {{send}} method. It sends a request to the broker and gets
//  a reply even if it has to retry several times. It takes ownership of 
//  the request message, and destroys it when sent. It returns the reply
//  message, or NULL if there was no reply after multiple attempts:

zmsg_t *
mdcli_send (mdcli_t *self, char *service, zmsg_t **request_p)
{
    assert (self);
    assert (request_p);
    zmsg_t *request = *request_p;

    //  Prefix request with protocol frames
    //  Frame 1:"MDPCxy" (six bytes, MDP/Client x.y)
    //  Frame 2: Service name (printable string)
    zmsg_pushstr (request, service);
    zmsg_pushstr (request, MDPC_CLIENT);
    if (self->verbose) {
        zclock_log ("I: send request to '%s' service:", service);
        zmsg_dump (request);
    }
    int retries_left = self->retries;
    while (retries_left && !zctx_interrupted) {
        zmsg_t *msg = zmsg_dup (request);
        zmsg_send (&msg, self->client);

        zmq_pollitem_t items [] = {
            { self->client, 0, ZMQ_POLLIN, 0 }
        };
        //  .split body of send 
        //  On any blocking call, {{libzmq}} will return -1 if there was
        //  an error; we could in theory check for different error codes,
        //  but in practice it's OK to assume it was {{EINTR}} (Ctrl-C):
        
        int rc = zmq_poll (items, 1, self->timeout * ZMQ_POLL_MSEC);
        if (rc == -1)
            break;          //  Interrupted

        //  If we got a reply, process it
        if (items [0].revents & ZMQ_POLLIN) {
            zmsg_t *msg = zmsg_recv (self->client);
            if (self->verbose) {
                zclock_log ("I: received reply:");
                zmsg_dump (msg);
            }
            //  We would handle malformed replies better in real code
            assert (zmsg_size (msg) >= 3);

            zframe_t *header = zmsg_pop (msg);
            assert (zframe_streq (header, MDPC_CLIENT));
            zframe_destroy (&header);

            zframe_t *reply_service = zmsg_pop (msg);
            assert (zframe_streq (reply_service, service));
            zframe_destroy (&reply_service);

            zmsg_destroy (&request);
            return msg;     //  Success
        }
        else
        if (--retries_left) {
            if (self->verbose)
                zclock_log ("W: no reply, reconnecting...");
            s_mdcli_connect_to_broker (self);
        }
        else {
            if (self->verbose)
                zclock_log ("W: permanent error, abandoning");
            break;          //  Give up
        }
    }
    if (zctx_interrupted)
        printf ("W: interrupt received, killing client...\n");
    zmsg_destroy (&request);
    return NULL;
}
```

让我们看看客户端API的运行情况,以一个测试程序为例,做100K的请求-回复循环.

```c
//  Majordomo Protocol client example
//  Uses the mdcli API to hide all MDP aspects

//  Lets us build this source without creating a library
#include"mdcliapi.c"

int main (int argc, char *argv [])
{
    int verbose = (argc > 1 && streq (argv [1],"-v"));
    mdcli_t *session = mdcli_new ("tcp://localhost:5555", verbose);

    int count;
    for (count = 0; count < 100000; count++) {
        zmsg_t *request = zmsg_new ();
        zmsg_pushstr (request,"Hello world");
        zmsg_t *reply = mdcli_send (session,"echo", &request);
        if (reply)
            zmsg_destroy (&reply);
        else
            break;              //  Interrupt or failure
    }
    printf ("%d requests/replies processed\n", count);
    mdcli_destroy (&session);
    return 0;
}
```

这里是worker的API.

```c
//  mdwrkapi class - Majordomo Protocol Worker API
//  Implements the MDP/Worker spec at http://rfc.zeromq.org/spec:7.

#include"mdwrkapi.h"

//  Reliability parameters
#define HEARTBEAT_LIVENESS  3       //  3-5 is reasonable

//  .split worker class structure
//  This is the structure of a worker API instance. We use a pseudo-OO
//  approach in a lot of the C examples, as well as the CZMQ binding:

//  Structure of our class
//  We access these properties only via class methods

struct _mdwrk_t {
    zctx_t *ctx;                //  Our context
    char *broker;
    char *service;
    void *worker;               //  Socket to broker
    int verbose;                //  Print activity to stdout

    //  Heartbeat management
    uint64_t heartbeat_at;      //  When to send HEARTBEAT
    size_t liveness;            //  How many attempts left
    int heartbeat;              //  Heartbeat delay, msecs
    int reconnect;              //  Reconnect delay, msecs

    int expect_reply;           //  Zero only at start
    zframe_t *reply_to;         //  Return identity, if any
};

//  .split utility functions
//  We have two utility functions; to send a message to the broker and
//  to (re)connect to the broker:

//  Send message to broker
//  If no msg is provided, creates one internally

static void
s_mdwrk_send_to_broker (mdwrk_t *self, char *command, char *option,
                        zmsg_t *msg)
{
    msg = msg? zmsg_dup (msg): zmsg_new ();

    //  Stack protocol envelope to start of message
    if (option)
        zmsg_pushstr (msg, option);
    zmsg_pushstr (msg, command);
    zmsg_pushstr (msg, MDPW_WORKER);
    zmsg_pushstr (msg,"");

    if (self->verbose) {
        zclock_log ("I: sending %s to broker",
            mdps_commands [(int) *command]);
        zmsg_dump (msg);
    }
    zmsg_send (&msg, self->worker);
}

//  Connect or reconnect to broker

void s_mdwrk_connect_to_broker (mdwrk_t *self)
{
    if (self->worker)
        zsocket_destroy (self->ctx, self->worker);
    self->worker = zsocket_new (self->ctx, ZMQ_DEALER);
    zmq_connect (self->worker, self->broker);
    if (self->verbose)
        zclock_log ("I: connecting to broker at %s...", self->broker);

    //  Register service with broker
    s_mdwrk_send_to_broker (self, MDPW_READY, self->service, NULL);

    //  If liveness hits zero, queue is considered disconnected
    self->liveness = HEARTBEAT_LIVENESS;
    self->heartbeat_at = zclock_time () + self->heartbeat;
}

//  .split constructor and destructor
//  Here we have the constructor and destructor for our mdwrk class:

//  Constructor

mdwrk_t *
mdwrk_new (char *broker,char *service, int verbose)
{
    assert (broker);
    assert (service);

    mdwrk_t *self = (mdwrk_t *) zmalloc (sizeof (mdwrk_t));
    self->ctx = zctx_new ();
    self->broker = strdup (broker);
    self->service = strdup (service);
    self->verbose = verbose;
    self->heartbeat = 2500;     //  msecs
    self->reconnect = 2500;     //  msecs

    s_mdwrk_connect_to_broker (self);
    return self;
}

//  Destructor

void
mdwrk_destroy (mdwrk_t **self_p)
{
    assert (self_p);
    if (*self_p) {
        mdwrk_t *self = *self_p;
        zctx_destroy (&self->ctx);
        free (self->broker);
        free (self->service);
        free (self);
        *self_p = NULL;
    }
}

//  .split configure worker
//  We provide two methods to configure the worker API. You can set the
//  heartbeat interval and retries to match the expected network performance.

//  Set heartbeat delay

void
mdwrk_set_heartbeat (mdwrk_t *self, int heartbeat)
{
    self->heartbeat = heartbeat;
}

//  Set reconnect delay

void
mdwrk_set_reconnect (mdwrk_t *self, int reconnect)
{
    self->reconnect = reconnect;
}

//  .split recv method
//  This is the {{recv}} method; it's a little misnamed because it first sends
//  any reply and then waits for a new request. If you have a better name
//  for this, let me know.

//  Send reply, if any, to broker and wait for next request.

zmsg_t *
mdwrk_recv (mdwrk_t *self, zmsg_t **reply_p)
{
    //  Format and send the reply if we were provided one
    assert (reply_p);
    zmsg_t *reply = *reply_p;
    assert (reply || !self->expect_reply);
    if (reply) {
        assert (self->reply_to);
        zmsg_wrap (reply, self->reply_to);
        s_mdwrk_send_to_broker (self, MDPW_REPLY, NULL, reply);
        zmsg_destroy (reply_p);
    }
    self->expect_reply = 1;

    while (true) {
        zmq_pollitem_t items [] = {
            { self->worker,  0, ZMQ_POLLIN, 0 } };
        int rc = zmq_poll (items, 1, self->heartbeat * ZMQ_POLL_MSEC);
        if (rc == -1)
            break;              //  Interrupted

        if (items [0].revents & ZMQ_POLLIN) {
            zmsg_t *msg = zmsg_recv (self->worker);
            if (!msg)
                break;          //  Interrupted
            if (self->verbose) {
                zclock_log ("I: received message from broker:");
                zmsg_dump (msg);
            }
            self->liveness = HEARTBEAT_LIVENESS;

            //  Don't try to handle errors, just assert noisily
            assert (zmsg_size (msg) >= 3);

            zframe_t *empty = zmsg_pop (msg);
            assert (zframe_streq (empty,""));
            zframe_destroy (&empty);

            zframe_t *header = zmsg_pop (msg);
            assert (zframe_streq (header, MDPW_WORKER));
            zframe_destroy (&header);

            zframe_t *command = zmsg_pop (msg);
            if (zframe_streq (command, MDPW_REQUEST)) {
                //  We should pop and save as many addresses as there are
                //  up to a null part, but for now, just save one...
                self->reply_to = zmsg_unwrap (msg);
                zframe_destroy (&command);
                //  .split process message
                //  Here is where we actually have a message to process; we
                //  return it to the caller application:
                
                return msg;     //  We have a request to process
            }
            else
            if (zframe_streq (command, MDPW_HEARTBEAT))
                ;               //  Do nothing for heartbeats
            else
            if (zframe_streq (command, MDPW_DISCONNECT))
                s_mdwrk_connect_to_broker (self);
            else {
                zclock_log ("E: invalid input message");
                zmsg_dump (msg);
            }
            zframe_destroy (&command);
            zmsg_destroy (&msg);
        }
        else
        if (--self->liveness == 0) {
            if (self->verbose)
                zclock_log ("W: disconnected from broker - retrying...");
            zclock_sleep (self->reconnect);
            s_mdwrk_connect_to_broker (self);
        }
        //  Send HEARTBEAT if it's time
        if (zclock_time () > self->heartbeat_at) {
            s_mdwrk_send_to_broker (self, MDPW_HEARTBEAT, NULL, NULL);
            self->heartbeat_at = zclock_time () + self->heartbeat;
        }
    }
    if (zctx_interrupted)
        printf ("W: interrupt received, killing worker...\n");
    return NULL;
}
```

让我们通过一个实现回声服务的测试程序的例子,来看看worker API的运行情况.

```c
//  mdwrkapi class - Majordomo Protocol Worker API
//  Implements the MDP/Worker spec at http://rfc.zeromq.org/spec:7.

#include"mdwrkapi.h"

//  Reliability parameters
#define HEARTBEAT_LIVENESS  3       //  3-5 is reasonable

//  .split worker class structure
//  This is the structure of a worker API instance. We use a pseudo-OO
//  approach in a lot of the C examples, as well as the CZMQ binding:

//  Structure of our class
//  We access these properties only via class methods

struct _mdwrk_t {
    zctx_t *ctx;                //  Our context
    char *broker;
    char *service;
    void *worker;               //  Socket to broker
    int verbose;                //  Print activity to stdout

    //  Heartbeat management
    uint64_t heartbeat_at;      //  When to send HEARTBEAT
    size_t liveness;            //  How many attempts left
    int heartbeat;              //  Heartbeat delay, msecs
    int reconnect;              //  Reconnect delay, msecs

    int expect_reply;           //  Zero only at start
    zframe_t *reply_to;         //  Return identity, if any
};

//  .split utility functions
//  We have two utility functions; to send a message to the broker and
//  to (re)connect to the broker:

//  Send message to broker
//  If no msg is provided, creates one internally

static void
s_mdwrk_send_to_broker (mdwrk_t *self, char *command, char *option,
                        zmsg_t *msg)
{
    msg = msg? zmsg_dup (msg): zmsg_new ();

    //  Stack protocol envelope to start of message
    if (option)
        zmsg_pushstr (msg, option);
    zmsg_pushstr (msg, command);
    zmsg_pushstr (msg, MDPW_WORKER);
    zmsg_pushstr (msg,"");

    if (self->verbose) {
        zclock_log ("I: sending %s to broker",
            mdps_commands [(int) *command]);
        zmsg_dump (msg);
    }
    zmsg_send (&msg, self->worker);
}

//  Connect or reconnect to broker

void s_mdwrk_connect_to_broker (mdwrk_t *self)
{
    if (self->worker)
        zsocket_destroy (self->ctx, self->worker);
    self->worker = zsocket_new (self->ctx, ZMQ_DEALER);
    zmq_connect (self->worker, self->broker);
    if (self->verbose)
        zclock_log ("I: connecting to broker at %s...", self->broker);

    //  Register service with broker
    s_mdwrk_send_to_broker (self, MDPW_READY, self->service, NULL);

    //  If liveness hits zero, queue is considered disconnected
    self->liveness = HEARTBEAT_LIVENESS;
    self->heartbeat_at = zclock_time () + self->heartbeat;
}

//  .split constructor and destructor
//  Here we have the constructor and destructor for our mdwrk class:

//  Constructor

mdwrk_t *
mdwrk_new (char *broker,char *service, int verbose)
{
    assert (broker);
    assert (service);

    mdwrk_t *self = (mdwrk_t *) zmalloc (sizeof (mdwrk_t));
    self->ctx = zctx_new ();
    self->broker = strdup (broker);
    self->service = strdup (service);
    self->verbose = verbose;
    self->heartbeat = 2500;     //  msecs
    self->reconnect = 2500;     //  msecs

    s_mdwrk_connect_to_broker (self);
    return self;
}

//  Destructor

void
mdwrk_destroy (mdwrk_t **self_p)
{
    assert (self_p);
    if (*self_p) {
        mdwrk_t *self = *self_p;
        zctx_destroy (&self->ctx);
        free (self->broker);
        free (self->service);
        free (self);
        *self_p = NULL;
    }
}

//  .split configure worker
//  We provide two methods to configure the worker API. You can set the
//  heartbeat interval and retries to match the expected network performance.

//  Set heartbeat delay

void
mdwrk_set_heartbeat (mdwrk_t *self, int heartbeat)
{
    self->heartbeat = heartbeat;
}

//  Set reconnect delay

void
mdwrk_set_reconnect (mdwrk_t *self, int reconnect)
{
    self->reconnect = reconnect;
}

//  .split recv method
//  This is the {{recv}} method; it's a little misnamed because it first sends
//  any reply and then waits for a new request. If you have a better name
//  for this, let me know.

//  Send reply, if any, to broker and wait for next request.

zmsg_t *
mdwrk_recv (mdwrk_t *self, zmsg_t **reply_p)
{
    //  Format and send the reply if we were provided one
    assert (reply_p);
    zmsg_t *reply = *reply_p;
    assert (reply || !self->expect_reply);
    if (reply) {
        assert (self->reply_to);
        zmsg_wrap (reply, self->reply_to);
        s_mdwrk_send_to_broker (self, MDPW_REPLY, NULL, reply);
        zmsg_destroy (reply_p);
    }
    self->expect_reply = 1;

    while (true) {
        zmq_pollitem_t items [] = {
            { self->worker,  0, ZMQ_POLLIN, 0 } };
        int rc = zmq_poll (items, 1, self->heartbeat * ZMQ_POLL_MSEC);
        if (rc == -1)
            break;              //  Interrupted

        if (items [0].revents & ZMQ_POLLIN) {
            zmsg_t *msg = zmsg_recv (self->worker);
            if (!msg)
                break;          //  Interrupted
            if (self->verbose) {
                zclock_log ("I: received message from broker:");
                zmsg_dump (msg);
            }
            self->liveness = HEARTBEAT_LIVENESS;

            //  Don't try to handle errors, just assert noisily
            assert (zmsg_size (msg) >= 3);

            zframe_t *empty = zmsg_pop (msg);
            assert (zframe_streq (empty,""));
            zframe_destroy (&empty);

            zframe_t *header = zmsg_pop (msg);
            assert (zframe_streq (header, MDPW_WORKER));
            zframe_destroy (&header);

            zframe_t *command = zmsg_pop (msg);
            if (zframe_streq (command, MDPW_REQUEST)) {
                //  We should pop and save as many addresses as there are
                //  up to a null part, but for now, just save one...
                self->reply_to = zmsg_unwrap (msg);
                zframe_destroy (&command);
                //  .split process message
                //  Here is where we actually have a message to process; we
                //  return it to the caller application:
                
                return msg;     //  We have a request to process
            }
            else
            if (zframe_streq (command, MDPW_HEARTBEAT))
                ;               //  Do nothing for heartbeats
            else
            if (zframe_streq (command, MDPW_DISCONNECT))
                s_mdwrk_connect_to_broker (self);
            else {
                zclock_log ("E: invalid input message");
                zmsg_dump (msg);
            }
            zframe_destroy (&command);
            zmsg_destroy (&msg);
        }
        else
        if (--self->liveness == 0) {
            if (self->verbose)
                zclock_log ("W: disconnected from broker - retrying...");
            zclock_sleep (self->reconnect);
            s_mdwrk_connect_to_broker (self);
        }
        //  Send HEARTBEAT if it's time
        if (zclock_time () > self->heartbeat_at) {
            s_mdwrk_send_to_broker (self, MDPW_HEARTBEAT, NULL, NULL);
            self->heartbeat_at = zclock_time () + self->heartbeat;
        }
    }
    if (zctx_interrupted)
        printf ("W: interrupt received, killing worker...\n");
    return NULL;
}
```

这里有一些关于workerAPI代码需要注意的地方.

* 这些API是单线程的.这意味着,例如,worker不会在后台发送心跳.令人高兴的是,这正是我们想要的:如果worker应用程序被卡住,心跳将停止,代理将停止向worker发送请求.

* workerAPI不做指数级的回退；这不值得额外的复杂性.

* 这些API不做任何错误报告.如果有不符合预期的情况,它们会提出一个断言(或异常,取决于语言).这对参考实现来说是理想的,所以任何协议错误都会立即显示出来.对于真正的应用来说,API 应该对无效的消息有很好的保护.

你可能想知道为什么workerAPI要手动关闭其套接字并打开一个新的套接字,而ZeroMQ会在对等体消失并回来时自动重新连接套接字.回顾一下简单海盗和偏执海盗的工作器就会明白.尽管ZeroMQ会在代理机死亡后又恢复的情况下自动重新连接worker,但这还不足以让worker重新在代理机上注册.我知道至少有两种解决方案.最简单的,也是我们在这里使用的,就是让worker使用心跳来监控连接,如果它认为代理已经死亡,就关闭它的套接字,用一个新的套接字重新开始.另一种方法是,当borkers收到来自worker的心跳时,挑战未知的worker,要求他们重新注册.这将需要协议支持.

现在让我们来设计Majordomo代理.它的核心结构是一组队列,每个服务一个.我们将在worker出现时创建这些队列(我们可以在worker消失时删除它们,但现在忘了这一点,因为这会变得很复杂).此外,我们为每个服务保留一个工人队列.

而这里是borkers.

```c
//  Majordomo Protocol broker
//  A minimal C implementation of the Majordomo Protocol as defined in
//  http://rfc.zeromq.org/spec:7 and http://rfc.zeromq.org/spec:8.

#include"czmq.h"
#include"mdp.h"

//  We'd normally pull these from config data

#define HEARTBEAT_LIVENESS  3       //  3-5 is reasonable
#define HEARTBEAT_INTERVAL  2500    //  msecs
#define HEARTBEAT_EXPIRY    HEARTBEAT_INTERVAL * HEARTBEAT_LIVENESS

//  .split broker class structure
//  The broker class defines a single broker instance:

typedef struct {
    zctx_t *ctx;                //  Our context
    void *socket;               //  Socket for clients & workers
    int verbose;                //  Print activity to stdout
    char *endpoint;             //  Broker binds to this endpoint
    zhash_t *services;          //  Hash of known services
    zhash_t *workers;           //  Hash of known workers
    zlist_t *waiting;           //  List of waiting workers
    uint64_t heartbeat_at;      //  When to send HEARTBEAT
} broker_t;

static broker_t *
    s_broker_new (int verbose);
static void
    s_broker_destroy (broker_t **self_p);
static void
    s_broker_bind (broker_t *self, char *endpoint);
static void
    s_broker_worker_msg (broker_t *self, zframe_t *sender, zmsg_t *msg);
static void
    s_broker_client_msg (broker_t *self, zframe_t *sender, zmsg_t *msg);
static void
    s_broker_purge (broker_t *self);

//  .split service class structure
//  The service class defines a single service instance:

typedef struct {
    broker_t *broker;           //  Broker instance
    char *name;                 //  Service name
    zlist_t *requests;          //  List of client requests
    zlist_t *waiting;           //  List of waiting workers
    size_t workers;             //  How many workers we have
} service_t;

static service_t *
    s_service_require (broker_t *self, zframe_t *service_frame);
static void
    s_service_destroy (void *argument);
static void
    s_service_dispatch (service_t *service, zmsg_t *msg);

//  .split worker class structure
//  The worker class defines a single worker, idle or active:

typedef struct {
    broker_t *broker;           //  Broker instance
    char *id_string;            //  Identity of worker as string
    zframe_t *identity;         //  Identity frame for routing
    service_t *service;         //  Owning service, if known
    int64_t expiry;             //  When worker expires, if no heartbeat
} worker_t;

static worker_t *
    s_worker_require (broker_t *self, zframe_t *identity);
static void
    s_worker_delete (worker_t *self, int disconnect);
static void
    s_worker_destroy (void *argument);
static void
    s_worker_send (worker_t *self, char *command, char *option,
                   zmsg_t *msg);
static void
    s_worker_waiting (worker_t *self);

//  .split broker constructor and destructor
//  Here are the constructor and destructor for the broker:

static broker_t *
s_broker_new (int verbose)
{
    broker_t *self = (broker_t *) zmalloc (sizeof (broker_t));

    //  Initialize broker state
    self->ctx = zctx_new ();
    self->socket = zsocket_new (self->ctx, ZMQ_ROUTER);
    self->verbose = verbose;
    self->services = zhash_new ();
    self->workers = zhash_new ();
    self->waiting = zlist_new ();
    self->heartbeat_at = zclock_time () + HEARTBEAT_INTERVAL;
    return self;
}

static void
s_broker_destroy (broker_t **self_p)
{
    assert (self_p);
    if (*self_p) {
        broker_t *self = *self_p;
        zctx_destroy (&self->ctx);
        zhash_destroy (&self->services);
        zhash_destroy (&self->workers);
        zlist_destroy (&self->waiting);
        free (self);
        *self_p = NULL;
    }
}

//  .split broker bind method
//  This method binds the broker instance to an endpoint. We can call
//  this multiple times. Note that MDP uses a single socket for both clients 
//  and workers:

void
s_broker_bind (broker_t *self, char *endpoint)
{
    zsocket_bind (self->socket, endpoint);
    zclock_log ("I: MDP broker/0.2.0 is active at %s", endpoint);
}

//  .split broker worker_msg method
//  This method processes one READY, REPLY, HEARTBEAT, or
//  DISCONNECT message sent to the broker by a worker:

static void
s_broker_worker_msg (broker_t *self, zframe_t *sender, zmsg_t *msg)
{
    assert (zmsg_size (msg) >= 1);     //  At least, command

    zframe_t *command = zmsg_pop (msg);
    char *id_string = zframe_strhex (sender);
    int worker_ready = (zhash_lookup (self->workers, id_string) != NULL);
    free (id_string);
    worker_t *worker = s_worker_require (self, sender);

    if (zframe_streq (command, MDPW_READY)) {
        if (worker_ready)               //  Not first command in session
            s_worker_delete (worker, 1);
        else
        if (zframe_size (sender) >= 4  //  Reserved service name
        &&  memcmp (zframe_data (sender),"mmi.", 4) == 0)
            s_worker_delete (worker, 1);
        else {
            //  Attach worker to service and mark as idle
            zframe_t *service_frame = zmsg_pop (msg);
            worker->service = s_service_require (self, service_frame);
            worker->service->workers++;
            s_worker_waiting (worker);
            zframe_destroy (&service_frame);
        }
    }
    else
    if (zframe_streq (command, MDPW_REPLY)) {
        if (worker_ready) {
            //  Remove and save client return envelope and insert the
            //  protocol header and service name, then rewrap envelope.
            zframe_t *client = zmsg_unwrap (msg);
            zmsg_pushstr (msg, worker->service->name);
            zmsg_pushstr (msg, MDPC_CLIENT);
            zmsg_wrap (msg, client);
            zmsg_send (&msg, self->socket);
            s_worker_waiting (worker);
        }
        else
            s_worker_delete (worker, 1);
    }
    else
    if (zframe_streq (command, MDPW_HEARTBEAT)) {
        if (worker_ready)
            worker->expiry = zclock_time () + HEARTBEAT_EXPIRY;
        else
            s_worker_delete (worker, 1);
    }
    else
    if (zframe_streq (command, MDPW_DISCONNECT))
        s_worker_delete (worker, 0);
    else {
        zclock_log ("E: invalid input message");
        zmsg_dump (msg);
    }
    free (command);
    zmsg_destroy (&msg);
}

//  .split broker client_msg method
//  Process a request coming from a client. We implement MMI requests
//  directly here (at present, we implement only the mmi.service request):

static void
s_broker_client_msg (broker_t *self, zframe_t *sender, zmsg_t *msg)
{
    assert (zmsg_size (msg) >= 2);     //  Service name + body

    zframe_t *service_frame = zmsg_pop (msg);
    service_t *service = s_service_require (self, service_frame);

    //  Set reply return identity to client sender
    zmsg_wrap (msg, zframe_dup (sender));

    //  If we got a MMI service request, process that internally
    if (zframe_size (service_frame) >= 4
    &&  memcmp (zframe_data (service_frame),"mmi.", 4) == 0) {
        char *return_code;
        if (zframe_streq (service_frame,"mmi.service")) {
            char *name = zframe_strdup (zmsg_last (msg));
            service_t *service =
                (service_t *) zhash_lookup (self->services, name);
            return_code = service && service->workers?"200":"404";
            free (name);
        }
        else
            return_code ="501";

        zframe_reset (zmsg_last (msg), return_code, strlen (return_code));

        //  Remove & save client return envelope and insert the
        //  protocol header and service name, then rewrap envelope.
        zframe_t *client = zmsg_unwrap (msg);
        zmsg_prepend (msg, &service_frame);
        zmsg_pushstr (msg, MDPC_CLIENT);
        zmsg_wrap (msg, client);
        zmsg_send (&msg, self->socket);
    }
    else
        //  Else dispatch the message to the requested service
        s_service_dispatch (service, msg);
    zframe_destroy (&service_frame);
}

//  .split broker purge method
//  This method deletes any idle workers that haven't pinged us in a
//  while. We hold workers from oldest to most recent so we can stop
//  scanning whenever we find a live worker. This means we'll mainly stop
//  at the first worker, which is essential when we have large numbers of
//  workers (we call this method in our critical path):

static void
s_broker_purge (broker_t *self)
{
    worker_t *worker = (worker_t *) zlist_first (self->waiting);
    while (worker) {
        if (zclock_time () < worker->expiry)
            break;                  //  Worker is alive, we're done here
        if (self->verbose)
            zclock_log ("I: deleting expired worker: %s",
                        worker->id_string);

        s_worker_delete (worker, 0);
        worker = (worker_t *) zlist_first (self->waiting);
    }
}

//  .split service methods
//  Here is the implementation of the methods that work on a service:

//  Lazy constructor that locates a service by name or creates a new
//  service if there is no service already with that name.

static service_t *
s_service_require (broker_t *self, zframe_t *service_frame)
{
    assert (service_frame);
    char *name = zframe_strdup (service_frame);

    service_t *service =
        (service_t *) zhash_lookup (self->services, name);
    if (service == NULL) {
        service = (service_t *) zmalloc (sizeof (service_t));
        service->broker = self;
        service->name = name;
        service->requests = zlist_new ();
        service->waiting = zlist_new ();
        zhash_insert (self->services, name, service);
        zhash_freefn (self->services, name, s_service_destroy);
        if (self->verbose)
            zclock_log ("I: added service: %s", name);
    }
    else
        free (name);

    return service;
}

//  Service destructor is called automatically whenever the service is
//  removed from broker->services.

static void
s_service_destroy (void *argument)
{
    service_t *service = (service_t *) argument;
    while (zlist_size (service->requests)) {
        zmsg_t *msg = zlist_pop (service->requests);
        zmsg_destroy (&msg);
    }
    zlist_destroy (&service->requests);
    zlist_destroy (&service->waiting);
    free (service->name);
    free (service);
}

//  .split service dispatch method
//  This method sends requests to waiting workers:

static void
s_service_dispatch (service_t *self, zmsg_t *msg)
{
    assert (self);
    if (msg)                    //  Queue message if any
        zlist_append (self->requests, msg);

    s_broker_purge (self->broker);
    while (zlist_size (self->waiting) && zlist_size (self->requests)) {
        worker_t *worker = zlist_pop (self->waiting);
        zlist_remove (self->broker->waiting, worker);
        zmsg_t *msg = zlist_pop (self->requests);
        s_worker_send (worker, MDPW_REQUEST, NULL, msg);
        zmsg_destroy (&msg);
    }
}

//  .split worker methods
//  Here is the implementation of the methods that work on a worker:

//  Lazy constructor that locates a worker by identity, or creates a new
//  worker if there is no worker already with that identity.

static worker_t *
s_worker_require (broker_t *self, zframe_t *identity)
{
    assert (identity);

    //  self->workers is keyed off worker identity
    char *id_string = zframe_strhex (identity);
    worker_t *worker =
        (worker_t *) zhash_lookup (self->workers, id_string);

    if (worker == NULL) {
        worker = (worker_t *) zmalloc (sizeof (worker_t));
        worker->broker = self;
        worker->id_string = id_string;
        worker->identity = zframe_dup (identity);
        zhash_insert (self->workers, id_string, worker);
        zhash_freefn (self->workers, id_string, s_worker_destroy);
        if (self->verbose)
            zclock_log ("I: registering new worker: %s", id_string);
    }
    else
        free (id_string);
    return worker;
}

//  This method deletes the current worker.

static void
s_worker_delete (worker_t *self, int disconnect)
{
    assert (self);
    if (disconnect)
        s_worker_send (self, MDPW_DISCONNECT, NULL, NULL);

    if (self->service) {
        zlist_remove (self->service->waiting, self);
        self->service->workers--;
    }
    zlist_remove (self->broker->waiting, self);
    //  This implicitly calls s_worker_destroy
    zhash_delete (self->broker->workers, self->id_string);
}

//  Worker destructor is called automatically whenever the worker is
//  removed from broker->workers.

static void
s_worker_destroy (void *argument)
{
    worker_t *self = (worker_t *) argument;
    zframe_destroy (&self->identity);
    free (self->id_string);
    free (self);
}

//  .split worker send method
//  This method formats and sends a command to a worker. The caller may
//  also provide a command option, and a message payload:

static void
s_worker_send (worker_t *self, char *command, char *option, zmsg_t *msg)
{
    msg = msg? zmsg_dup (msg): zmsg_new ();

    //  Stack protocol envelope to start of message
    if (option)
        zmsg_pushstr (msg, option);
    zmsg_pushstr (msg, command);
    zmsg_pushstr (msg, MDPW_WORKER);

    //  Stack routing envelope to start of message
    zmsg_wrap (msg, zframe_dup (self->identity));

    if (self->broker->verbose) {
        zclock_log ("I: sending %s to worker",
            mdps_commands [(int) *command]);
        zmsg_dump (msg);
    }
    zmsg_send (&msg, self->broker->socket);
}

//  This worker is now waiting for work

static void
s_worker_waiting (worker_t *self)
{
    //  Queue to broker and service waiting lists
    assert (self->broker);
    zlist_append (self->broker->waiting, self);
    zlist_append (self->service->waiting, self);
    self->expiry = zclock_time () + HEARTBEAT_EXPIRY;
    s_service_dispatch (self->service, NULL);
}

//  .split main task
//  Finally, here is the main task. We create a new broker instance and
//  then process messages on the broker socket:

int main (int argc, char *argv [])
{
    int verbose = (argc > 1 && streq (argv [1],"-v"));

    broker_t *self = s_broker_new (verbose);
    s_broker_bind (self,"tcp://*:5555");

    //  Get and process messages forever or until interrupted
    while (true) {
        zmq_pollitem_t items [] = {
            { self->socket,  0, ZMQ_POLLIN, 0 } };
        int rc = zmq_poll (items, 1, HEARTBEAT_INTERVAL * ZMQ_POLL_MSEC);
        if (rc == -1)
            break;              //  Interrupted

        //  Process next input message, if any
        if (items [0].revents & ZMQ_POLLIN) {
            zmsg_t *msg = zmsg_recv (self->socket);
            if (!msg)
                break;          //  Interrupted
            if (self->verbose) {
                zclock_log ("I: received message:");
                zmsg_dump (msg);
            }
            zframe_t *sender = zmsg_pop (msg);
            zframe_t *empty  = zmsg_pop (msg);
            zframe_t *header = zmsg_pop (msg);

            if (zframe_streq (header, MDPC_CLIENT))
                s_broker_client_msg (self, sender, msg);
            else
            if (zframe_streq (header, MDPW_WORKER))
                s_broker_worker_msg (self, sender, msg);
            else {
                zclock_log ("E: invalid message:");
                zmsg_dump (msg);
                zmsg_destroy (&msg);
            }
            zframe_destroy (&sender);
            zframe_destroy (&empty);
            zframe_destroy (&header);
        }
        //  Disconnect and delete any expired workers
        //  Send heartbeats to idle workers if needed
        if (zclock_time () > self->heartbeat_at) {
            s_broker_purge (self);
            worker_t *worker = (worker_t *) zlist_first (self->waiting);
            while (worker) {
                s_worker_send (worker, MDPW_HEARTBEAT, NULL, NULL);
                worker = (worker_t *) zlist_next (self->waiting);
            }
            self->heartbeat_at = zclock_time () + HEARTBEAT_INTERVAL;
        }
    }
    if (zctx_interrupted)
        printf ("W: interrupt received, shutting down...\n");

    s_broker_destroy (&self);
    return 0;
}
```

这是迄今为止我们所见过的最复杂的例子.它有将近500行的代码.为了写这个并使其具有一定的健壮性,花了两天时间.然而,对于一个完整的面向服务的borkers来说,这仍然是一段简短的代码.

这里有一些关于borkers代码的事情需要注意.

* Majordomo协议让我们在一个套接字上同时处理客户和worker.这对部署和管理代理的人来说更有利:它只需坐在一个ZeroMQ端点上,而不是大多数代理需要的两个端点.

* 代理商正确地实现了MDP/0.1的所有功能(据我所知),包括在代理发送无效命令时断开连接,心跳,以及其他功能.

* 它可以被扩展到运行多个线程,每个线程管理一个套接字和一组客户和worker.这对于大型架构的分割可能很有趣.C代码已经围绕着一个代理类组织起来,使之变得简单.

* 一个主要的/失效的或活的/活的代理的可靠性模型是很容易的,因为代理基本上没有任何状态,除了服务存在.如果客户和worker的第一选择没有启动和运行,就由他们选择另一个代理.

* 例子中使用了5秒钟的心跳,主要是为了减少你启用跟踪时的输出量.对于大多数局域网应用来说,现实的数值会更低.然而,任何重试都必须足够慢,以允许服务重新启动,比如至少10秒.

我们后来改进和扩展了协议和Majordomo的实现,现在它位于自己的Github项目中.如果你想要一个正确可用的Majordomo栈,请使用GitHub项目.

## 异步的Majordomo模式

上一节中的Majordomo实现是简单而愚蠢的.客户端只是原来的简单海盗,被包裹在一个性感的API中.当我在一个测试箱上启动客户端,代理和worker时,它可以在大约14秒内处理100,000个请求.这部分是由于代码的原因,它欢快地复制消息帧,好像CPU周期是免费的.但真正的问题是,我们正在进行网络往返.ZeroMQ禁用了[http://en.wikipedia.org/wiki/Nagles_algorithm Nagle's algorithm],但round-tripping仍然很慢.

理论上是很好的,但在实践中,实践是更好的.让我们用一个简单的测试程序来衡量往返的实际成本.这个程序发送了一堆消息,首先是等待每条消息的回复,其次是作为一个批次,把所有的回复作为一个批次读回来.这两种方法做的是同样的工作,但它们给出了非常不同的结果.我们模拟了一个客户端,代理和worker.

```c
//  Round-trip demonstrator
//  While this example runs in a single process, that is just to make
//  it easier to start and stop the example. The client task signals to
//  main when it's ready.

#include"czmq.h"

static void
client_task (void *args, zctx_t *ctx, void *pipe)
{
    void *client = zsocket_new (ctx, ZMQ_DEALER);
    zsocket_connect (client,"tcp://localhost:5555");
    printf ("Setting up test...\n");
    zclock_sleep (100);

    int requests;
    int64_t start;

    printf ("Synchronous round-trip test...\n");
    start = zclock_time ();
    for (requests = 0; requests < 10000; requests++) {
        zstr_send (client,"hello");
        char *reply = zstr_recv (client);
        free (reply);
    }
    printf (" %d calls/second\n",
        (1000 * 10000) / (int) (zclock_time () - start));

    printf ("Asynchronous round-trip test...\n");
    start = zclock_time ();
    for (requests = 0; requests < 100000; requests++)
        zstr_send (client,"hello");
    for (requests = 0; requests < 100000; requests++) {
        char *reply = zstr_recv (client);
        free (reply);
    }
    printf (" %d calls/second\n",
        (1000 * 100000) / (int) (zclock_time () - start));
    zstr_send (pipe,"done");
}

//  .split worker task
//  Here is the worker task. All it does is receive a message, and
//  bounce it back the way it came:

static void *
worker_task (void *args)
{
    zctx_t *ctx = zctx_new ();
    void *worker = zsocket_new (ctx, ZMQ_DEALER);
    zsocket_connect (worker,"tcp://localhost:5556");
    
    while (true) {
        zmsg_t *msg = zmsg_recv (worker);
        zmsg_send (&msg, worker);
    }
    zctx_destroy (&ctx);
    return NULL;
}

//  .split broker task
//  Here is the broker task. It uses the {{zmq_proxy}} function to switch
//  messages between frontend and backend:

static void *
broker_task (void *args)
{
    //  Prepare our context and sockets
    zctx_t *ctx = zctx_new ();
    void *frontend = zsocket_new (ctx, ZMQ_DEALER);
    zsocket_bind (frontend,"tcp://*:5555");
    void *backend = zsocket_new (ctx, ZMQ_DEALER);
    zsocket_bind (backend,"tcp://*:5556");
    zmq_proxy (frontend, backend, NULL);
    zctx_destroy (&ctx);
    return NULL;
}

//  .split main task
//  Finally, here's the main task, which starts the client, worker, and
//  broker, and then runs until the client signals it to stop:

int main (void)
{
    //  Create threads
    zctx_t *ctx = zctx_new ();
    void *client = zthread_fork (ctx, client_task, NULL);
    zthread_new (worker_task, NULL);
    zthread_new (broker_task, NULL);

    //  Wait for signal on client pipe
    char *signal = zstr_recv (client);
    free (signal);

    zctx_destroy (&ctx);
    return 0;
}
```

在我的开发箱上,这个程序说.

```
Setting up test...
Synchronous round-trip test...
 9057 calls/second
Asynchronous round-trip test...
 173010 calls/second
```

请注意,客户端线程在启动前做了一个小小的暂停.这是为了绕过路由器套接字的一个"特性":如果你用一个尚未连接的对等体的地址发送消息,该消息会被丢弃.在这个例子中,我们没有使用负载平衡机制,所以如果没有睡眠,如果工作线程的连接速度太慢,它就会丢失消息,使我们的测试变得混乱.

正如我们所看到的,在最简单的情况下,round-tripping比异步的,"以最快的速度把它推入管道"的方法要慢20倍.让我们看看是否可以把这个方法应用于Majordomo,使其更快.

首先,我们修改客户端的API,使其在两个独立的方法中发送和接收.

```[[type="fragment" name="mdclient-async"]]
mdcli_t *mdcli_new(char *broker).
void mdcli_destroy (mdcli_t **self_p);
int mdcli_send (mdcli_t *self, char *service, zmsg_t **request_p);
zmsg_t *mdcli_recv (mdcli_t *self);
```

将同步客户端API重构为异步的,只需要几分钟的时间.

```c
//  mdcliapi2 class - Majordomo Protocol Client API
//  Implements the MDP/Worker spec at http://rfc.zeromq.org/spec:7.

#include"mdcliapi2.h"

//  Structure of our class
//  We access these properties only via class methods

struct _mdcli_t {
    zctx_t *ctx;                //  Our context
    char *broker;
    void *client;               //  Socket to broker
    int verbose;                //  Print activity to stdout
    int timeout;                //  Request timeout
};

//  Connect or reconnect to broker. In this asynchronous class we use a
//  DEALER socket instead of a REQ socket; this lets us send any number
//  of requests without waiting for a reply.

void s_mdcli_connect_to_broker (mdcli_t *self)
{
    if (self->client)
        zsocket_destroy (self->ctx, self->client);
    self->client = zsocket_new (self->ctx, ZMQ_DEALER);
    zmq_connect (self->client, self->broker);
    if (self->verbose)
        zclock_log ("I: connecting to broker at %s...", self->broker);
}

//  The constructor and destructor are the same as in mdcliapi, except
//  we don't do retries, so there's no retries property.
//  .skip
//  ---------------------------------------------------------------------
//  Constructor

mdcli_t *
mdcli_new (char *broker, int verbose)
{
    assert (broker);

    mdcli_t *self = (mdcli_t *) zmalloc (sizeof (mdcli_t));
    self->ctx = zctx_new ();
    self->broker = strdup (broker);
    self->verbose = verbose;
    self->timeout = 2500;           //  msecs

    s_mdcli_connect_to_broker (self);
    return self;
}

//  Destructor

void
mdcli_destroy (mdcli_t **self_p)
{
    assert (self_p);
    if (*self_p) {
        mdcli_t *self = *self_p;
        zctx_destroy (&self->ctx);
        free (self->broker);
        free (self);
        *self_p = NULL;
    }
}

//  Set request timeout

void
mdcli_set_timeout (mdcli_t *self, int timeout)
{
    assert (self);
    self->timeout = timeout;
}

//  .until
//  .skip
//  The send method now just sends one message, without waiting for a
//  reply. Since we're using a DEALER socket we have to send an empty
//  frame at the start, to create the same envelope that the REQ socket
//  would normally make for us:

int
mdcli_send (mdcli_t *self, char *service, zmsg_t **request_p)
{
    assert (self);
    assert (request_p);
    zmsg_t *request = *request_p;

    //  Prefix request with protocol frames
    //  Frame 0: empty (REQ emulation)
    //  Frame 1:"MDPCxy" (six bytes, MDP/Client x.y)
    //  Frame 2: Service name (printable string)
    zmsg_pushstr (request, service);
    zmsg_pushstr (request, MDPC_CLIENT);
    zmsg_pushstr (request,"");
    if (self->verbose) {
        zclock_log ("I: send request to '%s' service:", service);
        zmsg_dump (request);
    }
    zmsg_send (&request, self->client);
    return 0;
}

//  .skip
//  The recv method waits for a reply message and returns that to the 
//  caller.
//  ---------------------------------------------------------------------
//  Returns the reply message or NULL if there was no reply. Does not
//  attempt to recover from a broker failure, this is not possible
//  without storing all unanswered requests and resending them all...

zmsg_t *
mdcli_recv (mdcli_t *self)
{
    assert (self);

    //  Poll socket for a reply, with timeout
    zmq_pollitem_t items [] = { { self->client, 0, ZMQ_POLLIN, 0 } };
    int rc = zmq_poll (items, 1, self->timeout * ZMQ_POLL_MSEC);
    if (rc == -1)
        return NULL;            //  Interrupted

    //  If we got a reply, process it
    if (items [0].revents & ZMQ_POLLIN) {
        zmsg_t *msg = zmsg_recv (self->client);
        if (self->verbose) {
            zclock_log ("I: received reply:");
            zmsg_dump (msg);
        }
        //  Don't try to handle errors, just assert noisily
        assert (zmsg_size (msg) >= 4);

        zframe_t *empty = zmsg_pop (msg);
        assert (zframe_streq (empty,""));
        zframe_destroy (&empty);

        zframe_t *header = zmsg_pop (msg);
        assert (zframe_streq (header, MDPC_CLIENT));
        zframe_destroy (&header);

        zframe_t *service = zmsg_pop (msg);
        zframe_destroy (&service);

        return msg;     //  Success
    }
    if (zctx_interrupted)
        printf ("W: interrupt received, killing client...\n");
    else
    if (self->verbose)
        zclock_log ("W: permanent error, abandoning request");

    return NULL;
}
```

不同之处在于.

* 我们使用DEALER套接字,而不是REQ,所以我们在每个请求和每个响应之前用一个空的分隔符帧来模拟REQ.
* 我们不重试请求；如果应用程序需要重试,它可以自己做.
* 我们将同步的{{send}}方法分成独立的{{send}}和{{recv}}方法.
* {{send}}方法是异步的,发送后立即返回.因此,调用者可以在得到响应之前发送一些信息.
* {{recv}}方法等待(有超时)一个响应,并将其返回给调用者.

下面是相应的客户端测试程序,它发送了100,000条消息,然后接收了100,000条返回.

```c
//  Majordomo Protocol client example - asynchronous
//  Uses the mdcli API to hide all MDP aspects

//  Lets us build this source without creating a library
#include"mdcliapi2.c"

int main (int argc, char *argv [])
{
    int verbose = (argc > 1 && streq (argv [1],"-v"));
    mdcli_t *session = mdcli_new ("tcp://localhost:5555", verbose);

    int count;
    for (count = 0; count < 100000; count++) {
        zmsg_t *request = zmsg_new ();
        zmsg_pushstr (request,"Hello world");
        mdcli_send (session,"echo", &request);
    }
    for (count = 0; count < 100000; count++) {
        zmsg_t *reply = mdcli_recv (session);
        if (reply)
            zmsg_destroy (&reply);
        else
            break;              //  Interrupted by Ctrl-C
    }
    printf ("%d replies received\n", count);
    mdcli_destroy (&session);
    return 0;
}
```

borkers和worker没有变化,因为我们根本没有修改协议.我们看到性能立即得到了改善.下面是同步客户端在100K请求-回复周期中的运行情况.

```
$ time mdclient
100000 requests/replies processed

real    0m14.088s
user    0m1.310s
sys     0m2.670s
```

这里是异步客户端,只有一个worker.

```
$ time mdclient2
100000 replies received

real    0m8.730s
user    0m0.920s
sys     0m1.550s
```

两倍的速度.不错,但让我们启动10个工人,看看它如何处理流量

```
$ time mdclient2
100000 replies received

real    0m3.863s
user    0m0.730s
sys     0m0.470s
```

它不是完全异步的,因为worker在严格的最后使用的基础上获得他们的消息.但随着worker的增多,它的扩展性会更好.在我的电脑上,在八个左右的worker之后,它并没有变得更快.四个核心只能延伸到目前为止.但我们只用了几分钟的时间,吞吐量就提高了4倍.borkers仍然没有被优化.它把大部分时间都花在了复制消息帧上,而不是做零拷贝,这是它可以做到的.但是,我们在一秒钟内得到了25K个可靠的请求/回复调用,而且工作量相当小.

然而,异步的Majordomo模式并不全是玫瑰花.它有一个基本的弱点,即它不能在没有更多工作的情况下,在borkers崩溃时幸存下来.如果你看一下{{mdcliapi2}}代码,你会发现它并没有尝试在失败后重新连接.一个正确的重新连接需要以下条件.

* 每个请求都有一个编号,每个回复都有一个匹配的编号,这最好需要修改协议来强制执行.
* 跟踪并保留客户端API中所有未完成的请求,即那些尚未收到回复的请求.
* 在故障转移的情况下,客户端API要*重新发送*所有未完成的请求给代理.

这不是一个破坏性的问题,但它确实表明,性能往往意味着复杂性.这对Majordomo来说值得做吗？这取决于你的用例.对于一个每次调用一次的姓名查询服务,不值得.对于为成千上万的客户提供服务的网络前端来说,可能是的.

## 服务发现

所以,我们有一个很好的面向服务的代理,但我们没有办法知道一个特定的服务是否可用.我们知道一个请求是否失败,但我们不知道为什么.能够询问代理,"回声服务在运行吗？"是很有用的.最明显的方法是修改我们的MDP/Client协议,添加命令来问这个问题.但MDP/Client具有简单的巨大魅力.给它增加服务发现会使它和MDP/Worker协议一样复杂.

另一个选择是做电子邮件所做的,要求将无法送达的请求返回.这在异步世界中可以很好地工作,但它也增加了复杂性.我们需要一些方法来区分返回的请求和回复,并正确处理这些请求.

让我们尝试使用我们已经建立的东西,在MDP的基础上建立而不是修改它.服务发现本身就是一种服务.它可能确实是几种管理服务中的一种,比如"禁用服务X","提供统计数据",等等.我们想要的是一个通用的,可扩展的解决方案,不影响协议或现有的应用.

因此,这里有一个小的RFC,它把这个东西放在MDP上面:[http://rfc.zeromq.org/spec:8 the Majordomo Management Interface (MMI)].我们已经在代理中实现了它,不过除非你读完了整篇文章,否则你可能会错过.我将解释它在borkers中是如何工作的.

* 当客户请求一个以{{mmi.}}开头的服务时,我们不是把它转给一个worker,而是在内部处理它.

* 我们在这个代理中只处理一个服务,即{{mmi.service}},服务发现服务.

* 请求的有效载荷是一个外部服务的名称(一个真正的服务,由worker提供).

* 代理商返回"200"(OK)或"404"(未找到),这取决于是否有worker为该服务注册.

下面是我们如何在一个应用程序中使用服务发现.

```c
//  MMI echo query example

//  Lets us build this source without creating a library
#include"mdcliapi.c"

int main (int argc, char *argv [])
{
    int verbose = (argc > 1 && streq (argv [1],"-v"));
    mdcli_t *session = mdcli_new ("tcp://localhost:5555", verbose);

    //  This is the service we want to look up
    zmsg_t *request = zmsg_new ();
    zmsg_addstr (request,"echo");

    //  This is the service we send our request to
    zmsg_t *reply = mdcli_send (session,"mmi.service", &request);

    if (reply) {
        char *reply_code = zframe_strdup (zmsg_first (reply));
        printf ("Lookup echo service: %s\n", reply_code);
        free (reply_code);
        zmsg_destroy (&reply);
    }
    else
        printf ("E: no response from broker, make sure it's running\n");

    mdcli_destroy (&session);
    return 0;
}
```

在有或没有worker运行的情况下尝试这样做,你应该看到小程序相应地报告"200"或"404".在我们的示例代理中,MMI的实现是不牢固的.例如,如果一个worker消失了,服务仍然"存在".在实践中,borkers应该在某个可配置的超时后删除没有worker的服务.

## Idempotent服务

Idempotency不是你吃药的东西.它的意思是,重复一个操作是安全的.检查时钟是同位素的.把自己的信用卡借给自己的孩子则不是.虽然许多客户对服务器的用例是同位素的,但有些却不是.空闲用例的例子包括.

* 无状态任务分配,即一个管道,其中服务器是无状态工人,纯粹根据请求提供的状态来计算回复.在这种情况下,多次执行同一个请求是安全的(尽管效率不高).

* 一个名称服务,将逻辑地址翻译成端点来绑定或连接.在这种情况下,多次执行相同的查询请求是安全的.

下面是一些非幂等性用例的例子.

* 一个日志服务.人们不希望相同的日志信息被多次记录.

* 任何对下游节点有影响的服务,例如,向其他节点发送信息.如果该服务多次得到相同的请求,下游节点将得到重复的信息.

* 任何以某种非empotent方式修改共享数据的服务；例如,一个从银行账户中扣款的服务在没有额外工作的情况下不是empotent的.

当我们的服务器应用程序不是空闲的时候,我们必须更仔细地考虑它们到底什么时候会崩溃.如果一个应用程序在空闲时或在处理一个请求时死亡,这通常是好的.我们可以使用数据库事务来确保借记和贷记总是一起完成,如果有的话.如果服务器在发送回复时死亡,那是一个问题,因为就它而言,它已经完成了它的工作.

如果网络在回复被送回客户端的过程中死亡,同样的问题也会出现.客户端会认为服务器死了,并会重新发送请求,而服务器会做两次同样的工作,这不是我们想要的.

为了处理非空闲操作,使用相当标准的解决方案来检测和拒绝重复的请求.这意味着.

* 客户端必须在每个请求上盖上一个唯一的客户标识符和一个唯一的消息编号.

* 服务器,在发回回复之前,使用客户端ID和消息号码的组合作为密钥来存储它.

* 服务器,当从一个给定的客户端得到一个请求时,首先检查它是否有一个针对该客户端ID和消息号码的回复.如果有,它就不处理这个请求,而只是重新发送回复.

## 断开的可靠性(泰坦尼克模式)

一旦你意识到Majordomo是一个"可靠"的消息代理,你可能会想加入一些旋转的铁锈(也就是基于铁的硬盘盘片).毕竟,这对所有的企业消息传递系统都是有效的.这是一个如此诱人的想法,以至于不得不对它持否定态度,这让人有点难过.但残酷的愤世嫉俗是我的专长之一.所以,你不希望基于锈蚀的borkers坐在你的架构中心的一些原因是.

* 正如你所看到的,Lazy Pirate客户端的表现出奇的好.它适用于各种架构,从直接的客户端到服务器到分布式队列broker.它确实倾向于假设worker是无状态的和空闲的.但我们可以绕过这个限制,而不必求助于Rust.

* Rust带来了一系列的问题,从缓慢的性能到额外的部件,你必须管理,修复和处理早上6点的恐慌,因为它们在日常操作开始时不可避免地会损坏.总的来说,海盗模式的魅力在于其简单性.他们不会崩溃.如果你仍然担心硬件问题,你可以转移到一个完全没有代理的点对点模式.我将在本章后面解释.

不过,说到这里,基于锈的可靠性有一个合理的用例,那就是异步断开的网络.它解决了Pirate的一个主要问题,即客户端必须实时地等待一个答案.如果客户端和worker只是零星的连接(想想电子邮件的比喻),我们就不能在客户端和worker之间使用无状态网络.我们必须把状态放在中间.

因此,这里是泰坦尼克模式[图],我们将消息写入磁盘,以确保它们永远不会丢失,无论客户端和worker是如何零星连接的.正如我们对服务发现所做的那样,我们将把Titanic分层在MDP之上,而不是扩展它.这是很好的懒惰,因为这意味着我们可以在一个专门的工作器中实现我们的fire-and-forget可靠性,而不是在代理中.这很好,有几个原因.

* 它*容易得多,因为我们分而治之:borkers处理消息路由,worker处理可靠性.
* 它允许我们将用一种语言编写的borkers和用另一种语言编写的worker混合起来.
* 它让我们可以独立地发展"防火"技术.

唯一的缺点是,在borkers和硬盘之间有一个额外的网络跳跃.这些好处是很容易值得的.

有很多方法可以使一个持久的请求-回复架构.我们的目标是一个简单而无痛的架构.在玩了几个小时之后,我能想到的最简单的设计是一个"代理服务".也就是说,泰坦尼克号根本不影响工人.如果一个客户想要立即得到回复,它就直接与服务对话,并希望该服务是可用的.如果客户乐意等待一段时间,它就与泰坦尼克对话,并问:"嘿,伙计,你能在我去买菜的时候帮我处理一下吗？

```[[type="textdiagram" title="The Titanic Pattern"]]
#-----------#   #-----------#   #-----------#
|           |   |           |   |           |
|  Client   |   |  Client   |   |  Client   |
|           |   |           |   |           |
'-----------'   '-----------'   '-----------'
      ^               ^               ^
      |               |               |
      '---------------+---------------'
"Titanic,             |        "Titanic,
 give me coffee"      |          give me tea"
                      v                                
                .-----------.     #---------#     .-------.
                |           |     |         |     |       |
                |  Broker   |<--->| Titanic |<--->| Disk  |
                |           |     |         |     |       |
                '-----------'     #---------#     '-------'
                      ^
                      |
      .---------------+---------------.
      |               |               |
      v               v               v
.-----------.   .-----------.   .-----------.
| "Water"  |   |  "Tea"   |   |"Coffee"  |
+-----------+   +-----------+   +-----------+
|  Worker   |   |  Worker   |   |  Worker   |
#-----------#   #-----------#   #-----------#
```

因此,泰坦尼克既是工人又是客户.客户和泰坦尼克之间的对话是这样的.

* 客户.请为我接受这个请求.Titanic.好的,完成.
* 客户.你有给我的答复吗？泰坦尼克号:是的,就在这里.或者,没有,还没有.
* 客户.好的,你现在可以擦掉这个请求了,我很高兴.泰坦尼克号.好的,完成.

而Titanic和broker以及worker之间的对话则是这样的.

* Titanic:嘿,Broker,有咖啡服务吗？borkers.呃,是的,看起来像.
* Titanic: 嘿,咖啡服务,请为我处理这个.
* 咖啡.当然,给你.
* 泰坦尼克号:太棒了!

你可以通过这个和可能的失败情况来解决.如果一个worker在处理一个请求时崩溃了,Titanic会无限期地重试.如果一个回复在某个地方丢失了,Titanic会重试.如果请求被处理了,但客户端没有得到回复,它将再次询问.如果Titanic在处理请求或回复时崩溃了,客户端将再次尝试.只要请求被完全提交给安全存储,工作就不会丢失.

握手是迂腐的,但可以进行流水线处理,也就是说,客户端可以使用异步的Majordomo模式来做大量的工作,然后稍后获得响应.

我们需要一些方法来让客户端请求*它*的回复.我们会有很多客户要求相同的服务,客户消失后又以不同的身份重新出现.这里有一个简单的,相当安全的解决方案.

* 每个请求都会产生一个普遍唯一的ID(UUID),Titanic在排完队后将其返回给客户端.
* 当客户端要求回复时,它必须指定原始请求的UUID.

在现实的情况下,客户端希望安全地存储其请求的UUID,例如,在本地数据库中.

在我们跳下去写另一个正式的规范之前(有趣,有趣！),让我们考虑一下客户端如何与Titanic对话.一种方法是使用一个单一的服务,向它发送三种不同的请求类型.另一种方法,似乎更简单,是使用三个服务.

* {{titanic.request}}:存储一个请求信息,并返回一个请求的UUID.
* {{titanic.reply}}:获取一个给定请求UUID的回复(如果有的话).
* {{titanic.close}}:确认回复已被存储和处理.

我们将只是做一个多线程的worker,正如我们从ZeroMQ的多线程经验中看到的那样,这是很微妙的.然而,让我们首先勾勒一下Titanic在ZeroMQ消息和框架方面的样子.这给了我们[http://rfc.zeromq.org/spec:9 Titanic服务协议(TSP)].

对于客户端应用程序来说,使用TSP显然比直接通过MDP访问服务更费事.这里有一个最短的健壮的"回声"客户端例子.

```c
//  Titanic client example
//  Implements client side of http://rfc.zeromq.org/spec:9

//  Lets build this source without creating a library
#include"mdcliapi.c"

//  Calls a TSP service
//  Returns response if successful (status code 200 OK), else NULL
//
static zmsg_t *
s_service_call (mdcli_t *session, char *service, zmsg_t **request_p)
{
    zmsg_t *reply = mdcli_send (session, service, request_p);
    if (reply) {
        zframe_t *status = zmsg_pop (reply);
        if (zframe_streq (status,"200")) {
            zframe_destroy (&status);
            return reply;
        }
        else
        if (zframe_streq (status,"400")) {
            printf ("E: client fatal error, aborting\n");
            exit (EXIT_FAILURE);
        }
        else
        if (zframe_streq (status,"500")) {
            printf ("E: server fatal error, aborting\n");
            exit (EXIT_FAILURE);
        }
    }
    else
        exit (EXIT_SUCCESS);    //  Interrupted or failed

    zmsg_destroy (&reply);
    return NULL;        //  Didn't succeed; don't care why not
}

//  .split main task
//  The main task tests our service call by sending an echo request:

int main (int argc, char *argv [])
{
    int verbose = (argc > 1 && streq (argv [1],"-v"));
    mdcli_t *session = mdcli_new ("tcp://localhost:5555", verbose);

    //  1. Send 'echo' request to Titanic
    zmsg_t *request = zmsg_new ();
    zmsg_addstr (request,"echo");
    zmsg_addstr (request,"Hello world");
    zmsg_t *reply = s_service_call (
        session,"titanic.request", &request);

    zframe_t *uuid = NULL;
    if (reply) {
        uuid = zmsg_pop (reply);
        zmsg_destroy (&reply);
        zframe_print (uuid,"I: request UUID");
    }
    //  2. Wait until we get a reply
    while (!zctx_interrupted) {
        zclock_sleep (100);
        request = zmsg_new ();
        zmsg_add (request, zframe_dup (uuid));
        zmsg_t *reply = s_service_call (
            session,"titanic.reply", &request);

        if (reply) {
            char *reply_string = zframe_strdup (zmsg_last (reply));
            printf ("Reply: %s\n", reply_string);
            free (reply_string);
            zmsg_destroy (&reply);

            //  3. Close request
            request = zmsg_new ();
            zmsg_add (request, zframe_dup (uuid));
            reply = s_service_call (session,"titanic.close", &request);
            zmsg_destroy (&reply);
            break;
        }
        else {
            printf ("I: no reply yet, trying again...\n");
            zclock_sleep (5000);     //  Try again in 5 seconds
        }
    }
    zframe_destroy (&uuid);
    mdcli_destroy (&session);
    return 0;
}
```

当然,这可以而且应该被包裹在某种框架或API中.要求普通的应用开发者学习消息传递的全部细节是不健康的:这伤害了他们的大脑,花费了时间,并且提供了太多的方法来制造错误的复杂性.此外,它还会让人很难增加智能.

例如,这个客户端在每次请求时都会阻塞,而在一个真正的应用中,我们希望在任务执行的同时做一些有用的工作.这需要一些不简单的管道来建立一个后台线程,并与之干净地对话.这就是你想用一个简单的API来包装的事情,一般的开发者是不会滥用的.这与我们用于Majordomo的方法相同.

下面是Titanic的实现.这个服务器使用三个线程来处理三个服务,正如所建议的那样.它使用最粗暴的方法实现了对磁盘的完全持久化:每个消息一个文件.它是如此简单,让人害怕.唯一复杂的部分是,它为所有的请求保留了一个单独的队列,以避免重复读取目录.

```c
//  Titanic service
//  Implements server side of http://rfc.zeromq.org/spec:9

//  Lets us build this source without creating a library
#include"mdwrkapi.c"
#include"mdcliapi.c"

#include"zfile.h"
#include <uuid/uuid.h>

//  Return a new UUID as a printable character string
//  Caller must free returned string when finished with it

static char *
s_generate_uuid (void)
{
    char hex_char [] ="0123456789ABCDEF";
    char *uuidstr = zmalloc (sizeof (uuid_t) * 2 + 1);
    uuid_t uuid;
    uuid_generate (uuid);
    int byte_nbr;
    for (byte_nbr = 0; byte_nbr < sizeof (uuid_t); byte_nbr++) {
        uuidstr [byte_nbr * 2 + 0] = hex_char [uuid [byte_nbr] >> 4];
        uuidstr [byte_nbr * 2 + 1] = hex_char [uuid [byte_nbr] & 15];
    }
    return uuidstr;
}

//  Returns freshly allocated request filename for given UUID

#define TITANIC_DIR".titanic"

static char *
s_request_filename (char *uuid) {
    char *filename = malloc (256);
    snprintf (filename, 256, TITANIC_DIR"/%s.req", uuid);
    return filename;
}

//  Returns freshly allocated reply filename for given UUID

static char *
s_reply_filename (char *uuid) {
    char *filename = malloc (256);
    snprintf (filename, 256, TITANIC_DIR"/%s.rep", uuid);
    return filename;
}

//  .split Titanic request service
//  The {{titanic.request}} task waits for requests to this service. It writes
//  each request to disk and returns a UUID to the client. The client picks
//  up the reply asynchronously using the {{titanic.reply}} service:

static void
titanic_request (void *args, zctx_t *ctx, void *pipe)
{
    mdwrk_t *worker = mdwrk_new (
       "tcp://localhost:5555","titanic.request", 0);
    zmsg_t *reply = NULL;

    while (true) {
        //  Send reply if it's not null
        //  And then get next request from broker
        zmsg_t *request = mdwrk_recv (worker, &reply);
        if (!request)
            break;      //  Interrupted, exit

        //  Ensure message directory exists
        zfile_mkdir (TITANIC_DIR);

        //  Generate UUID and save message to disk
        char *uuid = s_generate_uuid ();
        char *filename = s_request_filename (uuid);
        FILE *file = fopen (filename,"w");
        assert (file);
        zmsg_save (request, file);
        fclose (file);
        free (filename);
        zmsg_destroy (&request);

        //  Send UUID through to message queue
        reply = zmsg_new ();
        zmsg_addstr (reply, uuid);
        zmsg_send (&reply, pipe);

        //  Now send UUID back to client
        //  Done by the mdwrk_recv() at the top of the loop
        reply = zmsg_new ();
        zmsg_addstr (reply,"200");
        zmsg_addstr (reply, uuid);
        free (uuid);
    }
    mdwrk_destroy (&worker);
}

//  .split Titanic reply service
//  The {{titanic.reply}} task checks if there's a reply for the specified
//  request (by UUID), and returns a 200 (OK), 300 (Pending), or 400
//  (Unknown) accordingly:

static void *
titanic_reply (void *context)
{
    mdwrk_t *worker = mdwrk_new (
       "tcp://localhost:5555","titanic.reply", 0);
    zmsg_t *reply = NULL;

    while (true) {
        zmsg_t *request = mdwrk_recv (worker, &reply);
        if (!request)
            break;      //  Interrupted, exit

        char *uuid = zmsg_popstr (request);
        char *req_filename = s_request_filename (uuid);
        char *rep_filename = s_reply_filename (uuid);
        if (zfile_exists (rep_filename)) {
            FILE *file = fopen (rep_filename,"r");
            assert (file);
            reply = zmsg_load (NULL, file);
            zmsg_pushstr (reply,"200");
            fclose (file);
        }
        else {
            reply = zmsg_new ();
            if (zfile_exists (req_filename))
                zmsg_pushstr (reply,"300"); //Pending
            else
                zmsg_pushstr (reply,"400"); //Unknown
        }
        zmsg_destroy (&request);
        free (uuid);
        free (req_filename);
        free (rep_filename);
    }
    mdwrk_destroy (&worker);
    return 0;
}

//  .split Titanic close task
//  The {{titanic.close}} task removes any waiting replies for the request
//  (specified by UUID). It's idempotent, so it is safe to call more than
//  once in a row:

static void *
titanic_close (void *context)
{
    mdwrk_t *worker = mdwrk_new (
       "tcp://localhost:5555","titanic.close", 0);
    zmsg_t *reply = NULL;

    while (true) {
        zmsg_t *request = mdwrk_recv (worker, &reply);
        if (!request)
            break;      //  Interrupted, exit

        char *uuid = zmsg_popstr (request);
        char *req_filename = s_request_filename (uuid);
        char *rep_filename = s_reply_filename (uuid);
        zfile_delete (req_filename);
        zfile_delete (rep_filename);
        free (uuid);
        free (req_filename);
        free (rep_filename);

        zmsg_destroy (&request);
        reply = zmsg_new ();
        zmsg_addstr (reply,"200");
    }
    mdwrk_destroy (&worker);
    return 0;
}

//  .split worker task
//  This is the main thread for the Titanic worker. It starts three child
//  threads; for the request, reply, and close services. It then dispatches
//  requests to workers using a simple brute force disk queue. It receives
//  request UUIDs from the {{titanic.request}} service, saves these to a disk
//  file, and then throws each request at MDP workers until it gets a
//  response.

static int s_service_success (char *uuid);

int main (int argc, char *argv [])
{
    int verbose = (argc > 1 && streq (argv [1],"-v"));
    zctx_t *ctx = zctx_new ();

    void *request_pipe = zthread_fork (ctx, titanic_request, NULL);
    zthread_new (titanic_reply, NULL);
    zthread_new (titanic_close, NULL);

    //  Main dispatcher loop
    while (true) {
        //  We'll dispatch once per second, if there's no activity
        zmq_pollitem_t items [] = { { request_pipe, 0, ZMQ_POLLIN, 0 } };
        int rc = zmq_poll (items, 1, 1000 * ZMQ_POLL_MSEC);
        if (rc == -1)
            break;              //  Interrupted
        if (items [0].revents & ZMQ_POLLIN) {
            //  Ensure message directory exists
            zfile_mkdir (TITANIC_DIR);

            //  Append UUID to queue, prefixed with '-' for pending
            zmsg_t *msg = zmsg_recv (request_pipe);
            if (!msg)
                break;          //  Interrupted
            FILE *file = fopen (TITANIC_DIR"/queue","a");
            char *uuid = zmsg_popstr (msg);
            fprintf (file,"-%s\n", uuid);
            fclose (file);
            free (uuid);
            zmsg_destroy (&msg);
        }
        //  Brute force dispatcher
        char entry [] ="?.......:.......:.......:.......:";
        FILE *file = fopen (TITANIC_DIR"/queue","r+");
        while (file && fread (entry, 33, 1, file) == 1) {
            //  UUID is prefixed with '-' if still waiting
            if (entry [0] == '-') {
                if (verbose)
                    printf ("I: processing request %s\n", entry + 1);
                if (s_service_success (entry + 1)) {
                    //  Mark queue entry as processed
                    fseek (file, -33, SEEK_CUR);
                    fwrite ("+", 1, 1, file);
                    fseek (file, 32, SEEK_CUR);
                }
            }
            //  Skip end of line, LF or CRLF
            if (fgetc (file) == '\r')
                fgetc (file);
            if (zctx_interrupted)
                break;
        }
        if (file)
            fclose (file);
    }
    return 0;
}

//  .split try to call a service
//  Here, we first check if the requested MDP service is defined or not,
//  using a MMI lookup to the Majordomo broker. If the service exists,
//  we send a request and wait for a reply using the conventional MDP
//  client API. This is not meant to be fast, just very simple:

static int
s_service_success (char *uuid)
{
    //  Load request message, service will be first frame
    char *filename = s_request_filename (uuid);
    FILE *file = fopen (filename,"r");
    free (filename);

    //  If the client already closed request, treat as successful
    if (!file)
        return 1;

    zmsg_t *request = zmsg_load (NULL, file);
    fclose (file);
    zframe_t *service = zmsg_pop (request);
    char *service_name = zframe_strdup (service);

    //  Create MDP client session with short timeout
    mdcli_t *client = mdcli_new ("tcp://localhost:5555", false);
    mdcli_set_timeout (client, 1000);  //  1 sec
    mdcli_set_retries (client, 1);     //  only 1 retry

    //  Use MMI protocol to check if service is available
    zmsg_t *mmi_request = zmsg_new ();
    zmsg_add (mmi_request, service);
    zmsg_t *mmi_reply = mdcli_send (client,"mmi.service", &mmi_request);
    int service_ok = (mmi_reply
        && zframe_streq (zmsg_first (mmi_reply),"200"));
    zmsg_destroy (&mmi_reply);

    int result = 0;
    if (service_ok) {
        zmsg_t *reply = mdcli_send (client, service_name, &request);
        if (reply) {
            filename = s_reply_filename (uuid);
            FILE *file = fopen (filename,"w");
            assert (file);
            zmsg_save (reply, file);
            fclose (file);
            free (filename);
            result = 1;
        }
        zmsg_destroy (&reply);
    }
    else
        zmsg_destroy (&request);

    mdcli_destroy (&client);
    free (service_name);
    return result;
}
```

为了测试这一点,启动{{mdbroker}}和{{titanic}},然后运行{{ticlient}}.现在任意启动{{mdworker}},你应该看到客户端得到了响应,并愉快地退出.

关于这段代码的一些注意事项.

* 请注意,有些循环是从发送消息开始的,有些则是从接收消息开始的.这是因为Titanic在不同的角色中既是客户端又是工作器.
* Titanic代理使用MMI服务发现协议,只向那些看起来正在运行的服务发送请求.由于我们的小Majordomo代理中的MMI实现相当差,这不会一直工作下去.
* 我们使用一个inproc连接,将新的请求数据从{{titanic.request}}服务发送到主调度器.这使调度器不必扫描磁盘目录,加载所有请求文件,并按日期/时间排序.

这个例子的重要之处不是性能(虽然我没有测试过,但肯定很糟糕),而是它对可靠性契约的实现程度.要试一下,先启动mdbroker和titanic程序.然后启动ticlient,再启动mdworker回声服务.你可以使用{{-v}}选项来运行所有这四个程序,以进行粗略的活动追踪.你可以停止和重启任何一块*,除了客户端*,没有什么会丢失.

如果你想在实际案例中使用Titanic,你会迅速问:"我们怎样才能让它更快？"

以下是我要做的,从例子的实现开始.

* 对所有数据使用一个磁盘文件,而不是多个文件.操作系统在处理几个大文件时通常比处理许多小文件要好.
* 将该磁盘文件组织成一个循环缓冲区,以便新的请求可以连续写入(偶尔会有包裹).一个线程,全速写入一个磁盘文件,可以快速工作.
* 在内存中保留索引,在启动时从磁盘缓冲区重建索引.这就节省了为保持索引在磁盘上完全安全而需要的额外的磁盘头跳动.你会希望在每条信息之后进行fsync,或者每N毫秒进行一次,如果你准备在系统故障的情况下丢失最后的M条信息.
* 使用固态硬盘而不是旋转的氧化铁盘.
* 预先分配整个文件,或以大块方式分配,这允许循环缓冲区根据需要增长和缩小.这就避免了碎片化,并确保大多数读和写是连续的.

以此类推.我不建议将信息存储在数据库中,甚至是"快速"的键/值存储,除非你真的喜欢一个特定的数据库,并且没有性能方面的担忧.你将为这种抽象性付出高昂的代价,比原始磁盘文件高出十倍到一千倍.

如果你想让泰坦尼克号*更加可靠,就把请求复制到第二个服务器上,你可以把它放在第二个地方,这个地方离你的主服务器足够远,可以在核攻击中幸存下来,但又不会远到让你得到太多的延迟.

如果你想让泰坦尼克号*更快,更不可靠*,就把请求和回复完全存储在内存中.这将为你提供一个断开网络的功能,但请求不会在泰坦尼克服务器本身崩溃后存活.

## 高可用性对(二进制星形模式)

```[[type="textdiagram" title="High-Availability Pair, Normal Operation"]]
#------------#           #------------#
|            |           |            |
|  Primary   |<--------->|   Backup   |
| "active"   |           | "passive"  |
|            |           |            |
#------------#           #------------#
      ^
      |
      |
      |
#-----+------#
|            |
|   Client   |
|            |
#------------#
```

二元星模式将两台服务器置于主备高可用性对中[图].在任何时候,其中一个(主动)接受来自客户端应用程序的连接.另一个(被动)不做任何事情,但这两个服务器互相监视.如果主动者从网络中消失,在一定时间后,被动者将接管主动者的位置.

我们在iMatix为我们的[http://www.openamq.org OpenAMQ服务器]开发了双星模式.我们设计了它.

* 提供一个简单明了的高可用性解决方案.
* 足够简单,可以实际理解和使用.
* 在需要的时候可靠地进行故障转移,而且只在需要的时候.

假设我们有一个二进制星对在运行,以下是会导致故障转移的不同情况[图].

* 运行主服务器的硬件出现了致命的问题(电源爆炸,机器着火,或者有人误拔插头),并消失了.应用程序看到这一点,并重新连接到备份服务器.
* 主服务器所在的网段崩溃了--也许是路由器被电源尖峰击中--应用程序开始重新连接到备份服务器.
* 主服务器崩溃或被操作员杀死,并且没有自动重新启动.

```[[type="textdiagram" title="High-availability Pair During Failover"]]
#------------#           #------------#
|            |           |            |
|  Primary   |<--------->|   Backup   |
| "passive"  |           |  "active"  |
|            |           |            |
#------------#           #------------#
                                ^
                                |
      .-------------------------'
      |
#-----+------#
|            |
|   Client   |
|            |
#------------#
```

从故障转移中的恢复工作如下.

* 操作员重新启动主服务器并修复导致它从网络上消失的任何问题.
* 操作员在对应用程序造成最小干扰的时候停止备份服务器.
* 当应用程序重新连接到主服务器时,操作员重新启动备份服务器.

恢复(使用主服务器作为活动的)是一个手动操作.痛苦的经验告诉我们,自动恢复是不可取的.有几个原因.

* 故障转移会给应用程序造成服务中断,可能持续10-30秒.如果真的有紧急情况,这比完全中断要好得多.但是,如果恢复又会造成10-30秒的中断,最好是在非高峰期,当用户离开网络时发生.

* 当出现紧急情况时,绝对的首要任务是为那些试图修复事情的人提供确定性.自动恢复为系统管理员创造了不确定性,他们不能再确定哪台服务器是负责的,而要反复检查.

* 自动恢复会造成网络故障后又恢复的情况,使操作人员处于分析发生了什么的困难境地.有一个服务中断,但原因并不清楚.

说到这里,如果这个正在运行(再次),而备份服务器出现故障,二进制星形模式将故障返回到主服务器.事实上,这就是我们挑起恢复的方式.

二进制星形对的关闭过程是,要么.

* 停止被动服务器,然后在任何时间停止主动服务器,或者
* 以任何顺序停止两个服务器,但要在几秒钟内停止.

先停止主动服务器,再停止被动服务器,如果延迟时间超过故障转移超时,将导致应用程序断开连接,然后重新连接,然后再次断开连接,这可能会干扰用户.

### 详细要求

二进制之星是尽可能的简单,同时还能准确的工作.事实上,目前的设计是第三次完全重新设计.以前的每一个设计我们都发现太复杂了,想做的事情太多,我们剥离了一些功能,直到我们得出一个可以理解的,容易使用的,足够可靠的,值得使用的设计.

这些是我们对高可用性架构的要求.

* 故障转移的目的是为灾难性的系统故障提供保险,如硬件故障,火灾,事故等.有一些更简单的方法可以从普通的服务器崩溃中恢复,我们已经介绍了这些.

* 故障转移时间应在60秒以下,最好在10秒以下.

* 故障转移必须自动发生,而恢复必须手动发生.我们希望应用程序能够自动切换到备份服务器上,但我们不希望它们切换回主服务器,除非操作人员已经修复了任何问题,并决定这是一个再次中断应用程序的好时机.

* 客户端应用程序的语义应该是简单的,便于开发者理解.理想情况下,它们应该隐藏在客户端的API中.

* 应该为网络架构师提供明确的指示,说明如何避免可能导致*分裂大脑综合症*的设计,在这种情况下,双星对中的两个服务器都认为自己是活动服务器.

* 不应该对两个服务器的启动顺序有任何依赖性.

* 必须能够有计划地停止和重新启动任何一台服务器,而不停止客户端应用程序(尽管它们可能被迫重新连接).

* 操作员必须能够在任何时候监控两个服务器.

* 必须能够使用高速专用网络连接两个服务器.也就是说,故障转移同步必须能够使用一个特定的IP路由.

我们做了以下假设.

* 单一的备份服务器提供了足够的保险；我们不需要多层次的备份.

* 主服务器和备份服务器有同样的能力来承载应用负载.我们不试图在服务器之间平衡负载.

* 有足够的预算来支付一个完全冗余的备份服务器,几乎所有的时间都不做任何事情.

我们不试图涵盖以下内容.

* 使用一个活跃的备份服务器或负载平衡.在二进制星对中,备份服务器是不活跃的,在主服务器离线之前不做任何有用的工作.

* 以任何方式处理持久性信息或交易.我们假设存在一个不可靠的(可能是不信任的)服务器或双星对的网络.

*对网络的任何自动探索.二进制星对是在网络中手动明确定义的,并且是应用程序所知道的(至少在其配置数据中).

* 服务器之间的状态或消息的复制.所有服务器端的状态必须由应用程序在故障转移时重新创建.

下面是我们在二进制之星中使用的关键术语.

* *主要*:通常或最初处于活动状态的服务器.

* *备份*:通常是被动的服务器.如果主服务器从网络上消失,以及客户应用要求备份服务器连接时,它将成为活动的.

* *活动*:接受客户端连接的服务器.最多只有一个主动服务器.

* *被动*:如果主动服务器消失,则由该服务器接管.请注意,当一个二进制星对正常运行时,主服务器是主动的,而备份是被动的.当发生故障切换时,角色会被转换.

要配置一个二星对,你需要.

* 告诉主服务器备份服务器的位置.
* 告诉备份服务器主服务器的位置.
* 可以选择调整故障转移的响应时间,两个服务器必须是相同的.

调整的主要问题是你希望服务器检查对等状态的频率,以及你希望激活故障转移的速度.在我们的例子中,故障转移超时值默认为2,000毫秒.如果你降低这个值,备份服务器将更快地接管为活动状态,但可能在主服务器可以恢复的情况下接管.例如,你可能已经把主服务器包裹在一个shell脚本中,如果它崩溃了就重新启动.在这种情况下,超时应该高于重启主服务器所需的时间.

为了使客户端应用程序能够与二进制星对正常工作,它们必须.

* 知道两个服务器的地址.
* 尝试连接到主服务器,如果失败,则连接到备份服务器.
* 检测一个失败的连接,通常使用心跳声.
* 尝试重新连接到主服务器,然后是备份服务器(按顺序),重试之间的延迟至少要和服务器故障转移的超时时间一样高.
* 在服务器上重新创建他们需要的所有状态.
* 如果消息需要可靠的话,重新传输在故障切换期间丢失的消息.

这不是一个简单的工作,我们通常会把它包装在一个API中,对真正的终端用户应用进行隐藏.

这些是二进制星形模式的主要限制.

* 一个服务器进程不能成为一个以上的二进制星对的一部分.
* 一个主服务器只能有一个备份服务器,而不能有更多.
* 被动服务器不做任何有用的工作,因此是浪费的.
* 备份服务器必须能够处理完整的应用负载.
* 故障转移配置不能在运行时修改.
* 客户端应用程序必须做一些工作才能从故障转移中受益.

### 防止大脑分裂综合症

*当集群的不同部分认为它们同时处于活动状态时,就会发生*分脑综合症.它导致应用程序不再相互看到对方.二进制之星有一个检测和消除分脑的算法,它是基于一个三方决策机制(一个服务器在得到应用连接请求之前不会决定成为活动的,而且它不能看到其对等服务器).

然而,仍然有可能(错误地)设计一个网络来欺骗这个算法.一个典型的场景是一个分布在两座大楼之间的双星对,其中每座大楼也有一组应用程序,两座大楼之间有一个单一的网络链接.破坏这个链接会产生两套客户应用,每套应用都有二元星对的一半,每个故障转移服务器都会变成活动的.

为了防止分裂大脑的情况,我们必须使用专用的网络链接来连接二元星对,这可以简单到把它们都插入同一个交换机,或者更好的是在两台机器之间直接使用交叉电缆.

我们不能把一个二进制星形结构分成两个岛,每个岛都有一套应用程序.虽然这可能是一种常见的网络架构类型,但在这种情况下,你应该使用联合,而不是高可用性故障转移.

一个适当的偏执的网络配置将使用两个私有集群互连,而不是一个单一的.此外,用于集群的网卡将与用于消息流量的网卡不同,甚至可能在服务器硬件的不同路径上.其目的是将网络中可能出现的故障与集群中可能出现的故障分开.网络端口可能有一个相对较高的故障率.

### 二进制星的实现

不多说了,这里是二进制之星服务器的概念验证实现.主服务器和备份服务器运行相同的代码,你在运行代码时选择它们的角色.


  <summary><mark><font color=darkred>Binary Star server proof-of-concept implementation</font></mark></summary>

```c
//  Binary Star server proof-of-concept implementation. This server does no
//  real work; it just demonstrates the Binary Star failover model.

#include"czmq.h"

//  States we can be in at any point in time
typedef enum {
    STATE_PRIMARY = 1,          //  Primary, waiting for peer to connect
    STATE_BACKUP = 2,           //  Backup, waiting for peer to connect
    STATE_ACTIVE = 3,           //  Active - accepting connections
    STATE_PASSIVE = 4           //  Passive - not accepting connections
} state_t;

//  Events, which start with the states our peer can be in
typedef enum {
    PEER_PRIMARY = 1,           //  HA peer is pending primary
    PEER_BACKUP = 2,            //  HA peer is pending backup
    PEER_ACTIVE = 3,            //  HA peer is active
    PEER_PASSIVE = 4,           //  HA peer is passive
    CLIENT_REQUEST = 5          //  Client makes request
} event_t;

//  Our finite state machine
typedef struct {
    state_t state;              //  Current state
    event_t event;              //  Current event
    int64_t peer_expiry;        //  When peer is considered 'dead'
} bstar_t;

//  We send state information this often
//  If peer doesn't respond in two heartbeats, it is 'dead'
#define HEARTBEAT 1000          //  In msecs

//  .split Binary Star state machine
//  The heart of the Binary Star design is its finite-state machine (FSM).
//  The FSM runs one event at a time. We apply an event to the current state,
//  which checks if the event is accepted, and if so, sets a new state:

static bool
s_state_machine (bstar_t *fsm)
{
    bool exception = false;
    
    //  These are the PRIMARY and BACKUP states; we're waiting to become
    //  ACTIVE or PASSIVE depending on events we get from our peer:
    if (fsm->state == STATE_PRIMARY) {
        if (fsm->event == PEER_BACKUP) {
            printf ("I: connected to backup (passive), ready active\n");
            fsm->state = STATE_ACTIVE;
        }
        else
        if (fsm->event == PEER_ACTIVE) {
            printf ("I: connected to backup (active), ready passive\n");
            fsm->state = STATE_PASSIVE;
        }
        //  Accept client connections
    }
    else
    if (fsm->state == STATE_BACKUP) {
        if (fsm->event == PEER_ACTIVE) {
            printf ("I: connected to primary (active), ready passive\n");
            fsm->state = STATE_PASSIVE;
        }
        else
        //  Reject client connections when acting as backup
        if (fsm->event == CLIENT_REQUEST)
            exception = true;
    }
    else
    //  .split active and passive states
    //  These are the ACTIVE and PASSIVE states:

    if (fsm->state == STATE_ACTIVE) {
        if (fsm->event == PEER_ACTIVE) {
            //  Two actives would mean split-brain
            printf ("E: fatal error - dual actives, aborting\n");
            exception = true;
        }
    }
    else
    //  Server is passive
    //  CLIENT_REQUEST events can trigger failover if peer looks dead
    if (fsm->state == STATE_PASSIVE) {
        if (fsm->event == PEER_PRIMARY) {
            //  Peer is restarting - become active, peer will go passive
            printf ("I: primary (passive) is restarting, ready active\n");
            fsm->state = STATE_ACTIVE;
        }
        else
        if (fsm->event == PEER_BACKUP) {
            //  Peer is restarting - become active, peer will go passive
            printf ("I: backup (passive) is restarting, ready active\n");
            fsm->state = STATE_ACTIVE;
        }
        else
        if (fsm->event == PEER_PASSIVE) {
            //  Two passives would mean cluster would be non-responsive
            printf ("E: fatal error - dual passives, aborting\n");
            exception = true;
        }
        else
        if (fsm->event == CLIENT_REQUEST) {
            //  Peer becomes active if timeout has passed
            //  It's the client request that triggers the failover
            assert (fsm->peer_expiry > 0);
            if (zclock_time () >= fsm->peer_expiry) {
                //  If peer is dead, switch to the active state
                printf ("I: failover successful, ready active\n");
                fsm->state = STATE_ACTIVE;
            }
            else
                //  If peer is alive, reject connections
                exception = true;
        }
    }
    return exception;
}

//  .split main task
//  This is our main task. First we bind/connect our sockets with our
//  peer and make sure we will get state messages correctly. We use
//  three sockets; one to publish state, one to subscribe to state, and
//  one for client requests/replies:

int main (int argc, char *argv [])
{
    //  Arguments can be either of:
    //      -p  primary server, at tcp://localhost:5001
    //      -b  backup server, at tcp://localhost:5002
    zctx_t *ctx = zctx_new ();
    void *statepub = zsocket_new (ctx, ZMQ_PUB);
    void *statesub = zsocket_new (ctx, ZMQ_SUB);
    zsocket_set_subscribe (statesub,"");
    void *frontend = zsocket_new (ctx, ZMQ_ROUTER);
    bstar_t fsm = { 0 };

    if (argc == 2 && streq (argv [1],"-p")) {
        printf ("I: Primary active, waiting for backup (passive)\n");
        zsocket_bind (frontend,"tcp://*:5001");
        zsocket_bind (statepub,"tcp://*:5003");
        zsocket_connect (statesub,"tcp://localhost:5004");
        fsm.state = STATE_PRIMARY;
    }
    else
    if (argc == 2 && streq (argv [1],"-b")) {
        printf ("I: Backup passive, waiting for primary (active)\n");
        zsocket_bind (frontend,"tcp://*:5002");
        zsocket_bind (statepub,"tcp://*:5004");
        zsocket_connect (statesub,"tcp://localhost:5003");
        fsm.state = STATE_BACKUP;
    }
    else {
        printf ("Usage: bstarsrv { -p | -b }\n");
        zctx_destroy (&ctx);
        exit (0);
    }
    //  .split handling socket input
    //  We now process events on our two input sockets, and process these
    //  events one at a time via our finite-state machine. Our"work" for
    //  a client request is simply to echo it back:

    //  Set timer for next outgoing state message
    int64_t send_state_at = zclock_time () + HEARTBEAT;
    while (!zctx_interrupted) {
        zmq_pollitem_t items [] = {
            { frontend, 0, ZMQ_POLLIN, 0 },
            { statesub, 0, ZMQ_POLLIN, 0 }
        };
        int time_left = (int) ((send_state_at - zclock_time ()));
        if (time_left < 0)
            time_left = 0;
        int rc = zmq_poll (items, 2, time_left * ZMQ_POLL_MSEC);
        if (rc == -1)
            break;              //  Context has been shut down

        if (items [0].revents & ZMQ_POLLIN) {
            //  Have a client request
            zmsg_t *msg = zmsg_recv (frontend);
            fsm.event = CLIENT_REQUEST;
            if (s_state_machine (&fsm) == false)
                //  Answer client by echoing request back
                zmsg_send (&msg, frontend);
            else
                zmsg_destroy (&msg);
        }
        if (items [1].revents & ZMQ_POLLIN) {
            //  Have state from our peer, execute as event
            char *message = zstr_recv (statesub);
            fsm.event = atoi (message);
            free (message);
            if (s_state_machine (&fsm))
                break;          //  Error, so exit
            fsm.peer_expiry = zclock_time () + 2 * HEARTBEAT;
        }
        //  If we timed out, send state to peer
        if (zclock_time () >= send_state_at) {
            char message [2];
            sprintf (message,"%d", fsm.state);
            zstr_send (statepub, message);
            send_state_at = zclock_time () + HEARTBEAT;
        }
    }
    if (zctx_interrupted)
        printf ("W: interrupted\n");

    //  Shutdown sockets and context
    zctx_destroy (&ctx);
    return 0;
}
```
</details>

这是客户:


  <summary><mark><font color=darkred>Binary Star client proof-of-concept implementation</font></mark></summary>

```c
//  Binary Star client proof-of-concept implementation. This client does no
//  real work; it just demonstrates the Binary Star failover model.

#include"czmq.h"
#define REQUEST_TIMEOUT     1000    //  msecs
#define SETTLE_DELAY        2000    //  Before failing over

int main (void)
{
    zctx_t *ctx = zctx_new ();

    char *server [] = {"tcp://localhost:5001","tcp://localhost:5002" };
    uint server_nbr = 0;

    printf ("I: connecting to server at %s...\n", server [server_nbr]);
    void *client = zsocket_new (ctx, ZMQ_REQ);
    zsocket_connect (client, server [server_nbr]);

    int sequence = 0;
    while (!zctx_interrupted) {
        //  We send a request, then we work to get a reply
        char request [10];
        sprintf (request,"%d", ++sequence);
        zstr_send (client, request);

        int expect_reply = 1;
        while (expect_reply) {
            //  Poll socket for a reply, with timeout
            zmq_pollitem_t items [] = { { client, 0, ZMQ_POLLIN, 0 } };
            int rc = zmq_poll (items, 1, REQUEST_TIMEOUT * ZMQ_POLL_MSEC);
            if (rc == -1)
                break;          //  Interrupted

            //  .split main body of client
            //  We use a Lazy Pirate strategy in the client. If there's no
            //  reply within our timeout, we close the socket and try again.
            //  In Binary Star, it's the client vote that decides which
            //  server is primary; the client must therefore try to connect
            //  to each server in turn:
            
            if (items [0].revents & ZMQ_POLLIN) {
                //  We got a reply from the server, must match sequence
                char *reply = zstr_recv (client);
                if (atoi (reply) == sequence) {
                    printf ("I: server replied OK (%s)\n", reply);
                    expect_reply = 0;
                    sleep (1);  //  One request per second
                }
                else
                    printf ("E: bad reply from server: %s\n", reply);
                free (reply);
            }
            else {
                printf ("W: no response from server, failing over\n");
                
                //  Old socket is confused; close it and open a new one
                zsocket_destroy (ctx, client);
                server_nbr = (server_nbr + 1) % 2;
                zclock_sleep (SETTLE_DELAY);
                printf ("I: connecting to server at %s...\n",
                        server [server_nbr]);
                client = zsocket_new (ctx, ZMQ_REQ);
                zsocket_connect (client, server [server_nbr]);

                //  Send request again, on new socket
                zstr_send (client, request);
            }
        }
    }
    zctx_destroy (&ctx);
    return 0;
}
```
</details>

要测试二进制星,请按任何顺序启动服务器和客户端:

```
bstarsrv -p     # Start primary
bstarsrv -b     # Start backup
bstarcli
```

然后你可以通过杀死主服务器来引发故障转移,通过重启主服务器和杀死备份服务器来恢复.请注意,触发故障切换和恢复的是客户投票.

二进制星是由一个有限状态机驱动的[图].事件是对等体的状态,所以"对等体活跃"意味着另一个服务器已经告诉我们它在活动."客户端请求"意味着我们已经收到了一个客户端请求."客户端投票"意味着我们已经收到了一个客户端请求,并且我们的对等体在两个心跳中没有活动.

注意,服务器使用PUB-SUB套接字进行状态交换.其他的套接字组合在这里是不起作用的.如果没有对等体准备好接收消息,PUSH和DEALER就会阻塞.PAIR在对等体消失又回来的情况下不会重新连接.ROUTER需要对等体的地址,然后才能向其发送消息.


```[[type="textdiagram" title="Binary Star Finite State Machine"]]
    Start .-----------------------.   .----------.               Start
          |                       |   |          |                   
      |   |Client Request         |   |    Client| Request         |
      v   |                       v   v          |                 v
.---------+-.                 .-----------.      |           .-----------.
|           | Peer Backup     |           +------'           |           |
|  Primary  +---------------->|  Active   |<--------------.  |  Backup   |
|           |         .------>|           |<-----.        |  |           |
'-----+-----'         |       '-----+-----'      |        |  '-----+-----'
      |               |             |            |        |        |
  Peer| Active        |         Peer| Active     |        |    Peer| Active
      |               |             v            |        |        |
      |               |       .-----------.      |        |        |
      |               |       |           |      |        |        |
      |           Peer| Backup|  Error!   |  Peer| Primary|        |
      |               |       |           |      |        |        |
      |               |       '-----------'      |        |        |
      |               |             ^            |        |        |
      |               |         Peer| Passive    |  Client| Vote   |
      |               |             |            |        |        |
      |               |       .-----+-----.      |        |        |
      |               |       |           +------'        |        |
      |               '-------+  Passive  +---------------'        |
      '---------------------->|           |<-----------------------'
                              '-----------'
```

### 双星反应堆

Binary Star很有用,而且足够通用,可以打包成一个可重用的反应器类.反应器会在有消息需要处理时运行并调用我们的代码.这比把二进制之星的代码复制/粘贴到我们需要这种功能的每个服务器上要好得多.

在C语言中,我们封装了之前看到的CZMQ {{zloop}}类.{{zloop}}可以让你注册处理程序,对套接字和定时器事件作出反应.在二进制之星的反应器中,我们为投票者和状态变化(从主动到被动,反之亦然)提供处理程序.以下是{{bstar}}的内容 API.

```[[type="fragment" name="bstar"]]
* 创建一个新的二进制之星实例,使用本地(bind)和
* 远程(connect)端点来设置服务器对等.
bstar_t *bstar_new (int primary, char *local, char *remote).

* 销毁一个二进制星体实例
void bstar_destroy (bstar_t **self_p);

* 返回底层的zloop反应器,用于定时器和阅读器的
* 注册和取消.
zloop_t *bstar_zloop (bstar_t *self);

* 注册投票器
int bstar_voter (bstar_t *self, char *endpoint, int type,
                 zloop_fn handler, void *arg).

* 注册主状态变化处理程序
void bstar_new_active (bstar_t *self, zloop_fn handler, void *arg);
void bstar_new_passive (bstar_t *self, zloop_fn handler, void *arg);

* 启动反应器,如果一个回调函数返回-1,反应器就会结束.
* 或进程收到SIGINT或SIGTERM时结束.
int bstar_start (bstar_t *self);
```

而这里是类的实现.

  <summary><mark><font color=darkred>bstar class - Binary Star reactor</font></mark></summary>

```c
//  bstar class - Binary Star reactor

#include"bstar.h"

//  States we can be in at any point in time
typedef enum {
    STATE_PRIMARY = 1,          //  Primary, waiting for peer to connect
    STATE_BACKUP = 2,           //  Backup, waiting for peer to connect
    STATE_ACTIVE = 3,           //  Active - accepting connections
    STATE_PASSIVE = 4           //  Passive - not accepting connections
} state_t;

//  Events, which start with the states our peer can be in
typedef enum {
    PEER_PRIMARY = 1,           //  HA peer is pending primary
    PEER_BACKUP = 2,            //  HA peer is pending backup
    PEER_ACTIVE = 3,            //  HA peer is active
    PEER_PASSIVE = 4,           //  HA peer is passive
    CLIENT_REQUEST = 5          //  Client makes request
} event_t;

//  Structure of our class

struct _bstar_t {
    zctx_t *ctx;                //  Our private context
    zloop_t *loop;              //  Reactor loop
    void *statepub;             //  State publisher
    void *statesub;             //  State subscriber
    state_t state;              //  Current state
    event_t event;              //  Current event
    int64_t peer_expiry;        //  When peer is considered 'dead'
    zloop_fn *voter_fn;         //  Voting socket handler
    void *voter_arg;            //  Arguments for voting handler
    zloop_fn *active_fn;        //  Call when become active
    void *active_arg;           //  Arguments for handler
    zloop_fn *passive_fn;         //  Call when become passive
    void *passive_arg;            //  Arguments for handler
};

//  The finite-state machine is the same as in the proof-of-concept server.
//  To understand this reactor in detail, first read the CZMQ zloop class.
//  .skip

//  We send state information every this often
//  If peer doesn't respond in two heartbeats, it is 'dead'
#define BSTAR_HEARTBEAT     1000        //  In msecs

//  Binary Star finite state machine (applies event to state)
//  Returns -1 if there was an exception, 0 if event was valid.

static int
s_execute_fsm (bstar_t *self)
{
    int rc = 0;
    //  Primary server is waiting for peer to connect
    //  Accepts CLIENT_REQUEST events in this state
    if (self->state == STATE_PRIMARY) {
        if (self->event == PEER_BACKUP) {
            zclock_log ("I: connected to backup (passive), ready as active");
            self->state = STATE_ACTIVE;
            if (self->active_fn)
                (self->active_fn) (self->loop, NULL, self->active_arg);
        }
        else
        if (self->event == PEER_ACTIVE) {
            zclock_log ("I: connected to backup (active), ready as passive");
            self->state = STATE_PASSIVE;
            if (self->passive_fn)
                (self->passive_fn) (self->loop, NULL, self->passive_arg);
        }
        else
        if (self->event == CLIENT_REQUEST) {
            // Allow client requests to turn us into the active if we've
            // waited sufficiently long to believe the backup is not
            // currently acting as active (i.e., after a failover)
            assert (self->peer_expiry > 0);
            if (zclock_time () >= self->peer_expiry) {
                zclock_log ("I: request from client, ready as active");
                self->state = STATE_ACTIVE;
                if (self->active_fn)
                    (self->active_fn) (self->loop, NULL, self->active_arg);
            } else
                // Don't respond to clients yet - it's possible we're
                // performing a failback and the backup is currently active
                rc = -1;
        }
    }
    else
    //  Backup server is waiting for peer to connect
    //  Rejects CLIENT_REQUEST events in this state
    if (self->state == STATE_BACKUP) {
        if (self->event == PEER_ACTIVE) {
            zclock_log ("I: connected to primary (active), ready as passive");
            self->state = STATE_PASSIVE;
            if (self->passive_fn)
                (self->passive_fn) (self->loop, NULL, self->passive_arg);
        }
        else
        if (self->event == CLIENT_REQUEST)
            rc = -1;
    }
    else
    //  Server is active
    //  Accepts CLIENT_REQUEST events in this state
    //  The only way out of ACTIVE is death
    if (self->state == STATE_ACTIVE) {
        if (self->event == PEER_ACTIVE) {
            //  Two actives would mean split-brain
            zclock_log ("E: fatal error - dual actives, aborting");
            rc = -1;
        }
    }
    else
    //  Server is passive
    //  CLIENT_REQUEST events can trigger failover if peer looks dead
    if (self->state == STATE_PASSIVE) {
        if (self->event == PEER_PRIMARY) {
            //  Peer is restarting - become active, peer will go passive
            zclock_log ("I: primary (passive) is restarting, ready as active");
            self->state = STATE_ACTIVE;
        }
        else
        if (self->event == PEER_BACKUP) {
            //  Peer is restarting - become active, peer will go passive
            zclock_log ("I: backup (passive) is restarting, ready as active");
            self->state = STATE_ACTIVE;
        }
        else
        if (self->event == PEER_PASSIVE) {
            //  Two passives would mean cluster would be non-responsive
            zclock_log ("E: fatal error - dual passives, aborting");
            rc = -1;
        }
        else
        if (self->event == CLIENT_REQUEST) {
            //  Peer becomes active if timeout has passed
            //  It's the client request that triggers the failover
            assert (self->peer_expiry > 0);
            if (zclock_time () >= self->peer_expiry) {
                //  If peer is dead, switch to the active state
                zclock_log ("I: failover successful, ready as active");
                self->state = STATE_ACTIVE;
            }
            else
                //  If peer is alive, reject connections
                rc = -1;
        }
        //  Call state change handler if necessary
        if (self->state == STATE_ACTIVE && self->active_fn)
            (self->active_fn) (self->loop, NULL, self->active_arg);
    }
    return rc;
}

static void
s_update_peer_expiry (bstar_t *self)
{
    self->peer_expiry = zclock_time () + 2 * BSTAR_HEARTBEAT;
}

//  Reactor event handlers...

//  Publish our state to peer
int s_send_state (zloop_t *loop, int timer_id, void *arg)
{
    bstar_t *self = (bstar_t *) arg;
    zstr_sendf (self->statepub,"%d", self->state);
    return 0;
}

//  Receive state from peer, execute finite state machine
int s_recv_state (zloop_t *loop, zmq_pollitem_t *poller, void *arg)
{
    bstar_t *self = (bstar_t *) arg;
    char *state = zstr_recv (poller->socket);
    if (state) {
        self->event = atoi (state);
        s_update_peer_expiry (self);
        free (state);
    }
    return s_execute_fsm (self);
}

//  Application wants to speak to us, see if it's possible
int s_voter_ready (zloop_t *loop, zmq_pollitem_t *poller, void *arg)
{
    bstar_t *self = (bstar_t *) arg;
    //  If server can accept input now, call appl handler
    self->event = CLIENT_REQUEST;
    if (s_execute_fsm (self) == 0)
        (self->voter_fn) (self->loop, poller, self->voter_arg);
    else {
        //  Destroy waiting message, no-one to read it
        zmsg_t *msg = zmsg_recv (poller->socket);
        zmsg_destroy (&msg);
    }
    return 0;
}

//  .until
//  .split constructor
//  This is the constructor for our {{bstar}} class. We have to tell it 
//  whether we're primary or backup server, as well as our local and 
//  remote endpoints to bind and connect to:

bstar_t *
bstar_new (int primary, char *local, char *remote)
{
    bstar_t
        *self;

    self = (bstar_t *) zmalloc (sizeof (bstar_t));

    //  Initialize the Binary Star
    self->ctx = zctx_new ();
    self->loop = zloop_new ();
    self->state = primary? STATE_PRIMARY: STATE_BACKUP;

    //  Create publisher for state going to peer
    self->statepub = zsocket_new (self->ctx, ZMQ_PUB);
    zsocket_bind (self->statepub, local);

    //  Create subscriber for state coming from peer
    self->statesub = zsocket_new (self->ctx, ZMQ_SUB);
    zsocket_set_subscribe (self->statesub,"");
    zsocket_connect (self->statesub, remote);

    //  Set-up basic reactor events
    zloop_timer (self->loop, BSTAR_HEARTBEAT, 0, s_send_state, self);
    zmq_pollitem_t poller = { self->statesub, 0, ZMQ_POLLIN };
    zloop_poller (self->loop, &poller, s_recv_state, self);
    return self;
}

//  .split destructor
//  The destructor shuts down the bstar reactor:

void
bstar_destroy (bstar_t **self_p)
{
    assert (self_p);
    if (*self_p) {
        bstar_t *self = *self_p;
        zloop_destroy (&self->loop);
        zctx_destroy (&self->ctx);
        free (self);
        *self_p = NULL;
    }
}

//  .split zloop method
//  This method returns the underlying zloop reactor, so we can add
//  additional timers and readers:

zloop_t *
bstar_zloop (bstar_t *self)
{
    return self->loop;
}

//  .split voter method
//  This method registers a client voter socket. Messages received
//  on this socket provide the CLIENT_REQUEST events for the Binary Star
//  FSM and are passed to the provided application handler. We require
//  exactly one voter per {{bstar}} instance:

int
bstar_voter (bstar_t *self, char *endpoint, int type, zloop_fn handler,
             void *arg)
{
    //  Hold actual handler+arg so we can call this later
    void *socket = zsocket_new (self->ctx, type);
    zsocket_bind (socket, endpoint);
    assert (!self->voter_fn);
    self->voter_fn = handler;
    self->voter_arg = arg;
    zmq_pollitem_t poller = { socket, 0, ZMQ_POLLIN };
    return zloop_poller (self->loop, &poller, s_voter_ready, self);
}

//  .split register state-change handlers
//  Register handlers to be called each time there's a state change:

void
bstar_new_active (bstar_t *self, zloop_fn handler, void *arg)
{
    assert (!self->active_fn);
    self->active_fn = handler;
    self->active_arg = arg;
}

void
bstar_new_passive (bstar_t *self, zloop_fn handler, void *arg)
{
    assert (!self->passive_fn);
    self->passive_fn = handler;
    self->passive_arg = arg;
}

//  .split enable/disable tracing
//  Enable/disable verbose tracing, for debugging:

void bstar_set_verbose (bstar_t *self, bool verbose)
{
    zloop_set_verbose (self->loop, verbose);
}

//  .split start the reactor
//  Finally, start the configured reactor. It will end if any handler
//  returns -1 to the reactor, or if the process receives SIGINT or SIGTERM:

int
bstar_start (bstar_t *self)
{
    assert (self->voter_fn);
    s_update_peer_expiry (self);
    return zloop_start (self->loop);
}
```


这就给我们提供了下面这个服务器的简短主程序.


  <summary><mark><font color=darkred>Binary Star server, using bstar reactor</font></mark></summary>

```c
//  Binary Star server, using bstar reactor

//  Lets us build this source without creating a library
#include"bstar.c"

//  Echo service
int s_echo (zloop_t *loop, zmq_pollitem_t *poller, void *arg)
{
    zmsg_t *msg = zmsg_recv (poller->socket);
    zmsg_send (&msg, poller->socket);
    return 0;
}

int main (int argc, char *argv [])
{
    //  Arguments can be either of:
    //      -p  primary server, at tcp://localhost:5001
    //      -b  backup server, at tcp://localhost:5002
    bstar_t *bstar;
    if (argc == 2 && streq (argv [1],"-p")) {
        printf ("I: Primary active, waiting for backup (passive)\n");
        bstar = bstar_new (BSTAR_PRIMARY,
           "tcp://*:5003","tcp://localhost:5004");
        bstar_voter (bstar,"tcp://*:5001", ZMQ_ROUTER, s_echo, NULL);
    }
    else
    if (argc == 2 && streq (argv [1],"-b")) {
        printf ("I: Backup passive, waiting for primary (active)\n");
        bstar = bstar_new (BSTAR_BACKUP,
           "tcp://*:5004","tcp://localhost:5003");
        bstar_voter (bstar,"tcp://*:5002", ZMQ_ROUTER, s_echo, NULL);
    }
    else {
        printf ("Usage: bstarsrvs { -p | -b }\n");
        exit (0);
    }
    bstar_start (bstar);
    bstar_destroy (&bstar);
    return 0;
}
```
</details>

## 无经纪商的可靠性(自由人模式)

当我们经常把ZeroMQ解释为"无borkers的消息传递"时,如此关注基于borkers的可靠性似乎很讽刺.然而,在消息传递中,正如在现实生活中一样,中间人既是一种负担,也是一种好处.在实践中,大多数消息架构都从分布式和中介式消息传递的混合中受益.当你能自由决定你想做什么权衡时,你会得到最好的结果.这就是为什么我可以开车20分钟到批发商那里为一个聚会买五箱酒,但我也可以步行10分钟到街角的商店买一瓶酒来吃.我们对时间,能量和成本的高度情境敏感的相对估值对现实世界的经济至关重要.而且它们对基于消息的最佳架构也是至关重要的.

这就是为什么ZeroMQ没有*强加一个以borkers为中心的架构,尽管它确实给你提供了建立borkers(又称*代理)的工具,而且到目前为止我们已经建立了十几个不同的borkers,只是为了练习.

因此,在本章的最后,我们将解构我们到目前为止所构建的基于代理的可靠性,并将其转回一个分布式的对等架构,我称之为自由人模式.我们的用例将是一个名称解析服务.这是ZeroMQ架构的一个常见问题:我们如何知道要连接到哪个端点？在代码中硬编码TCP/IP地址是非常脆弱的.使用配置文件会造成管理上的恶梦.想象一下,如果你不得不在你使用的每台电脑或手机上手工配置你的网络浏览器,以实现"google.com"是"74.125.230.82".

一个ZeroMQ名称服务(我们将做一个简单的实现)必须做以下工作.

* 将一个逻辑名称解析为至少一个绑定端点,以及一个连接端点.一个现实的名字服务将提供多个绑定端点,可能还有多个连接端点.

* 允许我们管理多个并行环境,例如"测试"和"生产",而无需修改代码.

* 要可靠,因为如果它不可用,应用程序将无法连接到网络.

把名字服务放在面向服务的Majordomo代理后面,从某些角度看是很聪明的.然而,把名字服务作为一个服务器暴露出来,让客户直接连接到它,这样做更简单,也更令人吃惊.如果我们这样做对了,名字服务就成了我们需要在代码或配置文件中硬编码的*唯一的全球网络端点.

```[[type="textdiagram" title="The Freelance Pattern"]]
#-----------#   #-----------#   #-----------#
|  Client   |   |  Client   |   |  Client   |
'-----------'   '-----------'   '-----------'
   connect         connect         connect
      |               |               |
      |               |               |
      +---------------+---------------+
      |               |               |
      |               |               |
    bind            bind            bind
.-----------.   .-----------.   .-----------.
|  Server   |   |  Server   |   |  Server   |
#-----------#   #-----------#   #-----------#
```

我们旨在处理的故障类型是服务器崩溃和重启,服务器繁忙循环,服务器过载和网络问题.为了获得可靠性,我们将创建一个名称服务器池,这样如果一个服务器崩溃或消失,客户可以连接到另一个,以此类推.在实践中,两个就足够了.但在这个例子中,我们将假设池子可以是任何大小[图].

在这个架构中,一大批客户直接连接到一小批服务器.服务器与它们各自的地址绑定.这与像Majordomo这样基于borkers的方法有根本的不同,在这种方法中,worker连接到borkers.客户有几个选择.

* 使用REQ套接字和Lazy Pirate模式.这很简单,但需要一些额外的智能,这样客户就不会愚蠢地一次又一次地重新连接到死去的服务器.

* 使用DEALER套接字,发出请求(这些请求将被负载平衡到所有连接的服务器上),直到他们得到一个回复.有效,但不优雅.

* 使用ROUTER套接字,这样客户就可以解决特定的服务器.但客户如何知道服务器套接字的身份？要么服务器必须先ping客户(复杂),要么服务器必须使用一个硬编码的,客户知道的固定身份(讨厌).

我们将在下面的小节中分别讨论这些问题.

### 模式一:简单重试和故障转移

因此,我们的菜单似乎提供了:简单的,粗暴的,复杂的或讨厌的.让我们从简单的开始,然后再解决一些问题.我们采用Lazy Pirate,并重写它,使之与多个服务器端点一起工作.

首先启动一个或几个服务器,指定一个绑定的端点作为参数.



  <summary><mark><font color=darkred>Freelance server - Model 1</font></mark></summary>

```c++
//  Freelance server - Model 1
//  Trivial echo service

#include <iostream>
#include <zmq.hpp>

int main(int argc, char* argv[]) {
    if (argc < 2) {
        std::cout <<"I: syntax:" << argv[0] <<" <endpoint>" << std::endl;
        return 0;
    }
    zmq::context_t context{1};
    zmq::socket_t server(context, zmq::socket_type::rep);
    server.bind(argv[1]);

    std::cout <<"I: echo service is ready at" << argv[1] << std::endl;

    while (true) {
        zmq::message_t message;
        try {
            server.recv(message, zmq::recv_flags::none);
        } catch (zmq::error_t& e) {
            if (e.num() == EINTR)
                std::cout <<"W: interrupted" << std::endl;
            else
                std::cout <<"E: error, errnum =" << e.num() <<", what =" << e.what()
                          << std::endl;
            break;  // Interrupted
        }
        server.send(message, zmq::send_flags::none);
    }
    return 0;
}
```
</details>

然后启动客户端,指定一个或多个连接端点作为参数.


  <summary><mark><font color=darkred>Freelance client - Model 1</font></mark></summary>

```c++
//  Freelance client - Model 1
//  Uses REQ socket to query one or more services
#include <iostream>
#include <zmq.hpp>
#include <zmq_addon.hpp>

const int REQUEST_TIMEOUT = 1000;
const int MAX_RETRIES = 3;  //  Before we abandon

static std::unique_ptr<zmq::message_t> s_try_request(zmq::context_t &context,
                                                     const std::string &endpoint,
                                                     const zmq::const_buffer &request) {
    std::cout <<"I: trying echo service at" << endpoint << std::endl;
    zmq::socket_t client(context, zmq::socket_type::req);

    // Set ZMQ_LINGER to REQUEST_TIMEOUT milliseconds, otherwise if we send a message to a server
    // that is not working properly or even not exist, we may never be able to exit the program
    client.setsockopt(ZMQ_LINGER, REQUEST_TIMEOUT);

    client.connect(endpoint);

    //  Send request, wait safely for reply
    zmq::message_t message(request.data(), request.size());
    client.send(message, zmq::send_flags::none);

    zmq_pollitem_t items[] = {{client, 0, ZMQ_POLLIN, 0}};
    zmq::poll(items, 1, REQUEST_TIMEOUT);
    std::unique_ptr<zmq::message_t> reply = std::make_unique<zmq::message_t>();
    zmq::recv_result_t recv_result;
    if (items[0].revents & ZMQ_POLLIN) recv_result = client.recv(*reply, zmq::recv_flags::none);
    if (!recv_result) {
        reply.release();
    }
    return reply;
}

//  .split client task
//  The client uses a Lazy Pirate strategy if it only has one server to talk
//  to. If it has two or more servers to talk to, it will try each server just
//  once:

int main(int argc, char *argv[]) {
    zmq::context_t context{1};
    zmq::const_buffer request = zmq::str_buffer("Hello World!");

    std::unique_ptr<zmq::message_t> reply;

    int endpoints = argc - 1;

    if (endpoints == 0)
        std::cout <<"I: syntax:" << argv[0] <<"<endpoint> ..." << std::endl;
    else if (endpoints == 1) {
        //  For one endpoint, we retry N times
        int retries;
        for (retries = 0; retries < MAX_RETRIES; retries++) {
            std::string endpoint = std::string(argv[1]);
            reply = s_try_request(context, endpoint, request);
            if (reply) break;  //  Successful
            std::cout <<"W: no response from" << endpoint <<" retrying...\n" << std::endl;
        }
    } else {
        //  For multiple endpoints, try each at most once
        int endpoint_nbr;
        for (endpoint_nbr = 0; endpoint_nbr < endpoints; endpoint_nbr++) {
            std::string endpoint = std::string(argv[endpoint_nbr + 1]);
            reply = s_try_request(context, endpoint, request);
            if (reply) break;  //  Successful
            std::cout <<"W: no response from" << endpoint << std::endl;
        }
    }
    if (reply)
        std::cout <<"Service is running OK. Received message:" << reply->to_string() << std::endl;
    return 0;
}
```
</details>

一个运行样本是.

```
flserver1 tcp://*:5555 &
flserver1 tcp://*:5556 &
flclient1 tcp://localhost:5555 tcp://localhost:5556
```

虽然基本方法是Lazy Pirate,但客户端的目的只是为了获得一个成功的回复.它有两种技术,取决于你是运行单台服务器还是多台服务器.

* 对于单台服务器,客户端将重试几次,与Lazy Pirate完全一样.
* 对于多个服务器,客户端将最多尝试每个服务器一次,直到它收到回复或尝试了所有服务器.

这解决了Lazy Pirate的主要弱点,即它不能故障切换到备份或备用服务器.

然而,这种设计在实际应用中并不能很好地工作.如果我们连接了许多套接字,而我们的主名称服务器宕机了,我们每次都会经历这种痛苦的超时.

### 模式二:残酷的霰弹枪大屠杀

让我们把我们的客户端换成使用DEALER套接字.我们的目标是确保我们在最短的时间内得到回复,无论某个服务器是正常运行还是故障.我们的客户采取这种方式.

* 我们设置好东西,连接到所有的服务器.
*当我们有一个请求时,我们会根据我们的服务器的数量把它轰出去.
* 我们等待第一个回复,并接受它.
* 我们忽略任何其他的回复.

实际发生的情况是,当所有的服务器都在运行时,ZeroMQ将分配请求,使每个服务器得到一个请求并发送一个回复.当任何服务器离线和断开连接时,ZeroMQ将把请求分配给其余的服务器.所以在某些情况下,一个服务器可能会不止一次地收到相同的请求.

对客户端来说,更烦人的是,我们会得到多个回复,但不能保证我们会得到精确的回复数量.请求和回复可能会丢失(例如,如果服务器在处理一个请求时崩溃).

所以我们必须对请求进行编号,并忽略任何与请求编号不匹配的回复.我们的模型一服务器将工作,因为它是一个回声服务器,但巧合并不是一个很好的理解基础.所以我们要做一个模型二的服务器,它可以咀嚼消息并返回一个正确编号的回复,内容为"OK".我们将使用由两部分组成的消息:一个序列号和一个正文.

启动一个或多个服务器,每次都指定一个绑定端点.


  <summary><mark><font color=darkred>Freelance server - Model 2</font></mark></summary>

```c++

//  Freelance server - Model 2
//  Does some work, replies OK, with message sequencing

#include <iostream>
#include <zmqpp/zmqpp.hpp>

int main(int argc, char *argv[]) {
    if (argc < 2) {
        std::cout <<"I: syntax:" << argv[0] <<" <endpoint>" << std::endl;
        return 0;
    }
    zmqpp::context context;
    zmqpp::socket server(context, zmqpp::socket_type::reply);
    server.bind(argv[1]);

    std::cout <<"I: echo service is ready at" << argv[1] << std::endl;
    while (true) {
        zmqpp::message request;
        try {
            server.receive(request);
        } catch (zmqpp::zmq_internal_exception &e) {
            if (e.zmq_error() == EINTR)
                std::cout <<"W: interrupted" << std::endl;
            else
                std::cout <<"E: error, errnum =" << e.zmq_error() <<", what =" << e.what()
                          << std::endl;
            break;  // Interrupted
        }
        //  Fail nastily if run against wrong client
        assert(request.parts() == 2);

        uint identity;
        request.get(identity, 0);
        // std::cout <<"Received sequence:" << identity << std::endl;

        zmqpp::message reply;
        reply.push_back(identity);
        reply.push_back("OK");

        server.send(reply);
    }
    return 0;
}

```
</details>

然后启动客户端,将连接端点指定为参数:


  <summary><mark><font color=darkred>Freelance client - Model 2</font></mark></summary>

```c++
//  Freelance client - Model 2
//  Uses DEALER socket to blast one or more services

#include <chrono>
#include <iostream>
#include <memory>
#include <zmqpp/zmqpp.hpp>

//  If not a single service replies within this time, give up
const int GLOBAL_TIMEOUT = 2500;
//  Total requests times
const int TOTAL_REQUESTS = 10000;

//  .split class implementation
//  Here is the {{flclient}} class implementation. Each instance has a
//  context, a DEALER socket it uses to talk to the servers, a counter
//  of how many servers it's connected to, and a request sequence number:
class flclient {
   public:
    flclient();
    ~flclient() {}
    void connect(const std::string &endpoint);
    std::unique_ptr<zmqpp::message> request(zmqpp::message &request);

   private:
    zmqpp::context context_;  //  Our context
    zmqpp::socket socket_;    //  DEALER socket talking to servers
    size_t servers_;          //  How many servers we have connected to
    uint sequence_;           //  Number of requests ever sent
};

//  Constructor
flclient::flclient() : socket_(context_, zmqpp::socket_type::dealer) {
    socket_.set(zmqpp::socket_option::linger, GLOBAL_TIMEOUT);
    servers_ = 0;
    sequence_ = 0;
}

//  Connect to new server endpoint
void flclient::connect(const std::string &endpoint) {
    socket_.connect(endpoint);
    servers_++;
}

//  .split request method
//  This method does the hard work. It sends a request to all
//  connected servers in parallel (for this to work, all connections
//  must be successful and completed by this time). It then waits
//  for a single successful reply, and returns that to the caller.
//  Any other replies are just dropped:

std::unique_ptr<zmqpp::message> flclient::request(zmqpp::message &request) {
    //  Prefix request with sequence number and empty envelope
    request.push_front(++sequence_);
    request.push_front("");

    //  Blast the request to all connected servers
    size_t server;
    for (server = 0; server < servers_; server++) {
        zmqpp::message msg;
        msg.copy(request);
        socket_.send(msg);
    }
    //  Wait for a matching reply to arrive from anywhere
    //  Since we can poll several times, calculate each one
    std::unique_ptr<zmqpp::message> reply;

    zmqpp::poller poller;
    poller.add(socket_, zmqpp::poller::poll_in);
    auto endTime = std::chrono::system_clock::now() + std::chrono::milliseconds(GLOBAL_TIMEOUT);
    while (std::chrono::system_clock::now() < endTime) {
        int milliSecondsToWait = std::chrono::duration_cast<std::chrono::milliseconds>(
                                     endTime - std::chrono::system_clock::now())
                                     .count();
        if (poller.poll(milliSecondsToWait)) {
            if (poller.has_input(socket_)) {
                reply = std::make_unique<zmqpp::message>();
                //  Reply is [empty][sequence][OK]
                socket_.receive(*reply);
                assert(reply->parts() == 3);
                reply->pop_front();
                uint sequence;
                reply->get(sequence, 0);
                reply->pop_front();
                // std::cout <<"Current sequence:" << sequence_ <<", Server reply:" << sequence
                //           << std::endl;
                if (sequence == sequence_)
                    break;
                else
                    reply.release();
            }
        }
    }
    return reply;
}

int main(int argc, char *argv[]) {
    if (argc == 1) {
        std::cout <<"I: syntax:" << argv[0] <<" <endpoint> ..." << std::endl;
        return 0;
    }
    //  Create new freelance client object
    flclient client;

    //  Connect to each endpoint
    int argn;
    for (argn = 1; argn < argc; argn++) client.connect(argv[argn]);

    //  Send a bunch of name resolution 'requests', measure time
    int requests = TOTAL_REQUESTS;
    auto startTime = std::chrono::steady_clock::now();
    while (requests--) {
        zmqpp::message request;
        request.push_back("random name");
        std::unique_ptr<zmqpp::message> reply;
        reply = client.request(request);
        if (!reply) {
            std::cout <<"E: name service not available, aborting" << std::endl;
            break;
        }
    }
    auto endTime = std::chrono::steady_clock::now();
    std::cout
        <<"Average round trip cost:"
        << std::chrono::duration_cast<std::chrono::microseconds>(endTime - startTime).count() /
               TOTAL_REQUESTS
        <<" µs" << std::endl;
    return 0;
}
```
</details>

这里有一些关于客户端实现的事情需要注意.

* 客户端的结构是一个漂亮的基于类的API,它隐藏了创建ZeroMQ上下文和套接字以及与服务器对话的肮脏工作.也就是说,如果用霰弹枪打中腰部可以被称为"交谈"的话.

* 如果客户端在几秒钟内找不到*任何响应的服务器,它就会放弃追赶.

* 客户端必须创建一个有效的REP信封,即在信息的前面添加一个空的信息框架.

客户端执行10000个名称解析请求(假的,因为我们的服务器基本上什么都不做)并测量平均成本.在我的测试箱上,与一个服务器交谈,这需要大约60微秒.与三台服务器通话,需要大约80微秒.

我们的霰弹枪方法的优点和缺点是.

* 优点:它很简单,容易制作,容易理解.
* 优点:它能完成故障转移的工作,而且工作迅速,只要至少有一台服务器在运行.
* 缺点:它创造了冗余的网络流量.
* 缺点:我们无法确定服务器的优先次序,即先主后次.
* 弊端:服务器每次最多只能做一个请求,期间.

### 模式三.复杂而讨厌的

猎枪式的方法似乎好得不像真的.让我们科学地研究一下所有的替代方案.我们要探索复杂/讨厌的选项,即使只是为了最终意识到我们更喜欢粗暴.啊,我生命中的故事.

我们可以通过切换到ROUTER套接字来解决客户端的主要问题.这让我们可以向特定的服务器发送请求,避开我们知道已经死亡的服务器,而且一般来说,我们想怎么聪明就怎么聪明.我们也可以通过切换到ROUTER套接字来解决服务器的主要问题(单线程性).

但是在两个匿名套接字(还没有设置身份)之间做ROUTER到ROUTER是不可能的.双方只有在收到第一条消息时才会产生一个身份(对于另一个对等体),因此在第一次收到消息之前,双方都不能与对方交谈.摆脱这个难题的唯一方法是作弊,在一个方向上使用硬编码的身份.在客户端/服务器的情况下,正确的作弊方式是让客户端"知道"服务器的身份.反过来做将是疯狂的,除了复杂和讨厌之外,因为任何数量的客户端都应该能够独立出现.对于一个种族灭绝的独裁者来说,疯狂,复杂和讨厌是很好的属性,但对于软件来说则是糟糕的属性.

与其发明另一个概念来管理,我们不如使用连接端点作为身份.这是一个独特的字符串,双方都可以在这个字符串上达成一致,而不需要比他们已经拥有的猎枪模型更多的事先知识.这是一种偷偷摸摸的,有效的连接两个ROUTER套接字的方法.

记住ZeroMQ的身份是如何工作的.服务器ROUTER套接字在绑定其套接字之前设置一个身份.当客户端连接时,在任何一方发送真正的消息之前,他们做一个小小的握手来交换身份.客户端ROUTER套接字在没有设置身份时,向服务器发送一个空身份.服务器生成一个随机的UUID来指定客户,供其自己使用.服务器将它的身份(我们已经同意将是一个端点字符串)发送给客户端.

这意味着我们的客户端可以在连接建立后立即将消息发送到服务器(即在其ROUTER套接字上发送,指定服务器端点为身份).这不是在做完{{zmq_connect[3]}}后立即进行的,而是在之后的某个随机时间.这里有一个问题:我们不知道服务器什么时候会真正可用并完成其连接握手.如果服务器是在线的,可能是在几毫秒之后.如果服务器坏了,系统管理员出去吃午饭了,那就可能是一个小时以后.

这里有一个小小的悖论.我们需要知道服务器何时开始连接并可用于工作.在Freelance模式中,与我们在本章前面看到的基于代理的模式不同,服务器在被说到之前是沉默的.因此,在服务器告诉我们它在线之前,我们不能和它说话,而在我们问它之前,它也不能这样做.

我的解决方案是混合使用模型2中的霰弹枪方法,也就是说,我们会向任何可以开枪的东西(无害)射击,如果有东西移动,我们就知道它是活的.我们不打算发射真正的请求,而是一种乒乓式的心跳.

这又把我们带到了协议的领域,所以这里有一个[http://rfc.zeromq.org/spec:10 简短的规范,定义了Freelance客户和服务器如何交换ping-pong命令和请求-回复命令].

作为一个服务器,它的实现很简短,很贴心.这是我们的回声服务器,模型三,现在讲FLP.

  <summary><mark><font color=darkred>Freelance server - Model 3</font></mark></summary>

```c++
//  Freelance server - Model 3
//  Uses an ROUTER/ROUTER socket but just one thread

#include <iostream>
#include <zmqpp/zmqpp.hpp>

void dump_binary(const zmqpp::message &msg);

int main(int argc, char *argv[]) {
    int verbose = (argc > 1 && (std::string(argv[1]) =="-v"));
    if (verbose) std::cout <<"verbose active" << std::endl;
    zmqpp::context context;

    //  Prepare server socket with predictable identity
    std::string bind_endpoint ="tcp://*:5555";
    std::string connect_endpoint ="tcp://localhost:5555";
    zmqpp::socket server(context, zmqpp::socket_type::router);
    server.set(zmqpp::socket_option::identity, connect_endpoint);
    server.bind(bind_endpoint);
    std::cout <<"I: service is ready at" << bind_endpoint << std::endl;

    while (true) {
        zmqpp::message request;
        try {
            server.receive(request);
        } catch (zmqpp::zmq_internal_exception &e) {
            if (e.zmq_error() == EINTR)
                std::cout <<"W: interrupted" << std::endl;
            else
                std::cout <<"E: error, errnum =" << e.zmq_error() <<", what =" << e.what()
                          << std::endl;
            break;  // Interrupted
        }
        if (verbose) {
            std::cout <<"Message received from client, all data will dump." << std::endl;
            dump_binary(request);
        }

        //  Frame 0: identity of client
        //  Frame 1: PING, or client control frame
        //  Frame 2: request body
        std::string identity;
        std::string control;
        request >> identity >> control;

        zmqpp::message reply;
        if (control =="PING")
            reply.push_back("PONG");
        else {
            reply.push_back(control);
            reply.push_back("OK");
        }
        reply.push_front(identity);
        if (verbose) {
            std::cout <<"Message reply to client dump." << std::endl;
            dump_binary(reply);
        }
        server.send(reply);
    }
    return 0;
}

void dump_binary(const zmqpp::message &msg) {
    std::cout <<"Dump message ..." << std::endl;
    for (size_t part = 0; part < msg.parts(); ++part) {
        std::cout <<"Part:" << part << std::endl;
        const unsigned char *bin = static_cast<const unsigned char *>(msg.raw_data(part));
        for (size_t i = 0; i < msg.size(part); ++i) {
            std::cout << std::hex << static_cast<uint16_t>(*(bin++)) <<"";
        }
        std::cout << std::endl;
    }
    std::cout <<"Dump finish ..." << std::endl;
}

```
</details>

然而,自由职业者的客户端已经变得很大了.为了清楚起见,我们把它分成了一个应用实例和一个从事艰苦工作的类.这里是顶级的应用程序.

<summary><mark><font color=darkred>Freelance client - Model 3</font></mark></summary>

```c++
//  Freelance client - Model 3
//  Uses flcliapi class to encapsulate Freelance pattern

#include <chrono>

#include"flcliapi.hpp"

const int TOTAL_REQUESTS = 10000;

int main(void) {
    //  Create new freelance client object
    Flcliapi client;

    //  Connect to several endpoints
    client.connect("tcp://localhost:5555");
    client.connect("tcp://localhost:5556");
    client.connect("tcp://localhost:5557");

    //  Send a bunch of name resolution 'requests', measure time
    int requests = TOTAL_REQUESTS;
    auto startTime = std::chrono::steady_clock::now();
    while (requests--) {
        zmqpp::message request;
        request.push_back("random name");
        std::unique_ptr<zmqpp::message> reply;
        reply = client.request(request);
        if (!reply) {
            std::cout <<"E: name service not available, aborting" << std::endl;
            break;
        }
    }
    auto endTime = std::chrono::steady_clock::now();
    std::cout
        <<"Average round trip cost:"
        << std::chrono::duration_cast<std::chrono::microseconds>(endTime - startTime).count() /
               TOTAL_REQUESTS
        <<" µs" << std::endl;
    return 0;
}
```
</details>

而这里,几乎和Majordomoborkers一样复杂和庞大,是客户端API类.

<summary><mark><font color=darkred>flcliapi class - Freelance Pattern agent class</font></mark></summary>

```c++ 
//  flcliapi class - Freelance Pattern agent class
//  Implements the Freelance Protocol at http://rfc.zeromq.org/spec:10

#include"flcliapi.hpp"

//  If no server replies within this time, abandon request
const int GLOBAL_TIMEOUT = 3000;  //  msecs
//  PING interval for servers we think are alive
const int PING_INTERVAL = 500;  //  msecs
//  Server considered dead if silent for this long
const int SERVER_TTL = 1000;  //  msecs

//  This API works in two halves, a common pattern for APIs that need to
//  run in the background. One half is an frontend object our application
//  creates and works with; the other half is a backend"agent" that runs
//  in a background thread. The frontend talks to the backend over an
//  inproc pipe socket created by actor object:

//  Constructor
Flcliapi::Flcliapi()
    : actor_(std::bind(&Flcliapi::agent, this, std::placeholders::_1, std::ref(context_))) {}

Flcliapi::~Flcliapi() {}

//  connect interface
//  To implement the connect method, the frontend object sends a multipart
//  message to the backend agent. The first part is a string"CONNECT", and
//  the second part is the endpoint. It waits 100msec for the connection to
//  come up, which isn't pretty, but saves us from sending all requests to a
//  single server, at startup time:
void Flcliapi::connect(const std::string& endpoint) {
    zmqpp::message msg;
    msg.push_back("CONNECT");
    msg.push_back(endpoint);
    actor_.pipe()->send(msg);
    std::this_thread::sleep_for(std::chrono::milliseconds(100));  //  Allow connection to come up
}

//  request interface
//  To implement the request method, the frontend object sends a message
//  to the backend, specifying a command"REQUEST" and the request message:
std::unique_ptr<zmqpp::message> Flcliapi::request(zmqpp::message& request) {
    assert(request.parts() > 0);
    request.push_front("REQUEST");
    actor_.pipe()->send(request);
    std::unique_ptr<zmqpp::message> reply = std::make_unique<zmqpp::message>();
    actor_.pipe()->receive(*reply);
    if (0 != reply->parts()) {
        if (reply->get(0) =="FAILED") reply.release();
    } else {
        reply.release();
    }
    return reply;
}

Server::Server(const std::string& endpoint) {
    endpoint_ = endpoint;
    alive_ = false;
    ping_at_ = std::chrono::steady_clock::now() + std::chrono::milliseconds(PING_INTERVAL);
    expires_ = std::chrono::steady_clock::now() + std::chrono::milliseconds(SERVER_TTL);
}

Server::~Server() {}

int Server::ping(zmqpp::socket& socket) {
    if (std::chrono::steady_clock::now() >= ping_at_) {
        zmqpp::message ping;
        ping.push_back(endpoint_);
        ping.push_back("PING");
        socket.send(ping);
        ping_at_ = std::chrono::steady_clock::now() + std::chrono::milliseconds(PING_INTERVAL);
    }
    return 0;
}

int Server::tickless(std::chrono::time_point<std::chrono::steady_clock>& tickless_at) {
    if (tickless_at > ping_at_) tickless_at = ping_at_;
    return 0;
}

Agent::Agent(zmqpp::context& context, zmqpp::socket* pipe)
    : context_(context), pipe_(pipe), router_(context, zmqpp::socket_type::router) {
    router_.set(zmqpp::socket_option::linger, GLOBAL_TIMEOUT);
    sequence_ = 0;
}
Agent::~Agent() {}

//  control messages
//  This method processes one message from our frontend class
//  (it's going to be CONNECT or REQUEST):
void Agent::control_message(std::unique_ptr<zmqpp::message> msg) {
    std::string command = msg->get(0);
    msg->pop_front();

    if (command =="CONNECT") {
        std::string endpoint = msg->get(0);
        msg->pop_front();
        std::cout <<"I: connecting to" << endpoint <<"..." << std::endl;
        try {
            router_.connect(endpoint);
        } catch (zmqpp::zmq_internal_exception& e) {
            std::cerr <<"failed to bind to endpoint" << endpoint <<":" << e.what() << std::endl;
            return;
        }
        std::shared_ptr<Server> server = std::make_shared<Server>(endpoint);
        servers_.insert(std::pair<std::string, std::shared_ptr<Server>>(endpoint, server));
        // actives_.push_back(server);
        server->setPingAt(std::chrono::steady_clock::now() +
                          std::chrono::milliseconds(PING_INTERVAL));
        server->setExpires(std::chrono::steady_clock::now() +
                           std::chrono::milliseconds(SERVER_TTL));
    } else if (command =="REQUEST") {
        assert(!request_);  //  Strict request-reply cycle
        //  Prefix request with sequence number and empty envelope
        msg->push_front(++sequence_);
        //  Take ownership of request message
        request_ = std::move(msg);
        //  Request expires after global timeout
        expires_ = std::chrono::steady_clock::now() + std::chrono::milliseconds(GLOBAL_TIMEOUT);
    }
}

//  .split router messages
//  This method processes one message from a connected
//  server:
void Agent::router_message() {
    zmqpp::message reply;
    router_.receive(reply);
    //  Frame 0 is server that replied
    std::string endpoint = reply.get(0);
    reply.pop_front();
    assert(servers_.count(endpoint));
    std::shared_ptr<Server> server = servers_.at(endpoint);
    if (!server->isAlive()) {
        actives_.push_back(server);
        server->setAlive(true);
    }
    server->setPingAt(std::chrono::steady_clock::now() + std::chrono::milliseconds(PING_INTERVAL));
    server->setExpires(std::chrono::steady_clock::now() + std::chrono::milliseconds(SERVER_TTL));

    // Frame 1 may be sequence number for reply
    uint sequence;
    reply.get(sequence, 0);
    reply.pop_front();
    if (request_) {
        if (sequence == sequence_) {
            request_.release();
            reply.push_front("OK");
            pipe_->send(reply);
        }
    }
}

//  .split backend agent implementation
//  Finally, here's the agent task itself, which polls its two sockets
//  and processes incoming messages:
bool Flcliapi::agent(zmqpp::socket* pipe, zmqpp::context& context) {
    Agent self(context, pipe);

    zmqpp::poller poller;
    poller.add(*self.getPipe());
    poller.add(self.getRouter());
    pipe->send(zmqpp::signal::ok);  // signal we successfully started
    while (true) {
        //  Calculate tickless timer, up to 1 hour
        std::chrono::time_point<std::chrono::steady_clock> tickless =
            std::chrono::steady_clock::now() + std::chrono::hours(1);
        if (self.request_ && tickless > self.expires_) tickless = self.expires_;
        for (auto& kv : self.servers_) {
            kv.second->tickless(tickless);
        }
        if (poller.poll(std::chrono::duration_cast<std::chrono::milliseconds>(
                            tickless - std::chrono::steady_clock::now())
                            .count())) {
            if (poller.has_input(*self.getPipe())) {
                std::unique_ptr<zmqpp::message> msg = std::make_unique<zmqpp::message>();
                pipe->receive(*msg);
                if (msg->is_signal()) {
                    zmqpp::signal sig;
                    msg->get(sig, 0);
                    if (sig == zmqpp::signal::stop) break;  // actor receive stop signal, exit

                } else
                    self.control_message(std::move(msg));
            }
            if (poller.has_input(self.getRouter())) self.router_message();
        }

        //  If we're processing a request, dispatch to next server
        if (self.request_) {
            if (std::chrono::steady_clock::now() >= self.expires_) {
                //  Request expired, kill it
                self.request_.release();
                self.getPipe()->send("FAILED");
            } else {
                //  Find server to talk to, remove any expired ones
                while (self.actives_.size() > 0) {
                    auto& server = self.actives_.front();
                    if (std::chrono::steady_clock::now() >= server->getExpires()) {
                        server->setAlive(false);
                        self.actives_.pop_front();
                    } else {
                        zmqpp::message request;
                        request.copy(*self.request_);
                        request.push_front(server->getEndpoint());
                        self.getRouter().send(request);
                        break;
                    }
                }
            }
        }

        for (auto& kv : self.servers_) {
            kv.second->ping(self.getRouter());
        }
    }
    return true;  // will send signal::ok to signal successful shutdown
}
```

</details>

这个API的实现相当复杂,使用了一些我们以前没有见过的技术.

**多线程API**:客户端API由两部分组成,一个同步的{{flcliapi}}类在应用线程中运行,另一个异步的*agent*类作为后台线程运行.还记得ZeroMQ是如何使创建多线程的应用程序变得容易的吗？flcliapi和代理类通过{{inproc}}套接字与对方的消息对话.所有ZeroMQ方面(如创建和销毁上下文)都隐藏在API中.代理实际上就像一个小型的borkers,在后台与服务器对话,因此当我们提出请求时,它可以尽最大努力到达它认为可用的服务器.

**无滴答的轮询计时器**:在以前的轮询循环中,我们总是使用一个固定的滴答间隔,例如1秒,这很简单,但在对电源敏感的客户端(如笔记本或手机)上却不是很好,因为唤醒CPU会耗费电能.为了好玩,也为了帮助拯救地球,代理使用了一个*tickless计时器*,它根据我们期待的下一次超时来计算轮询延迟.一个适当的实现会保持一个有序的超时列表.我们只需检查所有的超时,并计算出直到下一次超时的投票延迟.

## 结语

在本章中,我们已经看到了各种可靠的请求-回复机制,每一种都有一定的成本和好处.示例代码虽然没有经过优化,但基本可以用于实际使用.在所有不同的模式中,在生产中使用的两个模式是基于borkers的可靠性的Majordomo模式和无borkers的可靠性的Freelance模式.
