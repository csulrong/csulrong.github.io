---
title: '内存寻址(1): X86微处理器的内存寻址原理'
date: 2017-03-26 08:52:53 +0800
author: csulrong
version: 3.4.3
categories: [linux]
tags: [Linux]
excerpt_separator: <!-- more -->
---

应用程序在运行过程中会频繁地访问内存中的某段地址并从中存取数据。那么内存寻址是怎么实现的呢？现代的很多高级编程语言避免了让程序员直接操纵内存单元，给编程工作带来了极大的简便。但是，了解了这些底层机制之后，对于理解编程语言与操作系统交互时的更深层次机理不无裨益，有助于写出更优秀性能的应用程序代码。这里，将通过一系列文章来揭示内存寻址的底层实现。本文是该系列文章的开篇，重点在于介绍X86微处理是如何实现内存寻址的。

<!-- more -->

当使用80x86微处理器时，必须区分以下三种不同类型的地址：

- `逻辑地址(Logical Address)`: 每个逻辑地址都有一个`段(segment)`和`偏移量(offset)`，偏移量指明了从段起始的地方到实际地址之间的距离。
- `线性地址(Linear Address)`或也称为`虚拟地址(Virtual Address)`: 32位无符号整数来表示的高达**4GB**的地址。
- `物理地址(Physical Address)`: 用于内存芯片级内存单元寻址，和从CPU地址引脚发送到内存总线上的电信号相对应。物理地址由32位或36位无符号数整数表示。

`逻辑地址`到`物理地址`的转换，需要经过内存控制单元(MMU)的`分段单元`和`分页单元`的硬件电路(如下图)。

{: .align-center}
![从逻辑地址到物理地址的转换过程]({{ site.baseurl }}/assets/images/2017/transform_address.svg)

### x86微处理器中的分段

一个逻辑地址由两部分组成:

- `段选择符(segment selector)`: 长度为16bit，如下图所示，后面会详细说明各个字段的用途。
- `偏移量`: 相对于段起始地址的偏移量，长度为32bit。

{: .align-center}
![段选择符]({{ site.baseurl }}/assets/images/2017/segment_selector.svg)

CPU提供段寄存器用于存放段选择符，这些段寄存器有6个：`cs(代码段寄存器)`、`ss(栈段寄存器)`、`ds(数据段寄存器)`、`es(附加数据段寄存器)`、`fs`和`gs`。`cs`寄存器还包含一个2bit的字段，用于指明CPU的`当前特权级别(Current Privilege Level, CPL)`，值为0代表最高优先级，值为3代表最低优先级。Linux只使用了0级和3级，分别对应`内核态`和`用户态`。

每个段由一个8字节的段描述符表示，它描述了段的特征。段描述符放在`全局描述符表(Global Descriptor Table, GDT)`或`局部描述符表(Local Descriptor Table, LDT)`。GDT在内存中的基地址和大小存放在`gdtr`控制寄存器中，当前正在使用的LDT基地址和大小存放在`ldtr`控制寄存器中。

<div class="note info">
  <p>通常只定义一个GDT，而每个进程除了存放在GDT中的段之外，如果还需要创建附加的段，就可以有自己的LDT。</p>
</div>

段描述符的格式见下图：

{: .align-center}
![数据段描述符和代码段描述符的格式]({{ site.baseurl }}/assets/images/2017/segment_descriptor.svg)

数据段描述符和代码段描述符的格式基本类似，以下是对其中一些重要字段的说明：

