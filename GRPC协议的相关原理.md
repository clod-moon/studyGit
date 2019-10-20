# GRPC协议的相关原理

​      GRPC的Client与Server，均通过Netty Channel作为数据通信，序列化、反序列化则使用Protobuf，每个请求都将被封装成HTTP2的Stream，在整个生命周期中，客户端Channel应该保持长连接，而不是每次调用重新创建Channel、响应结束后关闭Channel（即短连接、交互式的RPC），目的就是达到链接的复用，进而提高交互效率。

​    **1、Server端**

​    我们通常使用NettyServerBuilder，即IO处理模型基于Netty，将来可能会支持其他的IO模型。Netty Server的IO模型简析：

​    1）创建ServerBootstrap，设定BossGroup与workerGroup线程池

​    2）注册childHandler，用来处理客户端链接中的请求成帧

​    3）bind到指定的port，即内部初始化ServerSocketChannel等，开始侦听和接受客户端链接。

​    4）BossGroup中的线程用于accept客户端链接，并转发（轮训）给workerGroup中的线程。

​    5）workerGroup中的特定线程用于初始化客户端链接，初始化pipeline和handler，并将其注册到worker线程的selector上（每个worker线程持有一个selector，不共享）

​    6）selector上发生读写事件后，获取事件所属的链接句柄，然后执行handler（inbound），同时进行拆封package，handler执行完毕后，数据写入通过，由outbound handler处理（封包）通过链接发出。    注意每个worker线程上的数据请求是队列化的。

​    GRPC而言，只是对Netty Server的简单封装，底层使用了PlaintextHandler、Http2ConnectionHandler的相关封装等。具体Framer、Stream方式请参考Http2相关文档。

​    1）bossEventLoopGroup：如果没指定，默认为一个static共享的对象，即JVM内所有的NettyServer都使用同一个Group，默认线程池大小为1。

​    2）workerEventLoopGroup：如果没指定，默认为一个static共享的对象，线程池大小为coreSize * 2。这两个对象采用默认值并不会带来问题；通常情况下，即使你的application中有多个GRPC Server，默认值也一样能够带来收益。不合适的线程池大小，有可能会是性能受限。

​    3）channelType：默认为NioServerSocketChannel，通常我们采用默认值；当然你也可以开发自己的类。如果此值为NioServerSocketChannel，则开启keepalive，同时设定SO_BACKLOG为128；BACKLOG就是系统底层已经建立引入链接但是尚未被accept的Socket队列的大小，在链接密集型（特别是短连接）时，如果队列超过此值，新的创建链接请求将会被拒绝（有可能你在压力测试时，会遇到这样的问题），keepalive和BACKLOG特性目前无法直接修改。

​    对于Server端，我们需要关注上述keepalive的一些设置；如果Netty Client在空闲一段时间后，Server端会主动关闭链接，有可能Client仍然保持链接的句柄，将会导致RPC调用时发生异常。这也会导致GRPC客户端调用时偶尔发生错误的原因之一。

​    4）followControlWindow：流量控制的窗口大小，单位：字节，默认值为1M，HTTP2中的“Flow Control”特性；连接上，已经发送尚未ACK的数据帧大小，比如window大小为100K，且winow已满，每次向Client发送消息时，如果客户端反馈ACK（携带此次ACK数据的大小），window将会减掉此大小；每次向window中添加亟待发送的数据时，window增加；如果window中的数据已达到限定值，它将不能继续添加数据，只能等待Client端ACK。

​    5）maxConcurrentCallPerConnection：每个connection允许的最大并发请求数，默认值为Integer.MAX_VALUE;如果此连接上已经接受但尚未响应的streams个数达到此值，新的请求将会被拒绝。为了避免TCP通道的过度拥堵，我们可以适度调整此值，以便Server端平稳处理，毕竟buffer太多的streams会对server的内存造成巨大压力。

​    6）maxMessageSize：每次调用允许发送的最大数据量，默认为100M。

​    7）maxHeaderListSize：每次调用允许发送的header的最大条数，GRPC中默认为8192。

​    对于其他的比如SSL/TSL等，可以参考其他文档。

​    GRPC Server端，还有一个最终要的方法：addService。【如下文service代理模式】

​    在此之前，我们需要介绍一下bindService方法，每个GRPC生成的service代码中都有此方法，它以硬编码的方式遍历此service的方法列表，将每个方法的调用过程都与“被代理实例”绑定，这个模式有点类似于静态代理，比如调用sayHello方法时，其实内部直接调用“被代理实例”的sayHello方法（参见MethodHandler.invoke方法，每个方法都有一个唯一的index，通过硬编码方式执行）；bindService方法的最终目的是创建一个ServerServiceDefinition对象，这个对象内部位置一个map，key为此Service的方法的全名（fullname，{package}.{service}.{method}）,value就是此方法的GRPC封装类（ServerMethodDefinition）。

​    addService方法可以添加多个Service，即一个Netty Server可以为多个service服务，这并不违背设计模式和架构模式。addService方法将会把service保存在内部的一个map中，key为serviceName（即{package}.{service}）,value就是上述bindService生成的对象。

​    那么究竟Server端是如何解析RPC过程的？Client在调用时会将调用的service名称 + method信息保存在一个GRPC“保留”的header中，那么Server端即可通过获取这个特定的header信息，就可以得知此stream需要请求的service、以及其method，那么接下来只需要从上述提到的map中找到service，然后找到此method，直接代理调用即可。执行结果在Encoder之后发送给Client。（参见：NettyServerHandler）

