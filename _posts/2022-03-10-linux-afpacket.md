---
title: 'Deep dive into AF_PACKET socket'
date: 2022-03-10 14:52:53 +0800
author: csulrong
version: 3.4.3
categories: [linux]
tags: [Linux]
excerpt_separator: <!-- more -->
---

AF_PACKET socket is a way of capturing raw packets at link layer and then applications can handle the packets in user space. Applications like packet sniffering and soft packet processing could use it to receive and send the raw packets from and to the link layer directly. 

AF_PACKET had evolved a few versions of improvement in Linux kernel since it's introduced. In this artcile, we will discuss what options could be used to improve the efficiency for capturing and sending the packet via the AF_PACKET socket. 

<!-- more -->

## Plain AF_PACKET Socket

The AF_PACKET socket allows user-space application to capture raw packets at link layer so that it can see the whole packet data starting from link-layer headers and bottom up to transport layer and application payload. 

Application creates the AF_PACKET socket like other types of socket with the `socket` function:

{% highlight c %}
int fd = socket(AF_PACKET, SOCK_RAW, htons(ETH_P_ALL));
{% endhighlight %}

where, 
- The first argument `AF_PACKET` indicates socket family. 
- The second argument could be either `SOCK_RAW` or `SOCK_DGRAM`. If you want to receive the packet with the 14-byte Ethernet header, `SOCK_RAW` is the right socket type, or else the link layer head will be removed in case of `SOCK_DGRAM`. 
- The third arugment specifies the link-layer protocol, `ETH_P_ALL` indicates all protocols and `ETH_P_IP` indicates IPv4 protocol. The protocol definition follows the convention of `ETH_P_xxx`, which can be found at `linux/if_ether.h` header file.

## AF_PACKET with MMAP

Receving / sending packets from / to plain AF_PACKET socket is very inefficient as it uses very limited buffers and requires system call once every time to capture a packet or send a packet from / to kernel; meanwhile the packet data has to be moved between the user and kernel spaces. Given this, PACKET_MMAP arised to boost the performance by eliminating the need of moving packet data between user and kernel spaces and also reducing the number of system calls. A size configurable ring buffer is shared between kernel and user spaces so that user applications just need to wait for packets at receiving side. Concerning tranmission, multiple packets can be put to the ring buffer followed by one system call to notify kernel transmitting these packets.

### Ring Buffer

The AF_PACKET has `PACKET_RX_RING` and `PACKET_TX_RING` respectively for packet reception and transmission. A ring buffer is a contiguous physical region of memory, which is logicially segmented into a number of blocks. Each block contains a few frames and each frame has two parts:
- frame header: It contains the status of this frame.
- data buffer: It holds the packet data.

The `PACKET_MMAP` for AF_PACKET evolved 3 versions:
- TPACKET_V1
- TPACKET_V2
  * Timestamp resolution at nanosecond scale instead of microsecond.
  * VLAN metadata information is available for packets.
- TPACKET_V3
  * Read / poll is at block level instead of frame level.
  * Added poll timeout to avoid blocking poll.
  * RX hash data is available to user space application.

By default TPACKET_V1 is used, but use `setsockopt` function to change the version to TPACKET_V3 is highly recommended as polling at block level brings the benefit of 15% - 20% reduction in CPU usage, and ~20% increase in packet capture rate.

{% highlight c %}
int v = TPACKET_V3;
err = setsockopt(fd, SOL_PACKET, PACKET_VERSION, &v, sizeof(v));
{% endhighlight %}

To setup rings for RX and TX, TPACKET_V1 and TPACKET_V2 uses `struct tpacket_req` and TPACKET_V3 uses `struct tpacket_req3`, both struct's are defined in `uapi/linux/if_packet.h`. The following piece of code sets the `PACKET_RX_RING` with 128 blocks, each block has 4096 bytes and contains of 2 frames with frame size of 2048 bytes.

{% highlight c %}
struct tpacket_req3 req;
req.tp_block_size = 4096;
req.tp_frame_size = 2048;
req.tp_block_nr   = 128;
req.tp_frame_nr   = (req.tp_block_size * req.tp_block_nr) / req.tp_frame_size;
err = setsockopt(fd, SOL_PACKET, PACKET_RX_RING, &req, sizeof(req));
{% endhighlight %}

Similarly, you can also use the `setsockopt` function to setup the `PACKET_TX_RING` for packet transmission. Next, the application has to create the ring buffer with `mmap` function to share the memory between user and kernel spaces. The ring buffer will be formatted as blocks and frames based on the the parameters setting up the `PACKET_RX_RING` or `PACKET_TX_RING`.

