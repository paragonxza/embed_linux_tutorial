消息队列
========

Linux下的进程通信手段基本上是从Unix平台上的进程通信手段继承而来的。
而对Unix发展做出重大贡献的两大主力AT&T的贝尔实验室 以及 BSD（加州大学伯克利分校的伯克利软件发布中心），
他们在进程间通信方面的侧重点有所不同；

-   前者对Unix早期的进程间通信手段进行了系统的改进和扩充，
    形成了“system-V IPC”，通信进程局限在单个计算机内（同一个设备的不同进程间通讯）；
-   而后者则跳过了该限制，形成了基于套接字（socket）的进程间通信机制（多用于不同设备的进程间通讯）。
    Linux则把两者继承了下来，所以说Linux才是最成功的，既有“system-V IPC”，又支持“socket”。

**消息队列、共享内存 和 信号量** 被统称为 system-V IPC，V 是罗马数字5，
是 Unix 的AT&T 分支的其中一个版本，一般习惯称呼他们为 IPC对象，这些对象的操作接口都比较类似，
在系统中他们都使用一种叫做 key 的键值来唯一标识，而且他们都是“持续性”资源——即他们被创建之后，
不会因为进程的退出而消失，而会持续地存在，除非调用特殊的函数或者命令删除他们。

Linux的IPC对象（包括消息队列、共享内存和信号量）在内核内部使用链表维护，
不同的对象使用 ``IPC标识符`` 来标识，如消息队列标识符 msqid、共享内存标识符 shmid，信号量标识符 semid。

对于用户来说，内核提供了简洁的接口，不同的进程通过 ``IPC关键字（key）`` 即可访问具体的对象。

通过如下命令可以查看系统当前的IPC对象，没有使用的情况下可能为空：

.. code:: bash

    # 查询系统当前的IPC对象
    ipcs 

    # 以下是示例输出，没有使用的情况下可能为空
    --------- 消息队列 -----------
    键        msqid      拥有者  权限     已用字节数 消息      
    0x000004d2 98345      flyleaf    666        0            0  

    ------------ 共享内存段 --------------
    键        shmid      拥有者  权限     字节     连接数  状态      

    --------- 信号量数组 -----------
    键        semid      拥有者  权限     nsems     




消息队列的基本概念
-------------------------------

消息队列提供了一种从一个进程向另一个进程发送一个数据块的方法。  
每个数据块都被认为含有一个类型，接收进程可以独立地接收含有不同类型的数据结构。
我们可以通过发送消息来避免命名管道的同步和阻塞问题。


消息队列与信号管道的对比
---------------------------

**消息队列与信号的对比：**

-   信号承载的信息量少，而消息队列可以承载大量自定义的数据。

**消息队列与管道的对比：**

-   消息队列跟命名管道有不少的相同之处，它与命名管道一样，消息队列进行通信的进程可以是不相关的进程，
    同时它们都是通过发送和接收的方式来传递数据的。在命名管道中，发送数据用write()，接收数据用read()，
    则在消息队列中，发送数据用msgsnd()，接收数据用msgrcv()，消息队列对每个数据都有一个最大长度的限制。

-   消息队列也可以独立于发送和接收进程而存在，在进程终止时，消息队列及其内容并不会被删除。

-   管道只能承载无格式字节流，消息队列提供有格式的字节流，可以减少了开发人员的工作量。

-   消息队列是面向记录的，其中的消息具有特定的格式以及特定的优先级，接收程序可以通过消息类型有选择地接收数据，
    而不是像命名管道中那样，只能默认地接收。

-   消息队列可以实现消息的随机查询，消息不一定要以先进先出的顺序接收，也可以按消息的类型接收。

消息队列的实现包括创建或打开消息队列、发送消息、接收消息和控制消息队列这4 种操作。


消息队列函数说明
--------------------

Linux内核提供了一系列函数来使用消息队列：

-   其中创建或打开消息队列使用的函数是msgget()，这里创建的消息队列的数量会受到系统可支持的消息队列数量的限制；
-   发送消息使用的函数是msgsnd()函数，它把消息发送到已打开的消息队列末尾;
-   接收消息使用的函数是msgrcv()，它把消息从消息队列中取走，与FIFO 不同的是，这里可以指定取走某一条消息;
-   最后控制消息队列使用的函数是msgctl()，它可以完成多项功能。



