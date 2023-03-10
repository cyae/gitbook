
前几天在群里看到这样一个图片，引起了我的兴趣：如果要用UDP实现类似TCP的可靠传输，一般需要手工实现的机制有那些？接下来我就以我的理解来讨论一下这个问题。  
![](https://img2020.cnblogs.com/blog/1459011/202010/1459011-20201007163031471-363133743.png)

那么先说结论吧：

-   1、添加seq/ack机制，确保数据发送到对端
-   2、添加发送和接收缓冲区，主要是用户超时重传。
-   3、添加超时重传机制。

## TCP详解

我们都知道TCP是面向连接的，进行可靠性传输的协议。接下来我们来看看TCP使用哪些技术来实现可靠的传输协议呢？

### 三次握手四次挥手

所谓三次握手，就是在建立TCP连接时，需要客户端和服务端总共发送3个包确认连接的建立。在socket编程中，这一过程有客户端执行connect触发。整个流程如下：

![](https://img2020.cnblogs.com/blog/1459011/202010/1459011-20201007163050511-765396812.png)

简单来说，就是：

1.  建立连接时，客户端发送SYN包（SYN=i）到服务器，并进入到SYS_SENT状态，等待服务器确认
2.  服务器收到SYN包，必须确认客户的SYN（ack=i+1）,同时自己也发送一个SYN包（SYN=K）,即SYN+ACK包，此时服务器进入SYN_RECV状态
3.  客户端收到服务器的SYN+ACK包，想服务器发送确认报ACK（ack=K+1），此时发送完毕，客户端和服务端进入ESTABLISHED状态，完成三次握手，客户端与服务端开始传送数据。

通俗的说，加入你老大要跟你说你的程序有bug，给你打电话的过程如下：

> 老大：你听的到吗？
> 
> 你：我听的到，你听得到吗？
> 
> 老大：嗯，我也听得到。
> 
> 确认双方都能听得到对方的讲话，开始讲正事。。。。

所谓四次挥手，即断开TCP连接时，需要客户端和服务端总共发送4个包以确认连接的断开，在socket编程中，这一过程由客户端或者服务端任一方执行close来触发。**由于TCP是双全工的，因此每个方向都必须要单独关闭，这一原则是当一方完成数据发送后，发送一个FIN来终止这一方的连接，收到一个FIN只是意味着这一方向上没有数据流动了，即不会再发送数据了，但是在这个TCP连接上仍然能够发送数据，直到灵一方向也发送了FIN。首先进行关闭的一方将执行主动关闭，而另一方则执行被动关闭。**整个流程如下：

![](https://img2020.cnblogs.com/blog/1459011/202010/1459011-20201007163109240-1766471756.png)

简单来说，就是：

1.  Client发送一个FIN，用来关闭Client到Server的数据传送，Client进入FIN_WAIT_1状态。
2.  Server收到FIN后，发送一个ACK给Client，确认序号为收到序号+1（与SYN相同,一个FIN占用一个序号），Server进入CLOSE_WAIT状态。
3.  Server发送一个FIN，用来关闭Server到Client的数据传送，Server进入LAST_ACK状态。
4.  Client收到FIN后，Client进入TIME_WAIT状态，接着发送一个ACK给Server，确认序号为收到序号+1，Server进入CLOSED状态，完成四次挥手。

### TCP可靠性传输的几种实现机制

#### 1.确认应答（ACK）机制（ 即上面讲到的三次握手，四次挥手）

TCP将每个字节的数据都进行了编号，即为序列号（seq），确认序号=序号+1，每个ACK都有对应的确认序列号，意思是告诉发送者已经收到了数据，下一个数据应该从哪里发送

#### 2.超时重传机制（两种情况）

1.  如果主机A发送给主机B的报文，主机B在规定的时间内没有及时收到主机A发送的报文，我们可以认为是ACK丢了，这时就需要触发超时重传机制。
2.  如果主机A未收到B发来的确认应答，也可能是因为ACK丢了。因此主机B会收到很多重复的数据，那么，TCP协议需要能够识别出那些包是重复的包，并且把重复的包丢弃，这时候我们可以用前面提到的序列号，很容易做到去重的效果

#### 3.滑动窗口实现流量控制

所谓**流量控制，就是让发送方的发送速率不要太快，要让对方来的及接受。**

当发送一次数据，等到确认应答时才可以发送下一个数据段，这样的效率会很低，我们利用滑动窗口，无需等待确认应答而可以继续发送数据的最大值；收到第一个ACK后，滑动窗口向后移；操作系统为了维护这个滑动窗口，需要开辟发送缓冲区来记录当前还有那些数据没有应答，只有确认应答过的数据，才能从缓冲区删除掉；窗口越大，网络的吞吐率越高TCP为每一个连接设有一个持续计时器，只要TCP连接的一方收到对方的零窗口通知，就启动持续计时器，若持续计时器设置的时间到期，就发送一个零窗口探测报文段（仅携带一个字节的数据），对方就在确认这个探测报文段时给出了现在的窗口 值，如果仍然是0，那么收到这个报文段的一方就重新设置持续计时器；如果窗口不是0，那么死锁的僵局就可以打破

![](https://img2020.cnblogs.com/blog/1459011/202010/1459011-20201007163127513-476161812.png)

在这传输过程中，出现了丢包的情况，这里就不做解说了。

#### 4.拥塞控制

虽然有了滑动窗口机制，如果一开始就发送大量数据，很有可能引发很多问题。所以TCP加入慢启动机制，先发少量的数据探探路，看看当前网络的拥塞状态，再决定按照多大的速率进行传送，刚开始时，定义拥塞窗口的大小为1，每次接收到一个ACK应答，拥塞窗口值加1，每次发送数据包的时候，将拥塞窗口和接收端主机反馈的窗口大小做比较，取较小的值作为实际发送的窗口。这样 的拥塞窗口增长的速度是指数级别的，慢启动只是指初始时慢，但是增长速度很快，不久就可以造成网 络拥塞。为了不让窗口一直加倍增长，我们引入一个慢启动的阈值，当拥塞窗口超过这个阈值的时候，不在按指数方式增长，而是按照线性方式增长。

拥塞控制，归根结底是TCP协议想尽可能快的把数据传输给对方，但又要避免给网络造成最大压力的最好方案

## UDP详解

UDP是User Datagram Protocol，一种无连接的传输层协议，提供面向事务的简单不可靠信息传送服务。可靠性由上层应用实现，所以要实现udp可靠性传输，必须通过应用层来实现和控制。像实时视频，直播等要求以稳定的速度发送，能够容忍一些数据的丢失，但是不允许又较大的时延，就会采用UDP协议。

UDP提供的是不可靠传输服务，具有TCP没有的优势:

-   UDP是无连接的，在时间上不存在建立连接需要的延时，在空间上，TCP需要在系统中维护连接状态，需要一定的开销。UDP不需要维护连接状态，也不跟踪这些参数，开销小，时间和空间上都具有优势。
-   分组首部开销小，TCP首部20字节，UDP首部8字节。
-   UDP没有拥塞控制，应用层能够更好的控制要发送的苏剧和发送时间，网络中的拥塞控制也不会影响主机的发送速率。
-   UDP提供尽最大的努力交付，不保证可靠交付。所以维护传输可靠性的工作需要用户在应用层来完成，没有TCP的确认机制，重传机制。
-   UDP是面向报文的，对应用层传下来的报文，添加首部信息后直接向下交付给IP层。既不合并，也不拆分（TCP有粘包拆包问题）。对于IP层交上来的UDP用户数据报，在去除首部后就原封不动的交付给上层应用进程，报文不可分割，是UDP数据报的最小单位。
-   UDP常用一次性传输比较少量数据的网络应用，如DNS,SNMP等，因为对于这些应用，若是采用TCP，为连接的创建，维护和拆除带来不小的开销。UDP也常用于多媒体应用（如IP电话，实时视 频会议，流媒体等）数据的可靠传输对他们而言并不重要，TCP的拥塞控制会使他们有较大的延迟，也是不可容忍的。

## 代码

```java
package com.example.charon.entity;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.DataInputStream;
import java.io.DataOutputStream;
import java.io.IOException;
import java.net.SocketAddress;
import java.util.Arrays;

/**
 * @className: MessageEntity
 * @description: 请求发送消息实体的结构定义
 * @author: charon
 * @create: 2020-09-18 08:20
 */
public class RequestMessage {

    /**定义数据的长度*/
    private int totalLen;

    /**生成唯一的id*/
    private int id;

    /**数据*/
    private byte[] data;

    /**发送次数*/
    private int sendCount = 0;

    /**最后一次发送时间*/
    private Long lastSendTime = System.currentTimeMillis();

    /**发送者接受应答的地址*/
    private SocketAddress recvRespAddr;

    /**接收者的地址*/
    private SocketAddress remoteAddr;

    public RequestMessage(int id, byte[] data) {
        this.id = id;
        this.data = data;
        // 4+4是因为每个int类型占4个字节
        this.totalLen = 4 + 4 + data.length;
    }

    /**
     * 构造器将收到的udp数据解析为tcp的requestMessage对象
     * @param udpData udp数据
     */
    public RequestMessage(byte[] udpData){
        try {
            ByteArrayInputStream bais = new ByteArrayInputStream(udpData);
            DataInputStream dis = new DataInputStream(bais);
            this.totalLen = dis.readInt();
            this.id = dis.readInt();
            this.data = new byte[totalLen - 4 - 4];
            dis.readFully(data);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public byte[] toByte(){
        try {
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            DataOutputStream dos = new DataOutputStream(baos);
            // 写和读需要一一对应
            dos.writeInt(totalLen);
            dos.writeInt(id);
            dos.write(data);
            dos.flush();
            return baos.toByteArray();
        }catch (IOException e){
            e.printStackTrace();
        }
        return null;
    }

    public int getTotalLen() {
        return totalLen;
    }

    public void setTotalLen(int totalLen) {
        this.totalLen = totalLen;
    }

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public byte[] getData() {
        return data;
    }

    public void setData(byte[] data) {
        this.data = data;
    }

    public int getSendCount() {
        return sendCount;
    }

    public void setSendCount(int sendCount) {
        this.sendCount = sendCount;
    }

    public Long getLastSendTime() {
        return lastSendTime;
    }

    public void setLastSendTime(Long lastSendTime) {
        this.lastSendTime = lastSendTime;
    }

    /**
     * Gets the value of recvRespAddr
     *
     * @return the value of recvRespAddr
     */
    public SocketAddress getRecvRespAddr() {
        return recvRespAddr;
    }

    /**
     * Sets the recvRespAddr
     *
     * @param recvRespAddr recvRespAddr
     */
    public void setRecvRespAddr(SocketAddress recvRespAddr) {
        this.recvRespAddr = recvRespAddr;
    }

    /**
     * Gets the value of remoteAddr
     *
     * @return the value of remoteAddr
     */
    public SocketAddress getRemoteAddr() {
        return remoteAddr;
    }

    /**
     * Sets the remoteAddr
     *
     * @param remoteAddr remoteAddr
     */
    public void setRemoteAddr(SocketAddress remoteAddr) {
        this.remoteAddr = remoteAddr;
    }

    @Override
    public String toString() {
        return "RequestMessage{" +
                "totalLen=" + totalLen +
                ", id=" + id +
                ", data=" + Arrays.toString(data) +
                ", sendCount=" + sendCount +
                ", lastSendTime=" + lastSendTime +
                '}';
    }
}

```

```java
package com.example.charon.entity;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.DataInputStream;
import java.io.DataOutputStream;
import java.io.IOException;
import java.net.SocketAddress;
import java.util.Arrays;

/**
 * @className: ResponseMessage
 * @description: 响应信息实体的结构定义
 * @author: charon
 * @create: 2020-09-18 09:02
 */
public class ResponseMessage {

    /**总长度*/
    private int totalLen;

    /**对应接收到消息的id*/
    private int repId;

    /**响应的数据*/
    private byte[] data;

    /**接收状态 0：正确接收 其他：错误 */
    private int state = 0;

    /**应答方的发送时间*/
    private Long resTime;

    /**发送次数*/
    private int sendCount;

    /**最后一次发送时间*/
    private Long lastSendTime = System.currentTimeMillis();

    /**接收者的地址*/
    private SocketAddress remoteAddr;

    public ResponseMessage(int repId, int state, byte[] data) {
        this.repId = repId;
        this.state = state;
        this.data = data;
        // 4+4+4是因为每个int类型占4个字节
        this.totalLen = 4 + 4 + 4 + data.length;
    }

    public ResponseMessage(byte[] udpData){
        try {
            ByteArrayInputStream bais = new ByteArrayInputStream(udpData);
            DataInputStream dis = new DataInputStream(bais);
            this.totalLen = dis.readInt();
            this.repId = dis.readInt();
            this.state = dis.readInt();
            this.data = new byte[totalLen - 4 - 4 -4 ];
            dis.readFully(data);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public byte[] toByte(){
        try {
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            DataOutputStream dos = new DataOutputStream(baos);
            // 写和读需要一一对应
            dos.writeInt(totalLen);
            dos.writeInt(repId);
            dos.writeInt(state);
            dos.write(data);
            dos.flush();
            return baos.toByteArray();
        }catch (IOException e){
            e.printStackTrace();
        }
        return null;
    }



    public int getTotalLen() {
        return totalLen;
    }

    public void setTotalLen(int totalLen) {
        this.totalLen = totalLen;
    }

    public int getRepId() {
        return repId;
    }

    public void setRepId(int repId) {
        this.repId = repId;
    }

    public int getState() {
        return state;
    }

    public void setState(int state) {
        this.state = state;
    }

    public Long getResTime() {
        return resTime;
    }

    public void setResTime(Long resTime) {
        this.resTime = resTime;
    }

    /**
     * Gets the value of sendCount
     *
     * @return the value of sendCount
     */
    public int getSendCount() {
        return sendCount;
    }

    /**
     * Sets the sendCount
     *
     * @param sendCount sendCount
     */
    public void setSendCount(int sendCount) {
        this.sendCount = sendCount;
    }

    /**
     * Gets the value of lastSendTime
     *
     * @return the value of lastSendTime
     */
    public Long getLastSendTime() {
        return lastSendTime;
    }

    /**
     * Sets the lastSendTime
     *
     * @param lastSendTime lastSendTime
     */
    public void setLastSendTime(Long lastSendTime) {
        this.lastSendTime = lastSendTime;
    }

    /**
     * Gets the value of remoteAddr
     *
     * @return the value of remoteAddr
     */
    public SocketAddress getRemoteAddr() {
        return remoteAddr;
    }

    /**
     * Sets the remoteAddr
     *
     * @param remoteAddr remoteAddr
     */
    public void setRemoteAddr(SocketAddress remoteAddr) {
        this.remoteAddr = remoteAddr;
    }

    /**
     * Gets the value of data
     *
     * @return the value of data
     */
    public byte[] getData() {
        return data;
    }

    /**
     * Sets the data
     *
     * @param data data
     */
    public void setData(byte[] data) {
        this.data = data;
    }

    @Override
    public String toString() {
        return "ResponseMessage{" +
                "totalLen=" + totalLen +
                ", repId=" + repId +
                ", data=" + Arrays.toString(data) +
                ", state=" + state +
                ", resTime=" + resTime +
                ", sendCount=" + sendCount +
                ", lastSendTime=" + lastSendTime +
                '}';
    }
}

```

```java
package com.example.charon;

import com.example.charon.entity.RequestMessage;
import com.example.charon.entity.ResponseMessage;

import java.io.IOException;
import java.net.DatagramPacket;
import java.net.DatagramSocket;
import java.net.InetSocketAddress;
import java.net.SocketAddress;
import java.net.SocketException;
import java.util.Iterator;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

/**
 * @className: DataGramSend
 * @description: 数据报文发送：
 *                1.发送消息线程负责发送，发送后将消息放入容器中等待应答，
 *                2.接受线程接收应答，并发送消息给接收端自己已收到信息，从容器中匹配后删除
 *                3.重发线程负责重发，未收到应答的消息，发送3次后移出
 * @author: charon
 * @create: 2020-09-18 23:47
 */
public class DatagramSend {

    /**本地要发送的地址对象*/
    private SocketAddress localAddress;

    /**发送的socket对象*/
    private DatagramSocket datagramSender;

    /**目标地址*/
    private SocketAddress remoteAddress;

    /**本地缓存已发送的消息 Map key 为消息ID，value为消息对象本身*/
    private Map<Integer, RequestMessage> msgQueue = new ConcurrentHashMap<>();

    public static void main(String[] args) throws SocketException {
        new DatagramSend();
    }

    public DatagramSend() throws SocketException {
        localAddress = new InetSocketAddress("127.0.0.1",13000);
        datagramSender = new DatagramSocket(localAddress);
        remoteAddress = new InetSocketAddress("127.0.0.1",14000);

        // 启动三个线程
        // 1.发送消息线程
        startSendThread();
        // 接收线程接收应答
        startRecvResponseThread();
        // 重发线程负责重发
        startReSendThread();
    }

    /**
     * 启动重发线程
     */
    @SuppressWarnings("all")
    private void startReSendThread() {
        new Thread(new Runnable() {
            @Override
            public void run() {
                try{
                    while (true){
                        resendMsg();
                        Thread.sleep(1000);
                    }
                }catch (Exception e){
                    e.printStackTrace();
                }
            }
        }).start();
    }

    /**
     * 重发业务：判断map中的消息，如果超过3s未收到应答，则重发
     */
    private void resendMsg() {
        // 返回队列中所有的key
        Set<Integer> keySet = msgQueue.keySet();
        Iterator<Integer> iterator = keySet.iterator();
        while(iterator.hasNext()){
            Integer key = iterator.next();
            RequestMessage requestMessage = msgQueue.get(key);

            // 如果重发超过3次，则移出
            if(requestMessage.getSendCount() >= 3){
                iterator.remove();
                System.out.println("发送端--检测道丢失的消息：" + requestMessage);
            }

            long startTime =  System.currentTimeMillis();
            // 等待时间不超过3s
            if((startTime - requestMessage.getLastSendTime()) > 3000 && requestMessage.getSendCount() < 3 ){
                byte[] buffer = requestMessage.toByte();
                try {
                    DatagramPacket datagramPacket = new DatagramPacket(buffer,buffer.length,requestMessage.getRemoteAddr());
                    datagramSender.send(datagramPacket);
                    requestMessage.setSendCount(requestMessage.getSendCount()+1);
                    System.out.println("客户端重新发送消息:"+requestMessage);
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    /**
     * 启动接受应答线程
     */
    @SuppressWarnings("all")
    private void startRecvResponseThread() {
        new Thread(new Runnable() {
            @Override
            public void run() {
                try{
                    recvResponse();
                }catch (Exception e){
                    e.printStackTrace();
                }
            }
        }).start();
    }

    /**
     * 接受应答消息
     */
    private void recvResponse() throws IOException {
        System.out.println("接收端-接受应答线程启动");
        while (true){
            byte[] recvData = new byte[100];
            //创建接受数据包对象
            DatagramPacket recvRespPacket = new DatagramPacket(recvData,recvData.length);
            //发送
            datagramSender.receive(recvRespPacket);
            //接受返回数据
            RequestMessage requestMessage = new RequestMessage(recvRespPacket.getData());
            int repId = requestMessage.getId();
            RequestMessage requestMessage1 = msgQueue.get(new Integer(repId));
            if(requestMessage1 != null){
                System.out.println("发送端-原来发送的数据："+requestMessage1);
                System.out.println("接受的数据：" + requestMessage);
                System.out.println("发送端-已收到接收端返回的信息："+new String(requestMessage.getData()));
                msgQueue.remove(repId);
                //发送端需要告诉接收端，返回的数据已经收到
                //发送的数据
                byte[] msgData = (repId+" 数据已收到").getBytes();
                //创建要发送的消息对象
                RequestMessage sendMessage = new RequestMessage(repId,msgData);

                //要发送的数据，将要发送的数据转为字节数组
                byte[] buffer = sendMessage.toByte();
                //创建书包，指定内容，指定目标地址
                DatagramPacket datagramSocket = new DatagramPacket(buffer,buffer.length,remoteAddress);
                //发送数据
                datagramSender.send(datagramSocket);
            }
        }
    }

    /**
     * 发送消息线程
     */
    @SuppressWarnings("all")
    private void startSendThread() {
        new Thread(new Runnable() {
            @Override
            public void run() {
                try{
                    send();
                }catch (Exception e){
                    e.printStackTrace();
                }
            }
        }).start();
    }

    /**
     * 模拟发送消息
     */
    private void send() throws IOException, InterruptedException {
        System.out.println("发送端-发送数据线程启动");
        // 确认机制，id从0开始
        int id = 0;
        //模拟发送10个请求
        while (id < 1){
            id++;
            //发送的数据
            byte[] msgData = (id+" hello").getBytes();
            //创建要发送的消息对象
            RequestMessage sendMessage = new RequestMessage(id,msgData);

            //要发送的数据，将要发送的数据转为字节数组
            byte[] buffer = sendMessage.toByte();
            //创建书包，指定内容，指定目标地址
            DatagramPacket datagramSocket = new DatagramPacket(buffer,buffer.length,remoteAddress);
            //发送数据
            datagramSender.send(datagramSocket);
            // 缓存当前发送的请求
            sendMessage.setSendCount(1);
            sendMessage.setRemoteAddr(remoteAddress);
            msgQueue.put(id,sendMessage);
            System.out.println("客户端-数据已发送，缓存："+sendMessage);
            Thread.sleep(1000);
        }
    }
}

```

```java
package com.example.charon;

import com.example.charon.entity.RequestMessage;
import com.example.charon.entity.ResponseMessage;

import java.io.IOException;
import java.net.DatagramPacket;
import java.net.DatagramSocket;
import java.net.InetSocketAddress;
import java.net.SocketAddress;
import java.net.SocketException;
import java.util.Iterator;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

/**
 * @className: DatagramRecive
 * @description: 数据报文接受方
 * @author: charon
 * @create: 2020-09-20 17:56
 */
public class DatagramRecive {

    private SocketAddress localAddress;

    private DatagramSocket datagramSender;

    /**目标地址*/
    private SocketAddress remoteAddress;

    /**本地缓存已发送的消息 Map key 为消息ID，value为消息对象本身*/
    private Map<Integer, ResponseMessage> msgQueue = new ConcurrentHashMap<>();

    public static void main(String[] args) throws IOException {
        new DatagramRecive();
    }

    public DatagramRecive() throws SocketException {
        localAddress = new InetSocketAddress("127.0.0.1",14000);
        datagramSender = new DatagramSocket(localAddress);
        remoteAddress = new InetSocketAddress("127.0.0.1",13000);
        //  启动接收线程
        startDecvThread();
        // 接收线程接收应答
        startDecvResponseThread();
        // 重发线程负责重发
        startReSendThread();
    }

    /**
     * 重发
     */
    @SuppressWarnings("all")
    private void startReSendThread() {
        new Thread(new Runnable() {
            @Override
            public void run() {
                try{
                    while (true){
                        resendMsg();
                        Thread.sleep(1000);
                    }
                }catch (Exception e){
                    e.printStackTrace();
                }
            }
        }).start();
    }

    private void resendMsg() {
        // 返回队列中所有的key
        Set<Integer> keySet = msgQueue.keySet();
        Iterator<Integer> iterator = keySet.iterator();
        while(iterator.hasNext()){
            Integer key = iterator.next();
            ResponseMessage responseMessage = msgQueue.get(key);

            // 如果重发超过3次，则移出
            if(responseMessage.getSendCount() >= 3){
                iterator.remove();
                System.out.println("发送端--检测道丢失的消息：" + responseMessage);
            }

            long startTime =  System.currentTimeMillis();
            // 等待时间不超过3s
            if((startTime - responseMessage.getLastSendTime()) > 3000 && responseMessage.getSendCount() < 3 ){
                byte[] buffer = responseMessage.toByte();
                try {
                    DatagramPacket datagramPacket = new DatagramPacket(buffer,buffer.length,responseMessage.getRemoteAddr());
                    datagramSender.send(datagramPacket);
                    responseMessage.setSendCount(responseMessage.getSendCount()+1);
                    System.out.println("客户端重新发送消息:"+responseMessage);
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    @SuppressWarnings("all")
    private void startDecvResponseThread() {
        new Thread(new Runnable() {
            @Override
            public void run() {
                try{
                    decvResponse();
                }catch (Exception e){
                    e.printStackTrace();
                }
            }
        }).start();
    }

    private void decvResponse() throws IOException {
        System.out.println("发送端-接受应答线程启动");
        while (true){
            byte[] recvData = new byte[100];
            //创建接受数据包对象
            DatagramPacket recvRespPacket = new DatagramPacket(recvData,recvData.length);
            //接受数据
            datagramSender.receive(recvRespPacket);
            //接受返回数据
            RequestMessage requestMessage = new RequestMessage(recvRespPacket.getData());
            int repId = requestMessage.getId();
            System.out.println("接收端接受到的数据id：" + repId);
            ResponseMessage responseMessage = msgQueue.get(new Integer(repId));
            if(responseMessage != null){
                System.out.println("接收端发送的源数据："+responseMessage);
                System.out.println("接收端已收到发送端返回的数据："+ new String(requestMessage.getData()));
                msgQueue.remove(repId);
            }
        }
    }

    @SuppressWarnings("all")
    private void startDecvThread() {
        new Thread(new Runnable() {
            @Override
            public void run() {
                try{
                    recvMsg();
                }catch (Exception e){
                    e.printStackTrace();
                }
            }
        }).start();
    }

    private void recvMsg() throws IOException {
        System.out.println("启动接收线程");
        while (true){
            // 接受发送端发送过来的数据 100表示缓存的长度
            byte[] recvData = new byte[100];
            DatagramPacket datagramPacket = new DatagramPacket(recvData,recvData.length);
            datagramSender.receive(datagramPacket);
            //获取接收端发送的数据
            RequestMessage requestMessage = new RequestMessage(datagramPacket.getData());
            String requestMessageData = new String(requestMessage.getData());
            System.out.println("接收端收到发送端的数据：" + requestMessageData);

            // 将接收到的数据发送给发送端，
            byte[] responseData = (requestMessageData+" world").getBytes();
            ResponseMessage responseMessage = new ResponseMessage(requestMessage.getId(),0,responseData);
            System.out.println("接收端返回的数据：" + new String(responseMessage.getData()));
            byte[] data = responseMessage.toByte();
            DatagramPacket dp = new DatagramPacket(data,data.length,remoteAddress);
            datagramSender.send(dp);

            //将接收端返回的数据存入队列中，用于后面监听重发机制
            responseMessage.setLastSendTime(System.currentTimeMillis());
            responseMessage.setSendCount(1);
            responseMessage.setData(responseMessage.getData());
            //对于接收端来说，它需要返回的地址就是请求消息的本地
            responseMessage.setRemoteAddr(remoteAddress);
            msgQueue.put(requestMessage.getId(),responseMessage);

            System.out.println("接收端-已发送应答：" + responseMessage);
        }
    }
}

```

代码连接：[https://download.csdn.net/download/zj520_/12913781](https://download.csdn.net/download/zj520_/12913781)  
参考文章：[http://www.360doc.com/content/13/0602/11/11220452_289877920.shtml](http://www.360doc.com/content/13/0602/11/11220452_289877920.shtml)  
[https://blog.csdn.net/codes_first/article/details/78453713?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-4.channel_param&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-4.channel_param](https://blog.csdn.net/codes_first/article/details/78453713?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-4.channel_param&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-4.channel_param)