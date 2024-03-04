# SpringCloudAlibaba

> 基于[【官网】](https://spring.io/projects/spring-cloud-alibaba)、《尚硅谷 SpringCloud 框架开发教程》

## 一、基本概念

**[背景 1](https://spring.io/blog/2018/12/12/spring-cloud-greenwich-rc1-available-now)**：

通过 SpringCloud 章节我们可以发现，SpringCloud 中许多组件并不是 SpringCloud 本身开发的，例如：Eureka、Hystrix、Kafka 等，SpringCloud 只提供了一种兼容以及支持，将各种组件整合起来。

但是随着 SpringCloud Netflix 项目和相应的 Starter 进入维护模式，意味着 SpringCloud 团队不会再向该模块添加新功能，只是修复 block 级别的 bug 以及安全问题，新功能就需要使用其他组件进行代替，SpringCloud Alibaba 此时便应运而生。

2018-10-31，SpringCloud Alibaba 正式入驻了 SpringCloud 官方孵化器，并发布了 0.1.0（对应 SpringBoot 1.x）和 0.2.0（对应 SpringBoot 2.x）。

---

SpringCloud Alibaba 致力于**提供微服务开发的一站式解决方案**。此项目包含开发分布式应用微服务的必需组件，方便开发者通过 SpringCloud 编程模型轻松使用这些组件来开发分布式应用服务。

依托 SpringCloud Alibaba，只需要添加一些注解和少量的配置，就可以将 SpringCloud 应用接入阿里微服务解决方案，通过**阿里中间件**来迅速搭建分布式应用系统。

提供了以下的特性（仅展示 SpringCloud 方面的技术）：

- 流量控制和服务降级：Sentinel 提供流量监控、熔断和系统自适应保护。
- 服务注册和发现：实例可以注册到 Nacos，客户端可以使用 Spring 管理的 bean 发现实例。通过 SpringCloud Netfilx 支持客户端负载均衡器 Ribbon。
- 分布式配置：使用 Nacos 作为数据存储。
- 事件驱动：构建与 SpringCloud Stream RocketMQ Binder 连接的高度可扩展的事件驱动服务。
- 消息总线：用 SpringCloud Bus RocketMQ 链接分布式系统中的节点。
- 分布式事务：Seata 是支持高性能、易用的分布式事务解决方案。

## 二、Nacos

Nacos（Dynamic **Na**ming and **Co**nfiguration **S**ervice）是一个更易于构建云原生应用的动态服务发现、配置管理和服务管理平台。

> Nacos 相当于 Eureka/Consule + Config + Bus，此外还提供 Web 控制台，我们可以通过控制台更加简单地、可视化地管理**配置和服务**。

## 三、Sentinel

Sentinel 是面向分布式、多语言异构化服务架构的**流量治理组件**，主要以流量为切入点，从流量路由、流量控制、流量整形、熔断降级、系统自适应过载保护、热点流量防护等多个维度来帮助开发者保障微服务的稳定性。

**和 Hystrix 区别如下**：

|                | Sentinel                                                   | Hystrix                 |
| -------------- | ---------------------------------------------------------- | ----------------------- |
| 隔离策略       | 信号量隔离（并发线程数限流）                               | 线程池隔离/信号量隔离   |
| 熔断降级策略   | 基于响应时间、异常比率、异常数                             | 基于异常比率            |
| 实时统计实现   | 滑动窗口（LeapArray）                                      | 滑动窗口（基于 RxJava） |
| 动态规则配置   | 支持多种数据源                                             | 支持多种数据源          |
| 扩展性         | 多个扩展点                                                 | 插件的形式              |
| 基于注解的支持 | 支持                                                       | 支持                    |
| 限流           | 基于 QPS，支持基于调用关系的限流                           | 有限的支持              |
| 流量整形       | 支持预热模式、匀速器模式、预热排队模式                     | 不支持                  |
| 系统自适应保护 | 支持（根据系统硬件自动进行服务降级）                       | 不支持                  |
| 控制台         | 提供开箱即用的控制台，可配置规则、查看秒级监控、机器发现等 | 简单的监控查看          |

> Sentinel 提供详细的[从 Hystrix 迁移到 Sentinel](https://sentinelguard.io/zh-cn/blog/guideline-migrate-from-hystrix-to-sentinel.html)方案。

## 四、Seata

Seata 是一款**开源的分布式事务解决方案**，致力于提供高性能和简单易用的分布式事务服务。Seata 将为用户提供了 AT、TCC、SAGA 和 XA 事务模式，为用户打造一站式的分布式解决方案。