msgget()获取函数
~~~~~~~~~~~~~~~~~~~~~~~
收发消息前需要具体的消息队列对象，msgget()函数的作用是创建或获取一个消息队列对象，
并返回消息队列标识符。函数原型如下：

.. code:: c

       int msgget(key_t key, int msgflg);

若执行成功返回队列ID，失败返回-1。
它的两个输入参数说明如下：

-   key：消息队列的关键字值，多个进程可以通过它访问同一个消息队列。
    例如收发进程都使用同一个键值即可使用同一个消息队列进行通讯。
    其中有个特殊值IPC_PRIVATE，它用于创建当前进程的私有消息队列。

-   msgflg：表示创建的消息队列的模式标志参数，主要有IPC_CREAT，IPC_EXCL和权限mode，

        -   如果是 ``IPC_CREAT`` 为真表示：如果内核中不存在关键字与key相等的消息队列，则新建一个消息队列；
            如果存在这样的消息队列，返回此消息队列的标识符。
        -   而如果为 ``IPC_CREAT | IPC_EXCL`` 表示如果内核中不存在键值与key相等的消息队列，则新建一个消息队列；
            如果存在这样的消息队列则报错。
        -   mode指IPC对象存取权限，它使用Linux文件的数字权限表示方式，如0600，0666等。
            
    这些参数是可以通过“｜”运算符联合起来的，因为它始终是int类型的参数。如msgflag使用参数 ``IPC_CREAT | 0666`` 时表示，
    创建或返回已经存在的消息队列的标识符，且该消息队列的存取权限为0666，
    即消息的所有者，所属组用户，其他用户均可对该消息进行读写。

.. attention:: 

    -   选项 msgflg 是一个位掩码，因此 IPC_CREAT、IPC_EXCL 和权限 mode 可以用位或的方式叠加起来，
        比如: ``msgget(key, IPC_CREAT | 0666);`` 表示如果 key 对应的消息队列不存在就创建，
        且权限指定为 0666，若已存在则直接获取消息队列ID，此处的0666使用的是Linux文件权限的数字表示方式。
    -   权限只有读和写，执行权限是无效的，例如 0777 跟 0666 是等价的。
    -   当 key 被指定为 IPC_PRIVATE 时，系统会自动产生一个未用的 key 来对应一个新的消息队列对象，
        这个消息队列一般用于进程内部间的通信。

-   该函数可能返回以下错误代码：

    -  EACCES：指定的消息队列已存在，但调用进程没有权限访问它

    -  EEXIST：key指定的消息队列已存在，而msgflg中同时指定IPC_CREAT和IPC_EXCL标志

    -  ENOENT：key指定的消息队列不存在同时msgflg中没有指定IPC_CREAT标志

    -  ENOMEM：需要建立消息队列，但内存不足

    -  ENOSPC：需要建立消息队列，但已达到系统的限制



发送消息与接收消息
------------------

msgsnd()发送函数
~~~~~~~~~~~~~~~~~~~~~~~

这个函数的主要作用就是将消息写入到消息队列，俗称发送一个消息。函数原型如下：

.. code:: c

        int msgsnd(int msqid, const void *msgp, size_t msgsz, int msgflg);

参数说明：

-   msqid：消息队列标识符。

-   msgp：发送给队列的消息。msgp可以是任何类型的结构体，但第一个字段必须为long类型，
    即表明此发送消息的类型，msgrcv()函数则根据此接收消息。msgp定义的参照格式如下：

    .. code:: c

            /*msgp定义的参照格式*/
            struct s_msg{ 
                long type;  /* 必须大于0,消息类型 */
                char mtext[１];  /* 消息正文，可以是其他任何类型 */
            } msgp;

    -   msgsz：要发送消息的大小，不包含消息类型占用的4个字节，即mtext的长度。

    -   msgflg：如果为0则表示：当消息队列满时，msgsnd()函数将会阻塞，直到消息能写进消息队列；
        如果为IPC_NOWAIT则表示：当消息队列已满的时候，msgsnd()函数不等待立即返回；
        如果为IPC_NOERROR：若发送的消息大于size字节，则把该消息截断，截断部分将被丢弃，且不通知发送进程。