<div class="mobile-side-scroller">
<table>
  <thead>
    <tr>
      <th>字段名</th>
      <th>描述</th>
    </tr>
  </thead>
  <tbody>
    <tr class="setting">
      <td>
        <p class="description">Base</p>
      </td>
      <td>
        <p class="description">32bit，包含段的首字节的线性地址</p>
      </td>
    </tr>
    <tr class="setting">
      <td>
        <p class="description">G</p>
      </td>
      <td>
        <p class="description">粒度标志，该位清0，则段大小以字节为单位；否则以4KB的倍数计</p>
      </td>
    </tr>
    <tr class="setting">
      <td>
        <p class="description">Limit</p>
      </td>
      <td>
        <p class="description">20bit，决定段的长度，G标志位清0，段的大小在1个字节到1MB之间；否则将在4KB到4GB之间</p>
      </td>
    </tr>
    <tr class="setting">
      <td>
        <p class="description">S</p>
      </td>
      <td>
        <p class="description">系统标志位，该位清0，则说明这是一个系统段。数据段描述符和代码段描述符都将该位置1(非系统段)</p>
      </td>
    </tr>
    <tr class="setting">
      <td>
        <p class="description">Type</p>
      </td>
      <td>
        <p class="description">描述了段的类型特征和它的存取权限(请看表下面的描述)</p>
      </td>
    </tr>
    <tr class="setting">
      <td>
        <p class="description">DPL</p>
      </td>
      <td>
        <p class="description">
        描述符特权级别(Descriptor Privilege Level)，它表示为访问这个段而要求的CPU最小的优先级。
        因此，DPL设为0的段只能当CPL为0时(内核态)才是可访问的，而DPL设为3的段对任何CPL值都是可访问的
        </p>
      </td>
    </tr>
    <tr class="setting">
      <td>
        <p class="description">P</p>
      </td>
      <td>
        <p class="description">
        存在位标志，该位清0，则表示当前段不在主存中。Linux总是将该位设为1，因为它永远不会把整个段交换到磁盘上去
        </p>
      </td>
    </tr>
    <tr class="setting">
      <td>
        <p class="description">D或B</p>
      </td>
      <td>
        <p class="description">
        取决于是数据段还是代码段。D或B的含义在两种情况下稍微有所区别，但是如果段偏移量的地址是32位长，
        就基本上把它置为1，如果这个偏移量是16位长，它被清0(更详细的说明参见Intel使用手册)
        </p>
      </td>
    </tr>
    <tr>
      <td>
        <p class='description'>AVL</p>
      </td>
      <td>
        <p class='description'>可以由操作系统使用，但是被Linux忽略</p>
      </td>
    </tr>
  </tbody>
</table>
</div>


有了以上这些对段的基本概念的说明之后，我们来看看逻辑地址是如何转换到线性地址的(见下图)。

{: .align-center}
![逻辑地址到线性地址的转换]({{ site.baseurl }}/assets/images/2017/logical2linear_address.svg)

- 首先，从段寄存器中获取段选择符
- 检查段选择符中的TI字段，决定是从GDT中还是从LDT中读取段描述符，GDT和LDT的起始地址分别存于gdtr和ldtr寄存器中
- 段选择符中的索引号字段的值乘以8(因为段描述符大小为8个字节)再和gdtr或ldtr寄存器中的内容相加，得到了段描述符在GDT或LDT中的真实地址
- 段描述符中Base字段的值和逻辑地址的偏移量相加就得到了线性地址

### x86微处理中的分页

分页单元把线性地址转换成物理地址。线性地址被分成以固定长度为单位的`页(page)`。页内部连续的线性地址被映射到连续的物理地址中，这样，内核可以指定一个页的物理地址和其存取权限，而不用指定页所包含的全部线性地址的存取权限。遵照习惯说法，术语“页”既指一组线性地址，又指包含在这组地址中的数据。

把线性地址映射到物理地址的数据结构称为`页表(page table)`。页表存放在主存中，并在启用分页单元之前必须由内核对页表进行适当的初始化。80x86处理器通过设置`cr0寄存器`的`PG`标志启用分页，当`PG=0`时，线性地址就被直接解释成物理地址。

#### 常规分页

从80386起，Intel处理器的分页单元处理4KB的页。32位的线性地址被分成3个域：
- `目录(directory)`: 最高10位
- `页表(table)`: 中间10位
- `偏移量(offset)`: 最低12位

线性地址的转换分两步完成: 第一步基于页目录表，第二步基于页表，如下图所示。

<div class="note info">
  <p>
  使用两级转换表的好处是可以减少每个进程页表所需RAM的数量。如果只使用简单的一级页表，那将需要高达\( 2^{20} \)个表项，也就是说，在每项4个字节时，需要4MB RAM来表示每个进程的页表(如果进程使用全部4GB线性地址空间的话)。
  </p>
</div>

{: .align-center}
![80x86处理器的分页]({{site.baseurl}}/assets/images/2017/linear2real_address.svg)

