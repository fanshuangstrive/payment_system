







> http://www.xzbu.com/8/view-1699765.htm

摘 要：在银联交易中，各个入网机构与CUPS（中国银联信息处理中心系统）之间采用长连接的方式，即socket连接建立后，除非发生异常导致连接中断，否则不再关闭连接。为了保证通讯的稳定和高效，在银联技术规范2.0中引入了“组”的概念，指对多条连接进行分组，当同组内有一条连接中断时，要关闭组内的所有连接，并重建这些连接。描述了入网机构金卡前置机以组为单位的连接形式的设计与实现，重点对管理模块、发送模块和接收模块等部分进行了描述。 
中国论文网 http://www.xzbu.com/8/view-1699765.htm
　　� 
　　关键词：TCP/IP通信;长连接;连接组� 
　　中图分类号：TP319 文献标识码：A 文章编号：1672-7800（2012）003-0087-03�� 
　　� 
　　作者简介：李素若(1969-),男,湖北荆门人，武汉理工大学硕士研究生，荆楚理工学院计算机工程学院副教授,研究方向为软件工程和数据库。 
　　0 引言� 
　　TCP/IP长连接通信是指 Client 方与 Server 方先建立通讯连接，连接建立后不会断开，然后再进行报文发送和接收，报文发送与接收完毕后，原来的连接不会断开而继续存在，因此可以连续进行交易报文的发送与接收。在实际应用中TCP/IP长连接主要面临的问题是：由于只需建立一次连接，如果通信过程中连接断开而发送方使用write()或send()函数发送数据时并不马上报错，直到通信缓冲区阻塞溢出或TCP/IP资源耗尽导致系统报错。目前解决这个问题主要有两种方法，一种方法是在client端启动一个进程在一定间隔时间内向Server端发ping命令测试线路是否保持连接状态，如果Client端数量很大，大量发出ping包可能导致Server端通信堵塞；另一方法是在联机交易空闲时间间隔内Client端向Server端发送测试包（空包），在规定时间内能接受到返回包来测试线路。� 
　　在银联交易中，基于长连接中存在的问题，中国银联在《银行卡联网联合技术规范V2.0》中对通信接口规范作了相关规定：� 
　　（1）对于入网机构和处理中心之间的多条通信连接，增加“连接组”的概念。“连接组”的含义是指对多条通信连接进行分组，同组的一条连接中断时，要关闭同组内的其它连接并重建该组的所有连接。� 
　　（2）针对单工连接，该规范规定了3种情况。� 
　　第一种情况：在一个连接组中，如果入网机构发现它的发送连接中断，则关闭同组内的其它连接，并向处理中心发起连接请求，重建该组内的所有连接。 � 
　　第二种情况：在一个连接组中，如果入网机构检测到该连接已被中断或在5分钟之内在某接收连接上没有收到任何报文（包括空闲连接查询报文），则认为该连接已经中断，将关闭在本连接组内的所有连接，并向处理中心发起连接请求，重建该组内的所有连接。 � 
　　第三种情况：在一个连接组中，对于任意一个连接对，如果处理中心发现该连接的发送连接中断，而其相应的接收连接正常时，则处理中心首先关闭该接收连接，然后回复到侦听状态，等待接收入网机构重新建立连接的请求。 � 
　　1 设计图� 
　　根据银联通信接口规范，入网机构前置机采用TCP/IP长连接的单工方式，采用2进2出4条连接，具体连接见图1所示。� 
　　2 详细设计� 
　　2.1 manager设计� 
　　manager功能模块被设计管理同一“连接组”内的所有连接。具体的功能如下：①等待接收来自sender和receiver发来的状态消息（一般指通讯故障或者超时）；②发送消息通知sender跟CUPS建立连接（或重新建立连接）；③发送消息通知sender跟CUPS断开连接（关闭连接）；④发送消息通知receiver在特定端口诊听来自CUPS的连接请求；⑤发送消息通知receiver在特定端口断开与CUPS的连接，并继续诊听下一个连接。� 
　　manager处理sender和receiver发来的消息的方式如下：� 
　　（1）如果是sender发来的消息（一般指连接中断消息），说明发送连接中断，则manager应该关闭它管理的所有连接，并重建这些连接。� 
　　（2）如果是receiver发来的消息（包括连接中断和超时消息），说明接收连接中断，manager关闭它所管理的所有连接，并重建这些连接。� 
　　图3 manager流程� 
　　2.2 sender设计� 
　　sender表示一个用来发送数据的进程，它建立一个与服务器的socket连接，并一直保持该连接。manager根据配置文件中的设置，启动一个或多个sender进程，由它们共同负责发送数据。当连接中断时，它们会向manager发送一个连接中断消息，由manager进行处理。一个sender进程的工作流程按照以下步骤进行：①设置socket的“LINGER”属性为关闭状态，设置“REUSEADDR”属性为打开状态；②与服务器建立连接，并一直保持连接状态；③设置计时器，如果连接上4分钟（可配置）未发送数据，则发送“空闲连接查询报文”。发送完成后，将计时器重置；④当有发送数据的消息到来时，从消息队列中取出待发送的数据，通过已建立的连接进行发送。发送完成后，重置计时器；⑤当发送信息失败或发现连接已断开的时候，向manager发送连接中断消息；⑥当收到manager重建连接的消息时，关闭当前连接并重新建立；⑦当收到manager关闭连接的消息时，关闭当前连接并退出进程。� 
　　图4 sender流程� 
　　2.3 receiver设计� 
　　receiver表示一个接收数据的进程，它首先进行端口的监听，当有连接请求，并且是合法连接时就建立连接。receiver是由manager进行管理的，manager根据配置文件中的设置管理一个或多个receiver。每个receiver都保持着一个长连接，当连接中断时会向manager发送一个中断消息。receiver的工作流程为：①设置socket的“LINGER”属性为关闭状态，设置“REUSEADDR”属性为打开状态;②根据参数，建立socket并监听端口;③当有连接请求时，首先检查是否为合法地址，如果非法则继续监听；如果合法，则建立连接并一直保持;④启动计时器，如果5分钟（可配置）内没有接收到任何数据，则认为连接中断，向manager发送连接中断消息;⑤当有数据到达时，接收数据并将其发送到消息队列中。接收完成后，重置计时器;⑥当收到manager发送的重建消息时，关闭计时器，关闭连接并重新监听;⑦当收到manager发送的关闭消息时，关闭连接并退出进程。� 
　　2.4 通信设计� 
　　manager和sender、receiver之间通讯是同一台主机之间的不同进程之间的通讯，通讯的内容如下：� 
　　（1）manager向sender发送的消息有： 关闭连接、 重建连接� 
　　（2）manager向receiver发送的消息有： 关闭连接、 重建连接� 
　　（3）sender向manager发送的消息有： 发送连接关闭。� 
　　（4）receiver向manager发送的消息有： 接收连接关闭、 接收连接超时。� 
　　3 结束语� 
　　本文针对TCP/IP中长连接单工方式中连接中断后重新连接的问题，采用了类似守护进程的原理，使用统一的管理模块manger负责发送长连接sender和接收长连接receiver的管理。manger主要功能是发送消息通知sender跟cups建立连接或断开连接，发送消息通知receiver在特定端口侦听来自CUPS的连接请求，或断开连接并继续诊听下一个连接。manger之所以发出断开连接的消息是因为它收到“连接组”中sender或receiver发来的通讯故障或超时的状态消息。在具体连接中如果sender在规定时间内没有交易发生，则发送空闲连接查询报文，如果在规定时间内receiver没有收到响应报文来判断通讯故障或超时，向manger发送状态消息。 