-   返回值：如果成功则返回0，如果失败则返回-1，并且错误原因存于error中。错误代码：

    -  EAGAIN：参数msgflg设为IPC_NOWAIT，而消息队列已满。

    -  EIDRM：标识符为msqid的消息队列已被删除。

    -  EACCESS：无权限写入消息队列。

    -  EFAULT：参数msgp指向无效的内存地址。

    -  EINTR：队列已满而处于等待情况下被信号中断。

    -  EINVAL：无效的参数msqid、msgsz或参数消息类型type小于0。

msgsnd()为阻塞函数，当消息队列容量满或消息个数满会阻塞。若消息队列已被删除，则返回EIDRM错误；
若被信号中断返回E_INTR错误。

如果设置IPC_NOWAIT消息队列满或个数满时会返回-1，并且置EAGAIN错误。

msgsnd()解除阻塞的条件有以下三个条件：

-   消息队列中有容纳该消息的空间。
-   msqid代表的消息队列被删除。
-   调用msgsnd函数的进程被信号中断。

msgrcv()接收函数
~~~~~~~~~~~~~~~~~~~~~~~
msgrcv()函数是从标识符为msqid的消息队列读取消息并将消息存储到msgp中，
读取后把此消息从消息队列中删除，也就是俗话说的接收消息。函数原型：

.. code:: c

        ssize_t msgrcv(int msqid, void *msgp, size_t msgsz, long msgtyp, int msgflg);


参数说明：

-   msqid：消息队列标识符。

-   msgp：存放消息的结构体，结构体类型要与msgsnd()函数发送的类型相同。

-   msgsz：要接收消息的大小，不包含消息类型占用的4个字节。

-   msgtyp有多个可选的值：如果为0则表示接收第一个消息，如果大于0则表示接收类型等于msgtyp的第一个消息，
    而如果小于0则表示接收类型等于或者小于msgtyp绝对值的第一个消息。

-   msgflg用于设置接收的处理方式，取值情况如下：

    -  0: 阻塞式接收消息，没有该类型的消息msgrcv函数一直阻塞等待

    -  IPC_NOWAIT：若在消息队列中并没有相应类型的消息可以接收，则函数立即返回，此时错误码为ENOMSG

    -  IPC_EXCEPT：与msgtype配合使用返回队列中第一个类型不为msgtype的消息

    -  IPC_NOERROR：如果队列中满足条件的消息内容大于所请求的size字节，则把该消息截断，截断部分将被丢弃

-   返回值：msgrcv()函数如果接收消息成功则返回实际读取到的消息数据长度，否则返回-1，错误原因存于error中。错误代码：

    -  E2BIG：消息数据长度大于msgsz而msgflag没有设置IPC_NOERROR

    -  EIDRM：标识符为msqid的消息队列已被删除

    -  EACCESS：无权限读取该消息队列

    -  EFAULT：参数msgp指向无效的内存地址

    -  ENOMSG：参数msgflg设为IPC_NOWAIT，而消息队列中无消息可读

    -  EINTR：等待读取队列内的消息情况下被信号中断

msgrcv()函数解除阻塞的条件也有三个：

-   消息队列中有了满足条件的消息。
-   msqid代表的消息队列被删除。
-   调用msgrcv()函数的进程被信号中断。

msgctl()操作消息队列
~~~~~~~~~~~~~~~~~~~~~~~

消息队列是可以被用户操作的，比如设置或者获取消息队列的相关属性，那么可以通过msgctl()函数去处理它。函数原型：

.. code:: c

    int msgctl(int msqid, int cmd, struct msqid_ds *buf);


参数说明：

-   msqid：消息队列标识符。
-   cmd 用于设置使用什么操作命令，它的取值有多个：

    -   IPC_STAT 获取该 MSG 的信息，获取到的信息会储存在结构体 msqid_ds类型的buf中。

    -   IPC_SET 设置消息队列的属性，要设置的属性需先存储在结构体msqid_ds类型的buf中，
        可设置的属性包括：msg_perm.uid、msg_perm.gid、msg_perm.mode以及msg_qbytes，储存在结构体msqid_ds中。

    -   IPC_RMID 立即删除该 MSG，并且唤醒所有阻塞在该 MSG上的进程，同时忽略第三个参数。

    -   IPC_INFO 获得关于当前系统中 MSG 的限制值信息。

    -   MSG_INFO 获得关于当前系统中 MSG 的相关资源消耗信息。

    -   MSG_STAT 同 IPC_STAT，但 msgid为该消息队列在内核中记录所有消息队列信息的数组的下标，
        因此通过迭代所有的下标可以获得系统中所有消息队列的相关信息。

