---
title: 'Deep dive into AF_PACKET socket'
date: 2022-03-02 08:52:53 +0800
author: csulrong
version: 3.4.3
categories: [linux]
tags: [Linux]
excerpt_separator: <!-- more -->
---

AF_PACKET is a method of capturing raw packets at link layer and then handling the packets in user space processes. Applications like packet capturing and software packet procesing could use this AF_PACKET socket to receive raw packets and send the packet data from and to the wires. 

This AF_PACKET had underwent a few versions of improvement in Linux kernel since it's introduced. In this artcile, we will discuss what options could be used to improve the efficiency for capturing and sending the packet via the AF_PACKET socket. 

<!-- more -->

### plain AF_PACKET socket

The AF_PACKET socket allows user-space application to capture raw packets at link layer so that the user-space can see the packet headers 


### AF_PACKET with MMAP


### 