　　参考文献：� 

　　\[1\] 中国银联股份有限公司.Q/CUP 006.1-2005，银行卡联网联合技术规范V2.0\[S\].上海：中国银联股份有限公司，2005.� 

　　\[2\] 刘光宝.TCP/IP应用程序的通信连接模式.北京：developerWork中国，AIX and UNIX，文档库\[EB/OL\].http://www.省略/developerworks/cn/aix/library/0807_liugb_tcpip/#author.2008.7.� 

　　\[3\] W.RICHARD STEVENS.TCP/IP详解\[M\].范建华,译.北京：机械工业出版社，2000.� 

　　\[4\] 范建华,胥光辉,张涛泽.TCP/ IP 详解(卷1：协议)\[M\].北京：机械工业出版社，2000.� 

　　（责任编辑：杜能钢） 

　　 

　　�� 

　　 Design of the CUPS Based on TCP/IP Communication� Module of The-long-Connect the 

　　�� 

　　Abstract:CUP transactions in various network institutions and CUPS (Information Processing Center, China UnionPay system) using a long connection between the way that the socket connection is established, unless an exception causes the connection to break, or is no longer close the connection. In order to ensure stable and efficient communication in CUP 2.0 specification introduced a "group" concept, refers to the number of connections to be grouped within the same group when there is a connection is interrupted, turn off all connections within the group, and reconstruction these connections. This paper describes the front-end network agency Gold group units connected to form the design and implementation, focusing on the management module, sending module and receiver module and some other description. 

　　� 

　　Key Words: TCP/IP；Communication The-Long-Connection;Connection-Group 