---
layout: post
title: Introduction to P4 Progamming Language
date: 2017-07-07 15:04:00 +08:00
tags: Introduction, P4, Programming Language
---

写在前面：这篇文章的内容都是来源于[官网](p4.org)，[Tutorial](https://github.com/p4lang/tutorials/blob/master/SIGCOMM_2016/p4-tutorial-slides.pdf), [Spec](https://p4lang.github.io/p4-spec/)等等公开的资料，欢迎分享，并注明出处

### **what is P4**
P4 is short for [Programming Protocol-Independent Packet Processors](http://www.sigcomm.org/sites/default/files/ccr/papers/2014/July/0000000-0000004.pdf)，它是一门编程语言，详细的介绍可以看链接的paper。它是运行在switch上面的一种language，能够让switch像小型电脑一样，按照user的意愿处理packet。它有以下特点：

* **Protocol Independent**: switch事前不需要任何protocol的知识，完全是user自己定义的，也就是说自己想写一个新的protocol完全没问题(如果完全不考虑switch以外的任何因素
* **Target Independent**: 完全不用考虑switch硬件，就像只要是x86电脑装了VS，写个c++然后compile一下就能运行，而不需要知道CPU怎么运作的。因为不只是用在switch，还可以用在很多硬件上，所以官网称这些硬件为target
* **Field Reconfigurable** 其实就是programmable，可以重新编写对packet的parsing和processing  

### **why P4**
目的就是解决目前OpenFlow的痛点:
* Central controller is not scalable, 因为所有packet in都传到了central controller。但是如果switch能够简单处理packet，就能减少很多packet in
* OpenFlow每一种protocol/Header Fields都需要事先写好，随着支援的protocol/Header Fields原来越多，体量只会越来越大，但是很多时候要控制的网络可能只需要用到一部分protocol/Header Fields。openFlow从2009年支援12种Header Fields，到2013年增加到了41种
* P4和OpenFlow是相辅相成的，相当于将部分智慧从central controller拉回到switch。它们并不是互相排斥关系，而是可以同时使用。

目前来看，只要是p4没有定义的protocol，即使OpenFlow支援也不能使用。

### **Example: Very Simple Switch**
Very Simple Switch的architecture长这样:  
#

<img src="/assets/images/2017/Very-Simple-Switch-Architecture.PNG" width="80%" />

#

1. 每个target的input方式有三种，**packet in**, **CPU** (central controller), **Recirculate** (从output port重新导回来的)，output方式也是一样
2. 白色的部分都是需要user自己定义的，包括**parser**, **Match-action pipeline**, **Deparser**。因为parser决定了如何去拆解packet，而这部分是user写的，也就是说user决定了要用什么protocol，甚至想创造出什么的protocol(不考虑switch以外的因素)  
3. 浅灰色的部分是fixed-function blocks，也就是不用user写的，包括**Arbiter**, **parser runtime**, **Demux**
,下面介绍他们的功能，我直接copy Spec里面的介绍，英文不算难，就不翻译了

##### **Arbiter**
>* It receives packets from one of the physical input Ethernet ports, from the control plane, or from
the input recirculation port.
>* For packets received from Ethernet ports, the block computes the Ethernet trailer checksum and
verifies it. If the checksum does not match, the packet is discarded. If the checksum does match,
it is removed from the packet payload.
>* Receiving a packet involves running an arbitration algorithm if multiple packets are available.
>* If the arbiter block is busy processing a previous packet and no queue space is available, input
ports may drop arriving packets, without indicating the fact that the packets were dropped in any
way.
>* After receiving a packet, the arbiter block sets the inCtrl.inputPort value that is an input to the
parser with the identity of the input port where the packet originated. Physical Ethernet ports are
numbered 0 to 7, while the input recirculation port has a number 13 and the CPU port has the
number 14.

##### **parser runtime**
> The parser runtime block works in concert with the parser. It provides an error code to the matchaction
pipeline, based on the parser actions, and it provides information about the packet payload
(e.g., the size of the remaining payload data) to the demux block. As soon as a packet's processing is
completed by the parser, the match-action pipeline is invoked with the associated metadata as inputs
(packet headers and user-defined metadata).  
>The run-time system contains implementations of basic low-level commands and may also implement higher-level commands and may support type checking, debugging, and even code generation and optimization.

其实就是辅助parser运行的

##### **Demux**
>The core functionality of the demux block is to receive the headers for the outgoing packet from the
deparser and the packet payload from the parser, to assemble them into a new packet and to send the
result to the correct output port. The output port is specified by the value of outCtrl.ouputPort, which
is set by the match-action pipeline.
>* Sending the packet to the drop port causes the packet to disappear.
>* Sending the packet to an output Ethernet port numbered between 0 and 7 causes it to be emitted
on the corresponding physical interface. The packet may be placed in a queue if the output interface
is already busy emitting another packet. When the packet is emitted, the physical interface
computes a correct Ethernet checksum trailer and appends it to the packet.
>* Sending a packet to the output CPU port causes the packet to be transferred to the control plane.
In this case, the packet that is sent to the CPU is the original input packet, and not the packet
received from the deparser—the latter packet is discarded.
>* Sending the packet to the output recirculation port causes it to appear at the input recirculation
port. Recirculation is useful when packet processing cannot be completed in a single pass.
>* If the outputPort has an illegal value (e.g., 9), the packet is dropped.
>* Finally, if the demux unit is busy processing a previous packet and there is no capacity to queue
the packet coming from the deparser, the demux unit may drop the packet, irrespective of the
output port indicated.

### **Very Simple Switch开发思路**
其实思路很简单，既然只有三个module是可以供user开发的,那个user只需要分别定义好三个module，然后把他们串联起来就完成开发了，是不是很简单lol，下面我们先看例子

### **Very Simple Switch框架**
```c++
// File "very_simple_switch_model.p4"，扩展名是p4
// Very Simple Switch P4 declaration
// core library needed for packet_in and packet_out definitions
# include <core.p4>

/* Various constants and structure declarations */
/* ports are represented using 4-bit values */
typedef bit<4> PortId;

/* only 8 ports are "real" */
const PortId REAL_PORT_COUNT = 4w8; // 4w8 is the unsigned number 8 in 4 bits, w代表unsigned, 只有8个有效port，目前还不知道为什么这么少port，先留个坑，以后来解答

/* metadata accompanying an input packet */
struct InControl {
  PortId inputPort; //利用了刚定义的PortId
}

/* special input port values */
const PortId RECIRCULATE_IN_PORT = 0xD; // 13 for recirculation
const PortId CPU_IN_PORT = 0xE; // 14 for CPU， 也就是controller

/* metadata that must be computed for outgoing packets */
struct OutControl {
  PortId outputPort; //利用刚才定义的PortId
}

/* special output port values for outgoing packet */
const PortId DROP_PORT = 0xF; //14 for drop
const PortId CPU_OUT_PORT = 0xE;
const PortId RECIRCULATE_OUT_PORT = 0xD;


/* Prototypes for all programmable blocks */
/**
* Programmable parser.
* @param <H> type of headers; defined by user
* @param b input packet
* @param parsedHeaders headers constructed by parser
*/
parser Parser<H>(packet_in b, out H parsedHeaders); // 类型packet_in的定义在core.p4, output的类型是自定义的H，关键字out表示变量parsedHeaders还没初始化，而且可以被改变值


/**
* Match-action pipeline
* @param <H> type of input and output headers
* @param headers headers received from the parser and sent to the deparser
* @param parseError error that may have surfaced during parsing
* @param inCtrl information from architecture, accompanying input packet
* @param outCtrl information for architecture, accompanying output packet
*/
control Pipe<H>(inout H headers, // inout表示既是input类型也是output类型
in error parseError,// parser error， 来自parser runtime的error input
in InControl inCtrl, // input port
out OutControl outCtrl); // output port


/**
* VSS deparser.
* @param <H> type of headers; defined by user
* @param b output packet
* @param outputHeaders headers for output packet
*/
control Deparser<H>(inout H outputHeaders,  packet_out b); //packet_out类型的定义同样在core.p4


/**
* Top-level package declaration - must be instantiated by user.
* The arguments to the package indicate blocks that
* must be instantiated by the user.
* @param <H> user-defined type of the headers processed.
*/
package VSS<H>(Parser<H> p, Pipe<H> map, Deparser<H> d); //将三个user self-defined的module串联起来就是我们的Very Simple Switch(VSS)咯

// Architecture-specific objects that can be instantiated，一个可以供这个architecture调用的external block
// Checksum unit
extern Checksum16 {
  Checksum16(); // constructor
  void clear(); // prepare unit for computation，初始化checksum unit
  void update<T>(in T data); // add data to checksum,往unit里面添加数据
  void remove<T>(in T data); // remove data from existing checksum，从unit里面删掉数据
  bit<16> get(); // get the checksum for the data added since last clear，执行checksum获得结果
}
```

#### **Very Simple Switch完整代码**
####
代码的处理流程如下:  
<img src="/assets/images/2017/match-action_pipeline_expressed_by_VSS.png" width="80%" />  
####

可以看到这里没有使用到recirculation功能。首先parser会识别**Ethernet**和**IPv4** header,如果其中一个除了问题，就会报错，然后drop。没错误的话就会把header抽出来放到**Parsed_packet**,然后再进行一系列的match-action：
>* If any parser error has occurred, the packet is dropped (i.e., by assigning outputPort to DROP_PORT)
>* The first table uses the IPv4 destination address to determine the outputPort and the IPv4 address
of the next hop. If this lookup fails, the packet is dropped. The table also decrements the IPv4 ttl
value.
>* The second table checks the ttl value: if the ttl becomes 0, the packet is sent to the control plane
through the CPU port.
>* The third table uses the IPv4 address of the next hop (which was computed by the first table) to
determine the Ethernet address of the next hop.
>* Finally, the last table uses the outputPort to identify the source Ethernet address of the current
switch, which is set in the outgoing packet.

Match-ations结束之后，deparser会重新将packet组装起来并送到设定好的output port。源代码如下：

```c++
// Include P4 core library
# include <core.p4>
// Include very simple switch architecture declarations
# include "very_simple_switch_model.p4" //将我们上面写好的框架include进来

// This program processes packets comprising an Ethernet and an IPv4
// header, and it forwards packets using the destination IP address
typedef bit<48> EthernetAddress; //自定义48个bit的新的类型叫EthernetAddress
typedef bit<32> IPv4Address;    //自定义32个bit的新的类型叫IPv4Address

// Standard Ethernet header
header Ethernet_h { //user定义Ethernet_h header长什么样，也就是这里体现出Protocol-Independent的特点
  EthernetAddress dstAddr; //定义类型是EthernetAddress的变数dstAddr
  EthernetAddress srcAddr; //定义类型是EthernetAddress的变数srcAddr
  bit<16> etherType; //16位的变数
}

// IPv4 header (without options)
header IPv4_h {  //user定义IPv4_h header长什么样
  bit<4> version;
  bit<4> ihl;
  bit<8> diffserv;
  bit<16> totalLen;
  bit<16> identification;
  bit<3> flags;
  bit<13> fragOffset;
  bit<8> ttl;
  bit<8> protocol;
  bit<16> hdrChecksum;
  IPv4Address srcAddr;
  IPv4Address dstAddr;
}
// Structure of parsed headers
struct Parsed_packet { //定义将要被parse的packet的结构
  Ethernet_h ethernet; //第一部分是header类型是Ethernet_h的header ethernet
  IPv4_h ip;           //第二部分是header类型是IPv4_h的header ip
}

// Parser section
// User-defined errors that may be signaled during parsing
error {
  IPv4OptionsNotSupported,
  IPv4IncorrectVersion,
  IPv4ChecksumError
}

//这里每个state都是相当于状态机里面的某个state，只能从一个state转移到另外一个state。整个状态机的起始点是state start，然后到达state parse_ipv4，终点是accept。到达state accept就是一次成功的parse

parser TopParser(packet_in b, out Parsed_packet p) { //Parsed_packet的定义在前面
  Checksum16() ck; // instantiate checksum unit
  state start {
      b.extract(p.ethernet);
      transition select(p.ethernet.etherType) {
        0x0800: parse_ipv4;
        // no default rule: all other packets rejected
      }
  }
  state parse_ipv4 {
    b.extract(p.ip);
    verify(p.ip.version == 4w4, error.IPv4IncorrectVersion);
    verify(p.ip.ihl == 4w5, error.IPv4OptionsNotSupported);
    ck.clear();
    ck.update(p.ip);
    // Verify that packet checksum is zero
    verify(ck.get() == 16w0, error.IPv4ChecksumError);
    transition accept;
  }
}

// Match-action pipeline section，开始定义pipe
control TopPipe(inout Parsed_packet headers, //Parsed_packet的定义在前面，TopParser的output就是这里的input
in error parseError, // parser error，由parser runtime提供
in InControl inCtrl, // input port
out OutControl outCtrl)/*output port，还没初始化*/ {

IPv4Address nextHop; // local variable

/**
* Indicates that a packet is dropped by setting the
* output port to the DROP_PORT
*/
action Drop_action() { //这里的keyword action 对应的是OpenFlow里面的action，会改动header里面的内容
  outCtrl.outputPort = DROP_PORT;
}

/**
* Set the next hop and the output port.
* Decrements ipv4 ttl field.
* @param ivp4_dest ipv4 address of next hop
* @param port output port
*/
action Set_nhop(IPv4Address ipv4_dest, PortId port) {
  nextHop = ipv4_dest;
  headers.ip.ttl = headers.ip.ttl - 1;
  outCtrl.outputPort = port;
}
/**
* Computes address of next IPv4 hop and output port
* based on the IPv4 destination of the current packet.
* Decrements packet IPv4 TTL.
* @param nextHop IPv4 address of next hop
*/
table ipv4_match {
  key = { headers.ip.dstAddr: lpm; } // longest-prefix match
  actions = {
    Drop_action;
    Set_nhop;
  }
  size = 1024;
  default_action = Drop_action;
}

/**
* Send the packet to the CPU port
*/
action Send_to_cpu() {
  outCtrl.outputPort = CPU_OUT_PORT;
}

/**
* Check packet TTL and send to CPU if expired.
*/
table check_ttl {
  key = { headers.ip.ttl: exact; }
  actions = { Send_to_cpu; NoAction; }
  const default_action = NoAction; // defined in core.p4
}

/**
* Set the destination MAC address of the packet
* @param dmac destination MAC address.
*/
action Set_dmac(EthernetAddress dmac) {
  headers.ethernet.dstAddr = dmac;
}

/**
* Set the destination Ethernet address of the packet
* based on the next hop IP address.
* @param nextHop IPv4 address of next hop.
*/
table dmac {
  key = { nextHop: exact; }
  actions = {
    Drop_action;
    Set_dmac;
  }
  size = 1024;
  default_action = Drop_action;
}

/**
* Set the source MAC address.
* @param smac: source MAC address to use
*/
action Set_smac(EthernetAddress smac) {
  headers.ethernet.srcAddr = smac;
}

/**
* Set the source mac address based on the output port.
*/
table smac {
  key = { outCtrl.outputPort: exact; }
  actions = {
    Drop_action;
    Set_smac;
  }
  size = 16;
  default_action = Drop_action;
}

apply {
  if (parseError != error.NoError) {
    Drop_action(); // invoke drop directly
    return;
  }
  ipv4_match.apply(); // Match result will go into nextHop
  if (outCtrl.outputPort == DROP_PORT) return;
  check_ttl.apply();
  if (outCtrl.outputPort == CPU_OUT_PORT) return;
  dmac.apply();
  if (outCtrl.outputPort == DROP_PORT) return;
  smac.apply();
  }
}

// deparser section
control TopDeparser(inout Parsed_packet p, packet_out b) {
  Checksum16() ck;
  apply {
    b.emit(p.ethernet);
    if (p.ip.isValid()) {
      ck.clear(); // prepare checksum unit
      p.ip.hdrChecksum = 16w0; // clear checksum
      ck.update(p.ip); // compute new checksum.
      p.ip.hdrChecksum = ck.get();
    }
    b.emit(p.ip);
  }
}

// Instantiate the top-level VSS package
VSS(TopParser(), TopPipe(), TopDeparser()) main;
```
