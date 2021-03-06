Nagle算法是以他的发明人John Nagle的名字命名的，它用于自动连接许多的小缓冲器消息；这一过程（称为nagling）通过减少必须发送包的个数来增加网络软件系统的效率。

Nagle算法于1984年定义为福特航空和通信公司IP/TCP拥塞控制方法，这使福特经营的最早的专用TCP/IP网络减少拥塞控制，从那以后这一方法得到了广泛应用。Nagle的文档里定义了处理他所谓的小包问题的方法，这种问题指的是应用程序一次产生一字节数据，这样会导致网络由于太多的包而过载（一个常见的情况是发送端的"糊涂窗口综合症(Silly Window Syndrome)"）。从键盘输入的一个字符，占用一个字节，可能在传输上造成41字节的包，其中包括1字节的有用信息和40字节的首部数据。这种情况转变成了4000%的消耗，这样的情况对于轻负载的网络来说还是可以接受的，但是重负载的福特网络就受不了了，它没有必要在经过节点和网关的时候重发，导致包丢失和妨碍传输速度。吞吐量可能会妨碍甚至在一定程度上会导致连接失败。Nagle的算法通常会在TCP程序里添加两行代码，在未确认数据发送的时候让发送器把数据送到缓存里。任何数据随后继续直到得到明显的数据确认或者直到攒到了一定数量的数据了再发包。尽管Nagle的算法解决的问题只是局限于福特网络，然而同样的问题也可能出现在ARPANet。这种方法在包括因特网在内的整个网络里得到了推广，成为了默认的执行方式，尽管在高互动环境下有些时候是不必要的，例如在客户/服务器情形下。在这种情况下，nagling可以通过使用TCP_NODELAY 套接字选项关闭。

## 算法规则

1. <u>如果包长度达到MSS，则允许发送</u>
2. <u>如果包含FIN，则允许发送</u>
3. <u>如果设置了TCP_NODELAY，则允许发送</u>
4. <u>未设置TCP_CORK选项时，若所有发出去的小数据包（包长度小于MSS）均被确认，则允许发送</u>
5. <u>上述条件都未满足，但发生了超时（一般为200ms），则立即发送。</u>

**Nagle算法只允许一个未被ACK的包存在于网络，它并不管包的大小，因此它事实上就是一个扩展的停-等协议，只不过它是基于包停-等的，而不是基于字节停-等的。Nagle算法完全由TCP协议的ACK机制决定，这会带来一些问题，比如如果对端ACK回复很快的话，Nagle事实上不会拼接太多的数据包，虽然避免了网络拥塞，网络总体的利用率依然很低。**

也就是说，Nagle的算法是控制发送的，发送端在有未确认数据包并且本次发送包较小时，会等待后续数据包，直至超过mss才发送

**举个例子，client端调用socket的write操作将一个int型数据（称为A块）写入到网络中，由于此时连接是空闲的（也就是说还没有未被确认的小段），因此这个int型数据会被马上发送到server端，接着，client端又调用write操作写入‘rn’（简称B块），这个时候，A块的ACK没有返回，<u>所以可以认为已经存在了一个未被确认的小段</u>，所以B块没有立即被发送，一直等待A块的ACK收到（大概40ms之后），B块才被发送。**

这里还隐藏了一个问题，就是A块数据的ACK为什么40ms之后才收到？这是因为TCP/IP中不仅仅有nagle算法，还有一个TCP确认延迟机制 。当Server端收到数据之后，它并不会马上向client端发送ACK，而是会将ACK的发送延迟一段时间（假设为t），它希望在t时间内server端会向client端发送应答数据，这样ACK就能够和应答数据一起发送，就像是应答数据捎带着ACK过去。在我之前的时间中，t大概就是40ms。这就解释了为什么'rn'（B块）总是在A块之后40ms才发出。

抓包时经常看到PSH,ACK的包，正是这个机制的体现，将ack和数据一并送出

![clipboard.png](https://segmentfault.com/img/bVbIm3E)

其实就算没有延迟确认，Nagle这种只允许一个未确认包存在的机制，也是利用率很低的，因为完全串行化了，当网络延迟较高时，根本无法最大化利用物理带宽（有空试试Nagle在大带宽，高延迟的网络下的表现）

这两个机制配合在一起也是绝配，一个不回，一个不发，接收端不ack，发送端就不发送下一个小报文，也就造成了这个40ms（tcp delay ack）的延迟问题

**tcp_nodelay 禁用的是只是nagle算法，并没有禁用延迟确认**

禁用了nagle算法之后，相当于拆散了这对CP，发送端发送时也就不会再等待所有包都ack了；接收端的延迟ack影响并不是很大，只是会一定程度降低tcp 窗口滑动的速度而已