-   buf：相关信息结构体缓冲区。

    -   返回值：

    -  成功：0

    -  出错：-1，错误原因存于error中，错误代码：

        -  EACCESS：参数cmd为IPC_STAT，确无权限读取该消息队列。

        -  EFAULT：参数buf指向无效的内存地址。

        -  EIDRM：标识符为msqid的消息队列已被删除。

        -  EINVAL：无效的参数cmd或msqid。

        -  EPERM：参数cmd为IPC_SET或IPC_RMID，却无足够的权限执行。

消息队列示例
------------

接下来通过示例来讲解消息队列的使用，使用方法一般是:

发送者:

1.  获取消息队列的 ID
#.  将数据放入一个附带有标识的特殊的结构体，发送给消息队列。

接收者:

1.  获取消息队列的 ID
#.  将指定标识的消息读出。

当发送者和接收者都不再使用消息队列时，及时删除它以释放系统资源。

本次实验主要是两个进程（无血缘关系的进程）通过消息队列进行消息的传递，
一个进程发送消息，一个进程接收消息，并将其打印出来。

发送进程
~~~~~~~~~~~~~~~~~~~~~~~~

本示例的发送进程代码如下：

.. code-block:: c
    :caption: 消息队列发送进程（base_code/system_programing/msg/msg_send/sources/msg.c文件）
    :emphasize-lines: 22,41,51
    :linenos:

    #include <sys/types.h>
    #include <sys/ipc.h>
    #include <sys/msg.h>
    #include <stdio.h>
    #include <stdlib.h>
    #include <unistd.h>
    #include <string.h>


    #define BUFFER_SIZE 512

    struct message
    {
        long msg_type;
        char msg_text[BUFFER_SIZE];
    };
    int main()
    {
        int qid;
        struct message msg;

        /*创建消息队列*/
        if ((qid = msgget((key_t)1234, IPC_CREAT|0666)) == -1)
        {
            perror("msgget\n");
            exit(1);
        }

        printf("Open queue %d\n",qid);

        while(1)
        {
            printf("Enter some message to the queue:");
            if ((fgets(msg.msg_text, BUFFER_SIZE, stdin)) == NULL)
            {
                printf("\nGet message end.\n");
                exit(1);
            }  

            msg.msg_type = getpid();
            /*添加消息到消息队列*/
            if ((msgsnd(qid, &msg, strlen(msg.msg_text), 0)) < 0)
            {
                perror("\nSend message error.\n");
                exit(1);
            }
            else
            {
                printf("Send message.\n");
            }

            if (strncmp(msg.msg_text, "quit", 4) == 0)
            {
                printf("\nQuit get message.\n");
                break;
            }
        }

        exit(0);
    }

本代码重点说明如下：

-   第22行，调用msgget()函数创建/获取了一个key值为1234的消息队列，该队列的属性“0666”表示任何人都可读写，
    创建/获取到的队列ID存储在变量qid中。
-   第47行，调用msgsndb()函数把进程号以及前面用户输入的字符串，通过msg结构体添加到前面得到的qid队列中。
-   第51行，若用户发送的消息为quit，那么退出循环结束进程。

接收进程
~~~~~~~~~~~~~~~~~
接收进程示例如下：

