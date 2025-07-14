# SipStack

## 一、什么是 SipStack

`SipStack` 是 **SIP 协议栈（SIP Protocol Stack）** 的核心类，用于构建和管理 SIP 协议的完整通信机制。
 它是 **整个 SIP 协议实现的核心调度器与上下文容器**，你可以把它看成是 SIP 服务的“大脑”：

- 负责维护：
  - **消息传输层（UDP/TCP）**
  - **事务管理（Transaction Layer）**
  - **会话/对话（Dialog）**
  - **路由与分发机制**
- 通过它你可以：
  - 创建 ListeningPoint（监听端口）
  - 创建 SipProvider（事件监听与发起者）
  - 注册 SipListener（处理请求/响应）

在 Java 的 JAIN SIP 实现中，它是通过 `javax.sip.SipStack` 这个接口暴露出来的。

------

## 二、SipStack`的构建流程

用 JAIN-SIP 创建一个 `SipStack` 的标准步骤如下：

```java
SipFactory sipFactory = SipFactory.getInstance();
sipFactory.setPathName("gov.nist"); // 使用 NIST 实现

Properties props = new Properties();
props.setProperty("javax.sip.STACK_NAME", "MySipStack");
props.setProperty("javax.sip.IP_ADDRESS", "192.168.1.100");
props.setProperty("gov.nist.javax.sip.TRACE_LEVEL", "32");
props.setProperty("gov.nist.javax.sip.SERVER_LOG", "sip-server.log");

SipStack sipStack = sipFactory.createSipStack(props);
```

之后你通常还需要：

```java
ListeningPoint lp = sipStack.createListeningPoint("192.168.1.100", 5060, "udp");
SipProvider provider = sipStack.createSipProvider(lp);
provider.addSipListener(myListener); // 注册你的 SIP 消息监听器
```

------

## 三、SipStack在系统中扮演什么角色

从架构上看，它大致负责以下几个层次：

```
┌─────────────────────────────┐
│        应用层业务逻辑         │
└─────────────────────────────┘
            ▲
            │  通过 SipListener 收发事件
            ▼
┌─────────────────────────────┐
│        SipProvider          │ ← 注册在 SipStack 中
│        （事件调度）           │
└─────────────────────────────┘
            ▲
            │ 绑定监听端口
            ▼
┌─────────────────────────────┐
│      ListeningPoint         │ ← 监听 5060 端口（UDP 或 TCP）
└─────────────────────────────┘
            ▲
            │ SIP 报文收发
            ▼
┌─────────────────────────────┐
│          SipStack           │ ← SIP 协议栈：维护事务、对话、传输、路由等
└─────────────────────────────┘
```

------

## 四、常见配置项说明（JAIN SIP）

| 配置项                                  | 含义                        |
| --------------------------------------- | --------------------------- |
| `javax.sip.STACK_NAME`                  | 必须项，SIP栈的名称         |
| `javax.sip.IP_ADDRESS`                  | SIP栈绑定的本地IP地址       |
| `javax.sip.ROUTER_PATH`                 | （可选）自定义 SIP 路由类   |
| `gov.nist.javax.sip.DEBUG_LOG`          | 调试日志输出路径            |
| `gov.nist.javax.sip.TRACE_LEVEL`        | 日志级别（0=OFF，32=FULL）  |
| `gov.nist.javax.sip.REENTRANT_LISTENER` | 是否支持多线程处理 SIP 消息 |

------

## 五、和 Spring Boot 如何配合

你可以在 Spring Boot 中将 `SipStack` 作为一个 `@Bean` 注入，便于统一配置、管理与生命周期控制：

```java
@Configuration
public class SipConfiguration {

    @Bean
    public SipStack sipStack() throws PeerUnavailableException {
        SipFactory sipFactory = SipFactory.getInstance();
        sipFactory.setPathName("gov.nist");

        Properties props = new Properties();
        props.setProperty("javax.sip.STACK_NAME", "GB28181-Stack");
        props.setProperty("javax.sip.IP_ADDRESS", "192.168.1.100");
        props.setProperty("gov.nist.javax.sip.TRACE_LEVEL", "16");

        return sipFactory.createSipStack(props);
    }
}
```

还可以通过 `@PreDestroy` 来优雅地关闭 `sipStack.stop()`，避免内存泄露。