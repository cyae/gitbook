## RST 攻击

正常情况下客户端服务端双方可以通过 RST 来断开连接。假设不做 seq 校验，如果这时候有不怀好意的第三方介入，构造了一个 RST 包，且在 TCP 和 IP 等报头都填上客户端的信息，发到服务端，那么服务端就会断开这个连接。同理也可以伪造服务端的包发给客户端。这就叫 RST 攻击。
![[640 (9).png]]
受到 RST 攻击时，从现象上看，客户端老感觉服务端崩了，这非常影响用户体验。

实际消息发送过程中，接收窗口是不断移动的，seq 也是在飞快的变动中，此时第三方是比较难构造出合法 seq 的 RST 包的，那么通过这个 seq 校验，就可以拦下了很多不合法的消息。

加了窗口校验就不能用 RST 攻击了吗
不是，只是增加了攻击的成本。但如果想搞，还是可搞的。

从上面可以知道，不是每一个 RST 包都会导致连接重置的，要求是这个 RST 包的 seq 要在窗口范围内，所以，问题就变成了，我们怎么样才能构造出合法的 seq。

### 盲猜 seq

窗口数值 seq 本质上只是个 uint32 类型。

```c
struct tcp_skb_cb {
    __u32       seq;        /* Starting sequence number */
}
```

如果在这个范围内疯狂猜测 seq 数值，并构造对应的包，发到目的机器，虽然概率低，但是总是能被试出来，从而实现 RST 攻击。这种乱棍打死老师傅的方式，就是所谓的合法窗口盲打（blind in-window attacks）。

觉得这种方式比较笨？那有没有聪明点的方式，还真有，但是在这之前需要先看下面的这个问题。

已连接状态下收到第一次握手包会怎么样？
我们需要了解一个问题，比如服务端在已连接（ESTABLISHED）状态下，如果收到客户端发来的第一次握手包（SYN），会怎么样？

以前我以为服务单会认为客户端憨憨了，直接 RST 连接。

但实际，并不是。

```c
static bool tcp_validate_incoming()
{
    struct tcp_sock *tp = tcp_sk(sk);

    /* 判断seq是否在合法窗口内 */
    if (!tcp_sequence(tp, TCP_SKB_CB(skb)->seq, TCP_SKB_CB(skb)->end_seq)) {
        if (!th->rst) {
            // 收到一个不在合法窗口内的SYN包
            if (th->syn)
                goto syn_challenge;
        }
    }

    /*
     * RFC 5691 4.2 : 发送 challenge ack
     */
    if (th->syn) {
syn_challenge:
        tcp_send_challenge_ack(sk);
    }
}
```

当客户端发出一个不在合法窗口内的 SYN 包的时候，服务端会发一个带有正确的 seq 数据 ACK 包出来，这个 ACK 包叫 challenge ack。
![[640 (10).png]]
上图是抓包的结果，用 scapy 随便伪造一个 seq=5 的包发到服务端（端口 9090），服务端回复一个带有正确 seq 值的 challenge ack 包给客户端（端口 8888）。

### 利用 challenge ack 获取 seq

上面提到的这个 challenge ack ，仿佛为盲猜 seq 的老哥们打开了一个新世界。

在获得这个 challenge ack 后，攻击程序就可以以 ack 值为基础，在一定范围内设置 seq，这样造成 RST 攻击的几率就大大增加了。
![[640 (11).png]]

## 参考资料

[TCP 旁路攻击分析与重现](https://www.cxyzjd.com/article/qq_27446553/52416369)