​    因为是map存储，所以我们需要在定义.proto文件时，尽可能的指定package信息，以避免因为service过多导致名称可能重复的问题。

​    **2、Client端**

​    我们使用ManagedChannelBuilder来创建客户端channel，ManagedChannelBuilder使用了provider机制，具体是创建了哪种channel有provider决定，可以参看META-INF下同类名的文件中的注册信息。当前Channel有2种：NettyChannelBuilder与OkHttpChannelBuilder。本人的当前版本中为NettyChannelBuilder；我们可以直接使用NettyChannelBuilder来构建channel。如下描述则针对NettyChannelBuilder：

​    配置参数与NettyServerBuilder基本类似，再次不再赘言。默认情况下，Client端默认的eventLoopGroup线程池也是static的，全局共享的，默认线程个数为coreSize * 2。合理的线程池个数可以提高客户端的吞吐能力。

​    ManagedChannel是客户端最核心的类，它表示逻辑上的一个channel；底层持有一个物理的transport（TCP通道，参见NettyClientTransport），并负责维护此transport的活性；即在RPC调用的任何时机，如果检测到底层transport处于关闭状态（terminated），将会尝试重建transport。（参见TransportSet.obtainActiveTransport()）

​    通常情况下，我们不需要在RPC调用结束后就关闭Channel，Channel可以被一直重用，直到Client不再需要请求位置或者Channel无法真的异常中断而无法继续使用。当然，为了提高Client端application的整体并发能力，我们可以使用连接池模式，即创建多个ManagedChannel，然后使用轮训、随机等算法，在每次RPC请求时选择一个Channel即可。（备注，连接池特性，目前GRPC尚未提供，需要额外的开发）

​    每个Service客户端，都生成了2种stub：BlockingStub和FutureStub；这两个Stub内部调用过程几乎一样，唯一不同的是BlockingStub的方法直接返回Response Model，而FutureStub返回一个Future对象。BlockingStub内部也是基于Future机制，只是封装了阻塞等待的过程：

​    创建一个Stub的成本是非常低的，我们可以在每次请求时都通过channel创建新的stub，这并不会带来任何问题（只不过是创建了大量对象）；其实更好的方式是，我们应该使用一个Stub发送多次请求，即Stub也是可以重用的；直到Stub上的状态异常而无法使用。最常见的异常，就是“io.grpc.StatusRuntimeException: DEADLINE_EXCEEDED”，即表示DEADLINE时间过期，我们可以为每个Stub配置deadline时间，那么如果此stub被使用的时长超过此值（不是空闲的时间），将不能再发送请求，此时我们应该创建新的Stub。很多人想尽办法来使用“withDeadlineAfter”方法来实现一些奇怪的事情，此参数的主要目的就是表明：此stub只能被使用X时长，此后将不能再进行请求，应该被释放。所以，它并不能实现类似于“keepAlive”的语义，即使我们需要keepAlive，也应该在Channel级别，而不是在一个Stub上。

​    如果你使用了连接池，那么其实连接池不应该关注DEADLINE的错误，只要Channel本身没有terminated即可；就把这个问题交给调用者处理。如果你也对Stub使用了对象池，那么你就可能需要关注这个情况了，你不应该向调用者返回一个“DEADLINE”的stub，或者如果调用者发现了DEADLINE，你的对象池应该能够移除它。

​    1）实例化ManagedChannel，此channel可以被任意多个Stub实例引用；如上文说述，我们可以通过创建Channel池，来提高application整体的吞吐能力。此Channel实例，不应该被shutdown，直到Client端停止服务；在任何时候，特别是创建Stub时，我们应该判定Channel的状态。

  因为Channel是可以多路复用，所以我们用Pool机制（比如commons-pool）也可以实现连接池，只是这种池并非完全符合GRPC/HTTP2的设计语义，因为GRPC允许一个Channel上连续发送对个Requests（然后一次性接收多个Responses），而不是“交互式”的Request-Response模式，当然这么使用并不会有任何问题。

​    2）对于批量调用的场景，我们可以使用FutureStub，对于普通的业务类型RPC，我们应该使用BlockingStub。

​    3）每个RPC方法的调用，比如sayHello，调用开始后，将会为每个调用请求创建一个ClientCall实例，其内部封装了调用的方法、配置选项（headers）等。此后将会创建Stream对象，每个Stream都持有唯一的streamId，它是Transport用于分拣Response的凭证。最终调用的所有参数都会被封装在Stream中。

​    4）检测DEADLINE，是否已经过期，如果过期，将使用FailingClientStream对象来模拟整个RPC过程，当然请求不会通过通道发出，直接经过异常流处理过程。

​    5）然后获取transport，如果此时检测到transport已经中断，则重建transport。（自动重练机制，ClientCallImpl.start()方法）

​    6）发送请求参数，即我们Request实例。一次RPC调用，数据是分多次发送，但是ClientCall在创建时已经绑定到了指定的线程上，所以数据发送总是通过一个线程进行（不会乱序）。

​    7）将ClientCall实例置为halfClose，即半关闭，并不是将底层Channel或者Transport半关闭，只是逻辑上限定此ClientCall实例上将不能继续发送任何stream信息，而是等待Response。

​    8）Netty底层IO将会对reponse数据流进行解包（Http2ConnectionDecoder）,并根据streamId分拣Response，同时唤醒响应的ClientCalls阻塞。（参见ClientCalls，GrpcFuture）

​    9）如果是BlockingStub，则请求返回，如果响应中包含应用异常，则封装后抛出；如果是网络异常，则可能触发Channel重建、Stream重置等。