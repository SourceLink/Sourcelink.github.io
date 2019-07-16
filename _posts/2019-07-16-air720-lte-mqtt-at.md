---
layout: post
title:  "合宙Air720_MQTT概述"
date:   2019-07-16 15:24:56
catalog:  true
author: Sourcelink
tags:
    - MQTT
    - 4G模块

---


# 一. 概述


在使用合宙4G模块时, 发现该模块内嵌了MQTT功能, 在此结合MQTT协议对相应的AT指令解析;



# 二. AT指令参数详解


## 2.1  AT+MCONFIG

<table>
   <tr>
      <td>命令类型</td>
      <td>语法</td>
      <td>返回</td>
      <td>说明</td>
   </tr>
   <tr>
      <td rowspan="2">设置命令</td>
      <td rowspan="2" > AT+MCONFIG=&ltclientid&gt[,&ltusername &gt,&ltpassword&gt[,&ltwill_qos&gt,&ltwill_retain&gt,&ltwill_topic&gt,&ltwill_message&gt]] </td>
      <td>OK</td>
      <td>正常返回</td>
   </tr>
   <tr>
   <td>ERR</td>
   <td>输入格式有误</td>
   </tr>
   <tr>
      <td> 测试命令</td>
      <td>  AT+MCONFIG=?</td>
      <td colspan="2">+MCONFIG:
           &ltclientid&gt[,&ltusername&gt,&ltpassword&gt[,(0-2),(0,1),&ltwill_topic&gt,&ltwill_message&gt]] <br>
            OK
      </td>
   </tr>
</table>


参数详解:  

- clientid

连接服务端的每个客户端都有唯一的客户端标识符, 客户端和服务端都必须使用`ClientId` 识别两者之间的 MQTT 会话相关的状态, 一般客户端标识符是1到23字节长的大写字母,小写字母
或数字字符;

- username & password

服务端用于身份验证和授权;

- will_qos

遗嘱Qos用于指定发布遗嘱消息时的服务质量;

- will_retain

遗嘱保留, 如果该值设置为`0`则服务端必须将遗嘱消息当作非保留信息发布, 如果设置为`1`则服务端必须将遗嘱消息当作保留信息发布(当订阅者订阅主题时会立即收到保留消息);

- will_topic

遗嘱主题, 如果该客户端"挂掉", 服务端将会发布这个遗嘱主题的消息;

- will_message

遗嘱消息定义了将被发布到遗嘱主题的应用消息;


## 2.2 AT+MIPSTART

<table>
   <tr>
      <td>命令类型</td>
      <td>语法</td>
      <td>返回</td>
      <td>说明</td>
   </tr>
    <tr>
    <td rowspan="2">设置命令</td>
    <td rowspan="2">普通链接: <br>
                                    AT+MIPSTART=&ltsvraddr&gt,&ltport&gt
    </td>
    <td>OK</td>
    <td>正常返回</td>
    </tr>
    <tr>
    <td>ERROR</td>
    <td>输入格式有误</td>
    </tr>
    <tr>
    <td>测试命令</td>
    <td>AT+MIPSTART=?</td>
    <td colspan="2">+MIPSTART:"(0,255).(0,255).(0,255).(0,255)",(1-65535)<br>
           +MIPSTART:"DOMAIN NAME",(1-65535)<br>
           OK
    </td>
    </tr>
</table>

参数详解:  

- svraddr

服务端的IP地址或者域名, 我在开发中测试时使用的是开源服务器`iot.eclipse.org`;

- port

服务端开放的连接端口,`iot.eclipse.org`的开放端口为`1883`;


## 2.3 AT+MCONNECT

<table>
   <tr>
      <td>命令类型</td>
      <td>语法</td>
      <td>返回</td>
      <td>说明</td>
   </tr>
   <tr>
   <td rowspan="3">设置命令</td>
   <td rowspan="3">AT+MCONNECT=&ltclean_session&gt,&ltkeepalive&gt</td>
   </tr>
   <tr>
   <td>OK<br>
           CONNACK OK
   </td>
   <td>连接成功, 正常返回</td>
   </tr>
    <tr>
    <td>ERROR</td>
    <td>输入格式有误或连接失败</td>
    </tr>    
    <tr>
    <td rowspan="2" >测试命令</td>
    <td rowspan="2" >AT+MCONNECT=?</td>
    </tr>
    <tr>
    <td>+MCONNECT:(0-1),(1-65535) <br>
            OK
   </td>
   <td>测试命令的返回的是&ltclean_session&gt和<br>
            &ltkeepalive&gt的取值范围
    </td>
    </tr>
</table>


参数详解:  

- clean_session

如果清理会话(CleanSession)标志被设置为`0`, 服务端必须基于当前会话(使用客户端标识符识别)的状态恢复与客户端的通信。如果没有与这个客户端标识符关联的会话, 服务端必须创建一个新的会话。在连接断开之后, 当连接断开后, 客户端和服务端必须保存会话信息。当清理会话标志为`0`的会话连接断开之后,服务端必须将之后的`QoS 1`和`QoS 2`级别的消息保存为会话状态的一部分, 如果这些消息匹配断开连接时客户端的任何订阅。服务端也可以保存满足相同条件的`QoS 0`级别的消息。

如果清理会话(CleanSession)标志被设置为`1`, 客户端和服务端必须丢弃之前的任何会话并开始一个新的会话。会话仅持续和网络连接同样长的时间。与这个会话关联的状态数据不能被任何之后的会话重用。

- keepalive

保持连接时间, 单位**秒**; 它是指在客户端传输完成一个控制报文的时刻到发送下一个报文的时刻, 两者之间允许空闲的最大时间间隔; 如果保持连接的值非零, 并且服务端在一点五倍的保持连接时间内没有收到客户端的控制报文, 它必须断开客户端的网络连接, 认为网络连接已断开;



## 2.4 AT+MPUB

<table>
   <tr>
      <td>命令类型</td>
      <td>语法</td>
      <td>返回</td>
      <td>说明</td>
   </tr>
    <tr>
    <td rowspan="4">设置命令</td>
    <td rowspan="4">AT+MPUB=&lttopic&gt,&ltqos&gt,&ltretain&gt,&ltmessage&gt</td>
    <td>OK</td>
    <td>qos=0</td>
    </tr>
    <tr>
    <td>OK<br>PUBACK</td>
    <td>qos=1</td>
    </tr>
    <tr>
    <td>OK</td>
    <td>qos=2</td>
    </tr>
    <tr>
    <td>ERROR</td>
    <td>失败</td>
    </tr>
    <tr>
    <td>测试命令</td>
    <td>AT+MPUB=?</td>
    <td>+MPUB:<topic>,(0-2),(0-1),<message><br>OK</td>
    <td> </td>
    </tr>
</table>


参数详解:  

- topic

发布消息的主题

- qos

发布消息的质量等级

| Qos值 |    描述    |
| ---- | ---------- |
| 0    | 最多发送一次 |
| 1    | 至少发送一次 |
| 2    | 只发送一次   |

- retain

标志被设置为`1`, 服务端必须存储这个应用消息和它的服务质量等级(QoS), 以便它可以被分发给未来的主题名匹配的订阅者。
一个新的订阅建立时, 对每个匹配的主题名, 如果存在最近保留的消息, 它必须被发送给这个订阅者。

如果客户端发给服务端的 PUBLISH 报文的保留标志位`0`, 服务端不能存储这个消息也不能移除或替换任何现存的保留消息。

- message

发布的消息具体内容