{% highlight c %}
unsigned int total_size = req.tp_block_size * req.tp_block_nr;
ring = mmap(NULL, total_size, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
{% endhighlight %}

where,
- The fist argument specifies the starting address of the shared buffer, if it is `NULL`, kernel chooses the address at which to create the mapping. 
- The second argument specifies the total size of the shared buffer.
- `PROT_READ|PROT_WRITE` at the third argument indicates the mapping space is readable and writable. 
- Flag at the fourth argument determines whether updates to the mapping are visible to other processes mapping the same memory space.
- The last argument is an offset always set to 0 in mapping ring bufffer for AF_PACKET. 


### Receiving Packets

The following macros defined in `include/linux/if_packet.h` implies the status of a frame in the ring.

{% highlight c %}
#define TP_STATUS_KERNEL        0
#define TP_STATUS_USER          1
#define TP_STATUS_COPY          (1 << 1)
#define TP_STATUS_LOSING        (1 << 2)
#define TP_STATUS_CSUM_VALID    (1 << 7)
{% endhighlight %}

The kernel initializes all frames to `TP_STATUS_KERNEL`. When the kernel receives a packet it puts in the ring buffer and updates the status with at least the `TP_STATUS_USER` flag. Userspace application has to poll the socket file descriptor to check if there are new packets in the ring. Then the application can read the packet if the status has the `TP_STATUS_USER` flag, once the packet is read the application must zero the status field, so the kernel can reuse that frame buffer to store next received packet. The rest of status flags are explained in the following table.

<div class="mobile-side-scroller">
<table>
  <thead>
    <tr>
      <th>Macro</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>
        <p><code>TP_STATUS_COPY</code></p>
      </td>
      <td>
        <p>
          This flag indicates that the frame (and associated metadata) has been truncated because it’s larger than <code>tp_frame_size</code>. This packet can be read entirely with <code>recvfrom()</code>. However, in order to make this work it must to be enabled previously with <code>setsockopt()</code> and the <code>PACKET_COPY_THRESH</code> option.
        </p>
      </td>
    </tr>
    <tr>
      <td>
        <p><code>TP_STATUS_LOSING</code></p>
      </td>
      <td>
        <p>
          Indicates there were packet drops from last time statistics where checked with <code>getsockopt()</code> and the <code>PACKET_STATISTICS</code> option.
        </p>
      </td>
    </tr>
    <tr>
      <td>
        <p><code>TP_STATUS_CSUM_VALID</code></p>
      </td>
      <td>
        <p>
          This flag indicates that at least the transport header checksum of the packet has been already validated on the kernel side. If the flag is not set then the userspace applications are free to check the checksum provided that <code>TP_STATUS_CSUMNOTREADY</code> is also not set.
        </p>
      </td>
    </tr>
  </tbody>
</table>
</div>

#### TPACKET_V3 block descriptor

Since TPACKET_V3 introduced the polling at block level, there is a block descriptor describes the status and information of each block. The structure of a block is depicted as the following diagram.

![tpacketv3_block]({{ site.baseurl }}/assets/images/2022/tpv3_block.svg)

The following example code shows how to do block-level polling with TPACKET_V3 and walk through frames in the block.
{% highlight c %}
static void walk_block(struct block_desc *pbd, const int block_num)
{
    int num_pkts = pbd->h1.num_pkts, i;
    unsigned long bytes = 0;
    struct tpacket3_hdr *ppd;

    ppd = (struct tpacket3_hdr *) ((uint8_t *) pbd +
                                pbd->h1.offset_to_first_pkt);
    for (i = 0; i < num_pkts; ++i) {
            bytes += ppd->tp_snaplen;
            display(ppd);

            ppd = (struct tpacket3_hdr *) ((uint8_t *) ppd +
                                        ppd->tp_next_offset);
    }

    packets_total += num_pkts;
    bytes_total += bytes;
}

static void flush_block(struct block_desc *pbd)
{
    pbd->h1.block_status = TP_STATUS_KERNEL;
}

int main(int argc, char **argp)
{
    //......

    memset(&pfd, 0, sizeof(pfd));
    pfd.fd = fd;
    pfd.events = POLLIN | POLLERR;
    pfd.revents = 0;

    while (1) {
        pbd = (struct block_desc *) ring.rd[block_num].iov_base;

        if ((pbd->h1.block_status & TP_STATUS_USER) == 0) {
            poll(&pfd, 1, -1);
            continue;
        }

        walk_block(pbd, block_num);
        flush_block(pbd);
        block_num = (block_num + 1) % blocks;
    }

    //......
}
{% endhighlight %}

#### Load balancing

The `AF_PACKET` fanout mode enables load balancing capability for packet reception. You can load-balance the packet reception among multiple processes or CPUs based on the following policies.

<div class="mobile-side-scroller">
<table>
  <thead>
    <tr>
      <th>Fanout Policy</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><p><code>PACKET_FANOUT_HASH</code></p></td>
      <td><p>Schedule to socket by <code>skb</code>’s packet hash.</p></td>
    </tr>
    <tr>
      <td><p><code>PACKET_FANOUT_LB</code></p></td>
      <td><p>Schedule to socket by round-robin.</p></td>
    </tr>
    <tr>
      <td><p><code>PACKET_FANOUT_CPU</code></p></td>
      <td><p>Schedule to socket by CPU packet arrives on.</p></td>
    </tr>
    <tr>
      <td><p><code>PACKET_FANOUT_RND</code></p></td>
      <td><p>Schedule to socket by random selection.</p></td>
    </tr>
    <tr>
      <td><p><code>PACKET_FANOUT_ROLLOVER</code></p></td>
      <td><p>If one socket is full, rollover to another.</p></td>
    </tr>
    <tr>
      <td><p><code>PACKET_FANOUT_QM</code></p></td>
      <td><p>chedule to socket by <code>skb</code>'s recorded <code>queue_mapping</code>.</p></td>
    </tr>
  </tbody>
</table>
</div>


### Transmitting Packets

There are also macros defined for transmission process:

{% highlight c %}
#define TP_STATUS_AVAILABLE        0 // Frame is available
#define TP_STATUS_SEND_REQUEST     1 // Frame will be sent on next send()
#define TP_STATUS_SENDING          2 // Frame is currently in transmission
#define TP_STATUS_WRONG_FORMAT     4 // Frame format is not correct
{% endhighlight %}

First, the kernel initializes all frames to `TP_STATUS_AVAILABLE`. To send a packet, the application fills a data buffer of an available frame, sets `tp_len` to current data buffer size and sets its status field to `TP_STATUS_SEND_REQUEST`. This can be done on multiple frames. Once the application is ready to transmit, it calls `send()`. Then all buffers with status equal to `TP_STATUS_SEND_REQUEST` are forwarded to the network device. The kernel updates each status of sent frames with `TP_STATUS_SENDING` until the end of transfer. At the end of each transfer, buffer status returns to `TP_STATUS_AVAILABLE`. So when application fills packet into a frame, it should ensure not overriding packet that is in transmission.

Specific to TPACKET_V3, unlike the structure of blocks in RX ring, which has a block descriptor for each block, TX ring doesn't have the block descriptor as it doesn't need to poll. So sending a packet is quite straightforward like the below code does.

{% highlight c %}
struct tpacket3_hdr *hdr = NULL;

// iterate frames to find an available one for holding packet
for (i = 0; i < req.tp_frame_nr; i += 1) {
    hdr = (void*)(ring + (req.tp_frame_size * i));
    data = (uint8_t*)hdr + TPACKET_ALIGN(sizeof(struct tpacket3_hdr));
    
    if (hdr->tp_status == TP_STATUS_AVAILABLE) {
        memcpy(data, pkt_data, pkt_len);
        hdr->tp_len = pkt_len;
        hdr->tp_status = TP_STATUS_SEND_REQUEST;

        // notify kernel to sending the packet
        // you can also call the send after putting multiple packets to ring buffer
        send(socket_fd, NULL, 0, 0);
        
        break;
    }
}
{% endhighlight %}

You may want to aggressively exploit the transmission speed and reduce the latency as much as possible like packet generator software usually does, then the option `PACKET_QDISC_BYPASS` comes to rescure. You can set this option after socket created.

{% highlight c %}
int on = 1;
setsockopt(fd, SOL_PACKET, PACKET_QDISC_BYPASS, &on, sizeof(on));
{% endhighlight %}


The side effect of this option is AF_PACKET will bypass the kernel's qdisc layer and forcedly push packets to the driver directly. That means, the packets are not buffered and no TC disciplines are applied, and hence potentially increasing the loss in present of microburst. Generally, this option could be used for stress performance testing or in scenario where you really don't care too much of packet loss.

## The Golang Implementation

The [github.com/google/gopacket](gopacket) package provides a golang implementation of the three versions of TPacket's for AF_PACKET. Examples are available at [https://github.com/google/gopacket/tree/master/examples/afpacket](here). However, this gopacket package doesn't implement the TX_RING, I have forked the repo and committed my TX_RING implementation at [https://github.com/csulrong/gopacket/tree/master/afpacket](here). 


## References

- [packet(7) — Linux manual page](https://man7.org/linux/man-pages/man7/packet.7.html)
- [Linux Networking Documentation - Packet MMAP](https://www.kernel.org/doc/html/latest/networking/packet_mmap.html)