.. code-block:: c
    :caption: 消息队列接收进程（base_code/system_programing/msg/msg_recv/sources/msg.c文件）
    :emphasize-lines: 23,36,47
    :linenos:

    #include <sys/types.h>
    #include <sys/ipc.h>
    #include <sys/msg.h>
    #include <stdio.h>
    #include <stdlib.h>
    #include <unistd.h>
    #include <string.h>

    #define BUFFER_SIZE 512

    struct message
    {
        long msg_type;
        char msg_text[BUFFER_SIZE];
    };

    int main()
    {
        int qid;
        struct message msg;

        /*创建消息队列*/
        if ((qid = msgget((key_t)1234, IPC_CREAT|0666)) == -1)
        {
            perror("msgget");
            exit(1);
        }

        printf("Open queue %d\n", qid);

        do
        {
            /*读取消息队列*/
            memset(msg.msg_text, 0, BUFFER_SIZE);

            if (msgrcv(qid, (void*)&msg, BUFFER_SIZE, 0, 0) < 0)
            {
                perror("msgrcv");
                exit(1);
            }

            printf("The message from process %ld : %s", msg.msg_type, msg.msg_text);

        } while(strncmp(msg.msg_text, "quit", 4));

        /*从系统内核中删除消息队列 */
        if ((msgctl(qid, IPC_RMID, NULL)) < 0)
        {
            perror("msgctl");
            exit(1);
        }
        else
        {
            printf("Delete msg qid: %d.\n", qid);
        }

        exit(0);

    }

本代码重点说明如下：

-   第23行，调用msgget()函数创建/获取队列qid。可以注意到，此处跟发送进程是完全一样的，无论哪个进程先运行，
    若key值为1234的队列不存在则创建，把以实验时两个进程并没有先后启动顺序的要求。

-   第36行，在循环中调用msgrcv()函数接收qid队列的msg结构体消息，此处使用阻塞方式接收，
    若队列中没有消息，会停留在本行代码等待。

-   第47行，若前面接收到用户的消息为quit，会退出循环，在本行代码调用msgctl()删除消息队列并退出本进程。


编译及测试
~~~~~~~~~~~~~~~~~
示例代码分别位于base_code/system_programing/msg/的msg_send及msg_recv目录下，
将两个进程编译出来，分别运行即可，实验现象如下：

发送进程
^^^^^^^^^^^^^^^^^^^^^

在发送消息进程运行的时候，会提示让你输入要发送的消息，随便什么消息都可以的，使用回车完成消息的输入。
输入quit或使用Ctrl+D、Ctrl+C可结束进程。

.. code:: bash

    # 以下操作在 system_programing/msg/msg_send 代码目录进行
    # 编译X86版本程序发送进程
    make
    # 运行X86版本程序发送进程
    ./build_x86/msg_send_demo 
    
    # 输入消息测试，
    Open queue 98345
    Enter some message to the queue:embedfire
    Send message.
    Enter some message to the queue:test
    Send message.
    Enter some message to the queue:hello world
    Send message.
    # 发送quit消息并结束进程
    Enter some message to the queue:quit
    Send message.

    Quit get message.

查看消息队列
^^^^^^^^^^^^^^^^^^^^^

可以通过 ipcs -q 命令来查看系统中存在的消息队列，若以上队列没有关闭，它的查看结果如下：

.. code:: bash

    # 查询系统当前存在的队列
    ipcs -q

    # 以下为输出：
    --------- 消息队列 -----------
    键        msqid      拥有者  权限     已用字节数 消息      
    0x000004d2 98345      flyleaf    666        0            0  

    # 可查看到key键值 0x04d2(1234)，qid 98345 与进程中创建的一致。


接收进程
^^^^^^^^^^^^^^^^^^^^^

打开一个新终端，编译及运行接收消息进程，当你从发送消息进程输入消息时（按下回车键发送），
接收消息进程会打印出你输入的消息，若无消息则接收进程会阻塞等待，接收到quit消息会退出进程。


.. code:: bash

    # 以下操作在 system_programing/msg/msg_recv 代码目录进行
    # 编译X86版本程序发送进程
    make
    # 运行X86版本程序发送进程
    ./build_x86/msg_recv_demo 
    
    # 接收到的消息
    Open queue 98345
    The message from process 21023 : embedfire
    The message from process 21023 : test
    The message from process 21023 : hello world
    The message from process 21023 : quit
    Delete msg qid: 98345.

.. tip:: 
    在本例子中，若发送进程不是通过quit消息退出（如Ctrl+C或Ctrl+D），则不会触发接收进程主动删除消息队列，
    在这种情况下可通过 ``ipcs -q`` 命令查看到该消息队列依然存在，通过 ``ipcrm -q [消息队列qid]`` 即可删除。