每个活动进程必须有一个分配给它的页目录，不过只有在进程实际需要一个页表的时候才会给该页表分配RAM。当前正在使用的页目录表的起始地址存放在`cr3控制寄存器`中。线性地址中的`目录`字段决定页目录中的目录项，而目录项指向适当的页表。线性地址中的`页表`字段又决定页表中的表项，而表项含有页所在页框的物理地址。`偏移量`字段决定页框内的相对位置，由于它的长度是12位，故每一页含有4KB字节的数据。

页目录项和页表项有相同的结构，每项都包含如下一些字段：

<div class="mobile-side-scroller">
<table>
  <thead>
    <tr>
      <th>字段名</th>
      <th>描述</th>
    </tr>
  </thead>
  <tbody>
    <tr class="setting">
      <td>
        <p class="description">Present</p>
      </td>
      <td>
        <p class="description">
        该标志位置1，则说明所指的页(或页表)在主存中。当执行地址转换所需的页表项或页目录项中的Present
        标志被清0时，分页单元会把该线性地址存放到<code>cr2控制寄存器</code>中，并产生14号异常:
        <code>缺页异常</code>。
        </p>
      </td>
    </tr>
    <tr class="setting">
      <td>
        <p class="description">Base</p>
      </td>
      <td>
        <p class="description">长度为20bit，等于页框物理地址的最高20位。</p>
      </td>
    </tr>
    <tr class="setting">
      <td>
        <p class="description">Accessed</p>
      </td>
      <td>
        <p class="description">
        每当分页单元对相应页框进行寻址时就设置这个标志位。当选中的页被交换出去时，这一标志就可以
        由操作系统使用。分页单元从来不重置该标志，必须由操作系统去做。
        </p>
      </td>
    </tr>
    <tr class="setting">
      <td>
        <p class="description">Dirty</p>
      </td>
      <td>
        <p class="description">
        只应用于页表项中。每当对一个页框进行写操作时就设置这个标志。和<code>Accessed</code>
        标志一样，当选中的页被交换出去时，这一标志就可以由操作系统使用，分页单元不重置这个标志。
        </p>
      </td>
    </tr>
    <tr class="setting">
      <td>
        <p class="description">Read/Write</p>
      </td>
      <td>
        <p class="description">页或页表的存取权限标志位(参见<code><a href="#分页单元的特权保护方案">分页单元的特权保护方案</a></code>)</p>
      </td>
    </tr>
    <tr class="setting">
      <td>
        <p class="description">User/Supervisor</p>
      </td>
      <td>
        <p class="description">
        含有访问页或页表所需的特权级别(参见<code><a href="#分页单元的特权保护方案">分页单元的特权保护方案</a></code>)
        </p>
      </td>
    </tr>
    <tr class="setting">
      <td>
        <p class="description">PCD和PWT</p>
      </td>
      <td>
        <p class="description">
        控制硬件高速缓存处理页或页表的方式。
        </p>
      </td>
    </tr>
    <tr class="setting">
      <td>
        <p class="description">Page Size</p>
      </td>
      <td>
        <p class="description">
        只应用于页目录项，如果设置为1，则页目录项指的是2MB或4MB的页框(参见下一节
        <code><a href="#扩展分页">扩展分页</a></code>)
        </p>
      </td>
    </tr>
    <tr>
      <td>
        <p class='description'>Global</p>
      </td>
      <td>
        <p class='description'>
        只应用于页表项，用来防止常用页从TLB(translation lookaside buffer)高速缓存中刷新
        出去。只有在<code>cr4寄存器</code>的<code>页全局启用(PGE)</code>标志置位时，这个
        标志才起作用。
        </p>
      </td>
    </tr>
  </tbody>
</table>
</div>

### 小结

本文基于80x86对硬件的寻址方案进行了详细的说明，具体描述了分段单元如何将逻辑地址转换成线性地址，以及分页单元进而把线性地址转换成物理地址的细节。然而，针对分页单元，x86还有更多高级的支持，由于篇幅限制，将在下一篇文章中具体介绍分页单元中的高级特性，包括扩展分页、特权保护方案、高速缓存、TLB等。

-----------------

<div>
<a href="{% post_url 2017-04-09-linux-mem-address-02 %}">内存寻址(2): X86微处理器的分页单元的高级话题</a>
</div>
