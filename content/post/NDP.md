---
title: 'NDP'
date: 2024-10-01T00:00:00+08:00
draft: false
summary: "NDP协议/原理 & odhpd基本架构"
---
## ND协议/NDP原理

[邻居发现 - S600-E V200R021C00 配置指南-IP业务 - 华为 (huawei.com)](https://support.huawei.com/enterprise/zh/doc/EDOC1100213102/a7028751)

[odhcpd 中继模式原理、局限以及解决方案 | Silent Blog (icpz.dev)](https://blog.icpz.dev/articles/notes/odhcpd-relay-mode-discuss/)

**IPv6的邻居状态**

NDP的一个重要特征是检测同一链路上以前相连通的两个节点现在是否依然连通，这是通过NUD（Neighbor Unreachability Detection，邻居不可达检测）完成的。NUD帮助维护多个邻居条目组成的邻居缓存，每个邻居都有相应的状态，状态之间可以迁移。

RFC2461中定义了5种邻居状态，分别是：

**• 未完成(Incomplete)**：表示正在解析地址，但邻居链路层地址尚未确定。

**• 可达(Reachable)**：表示地址解析成功，该邻居可达。

**• 陈旧(Stale)**：表示可达时间耗尽，未确定邻居是否可达。

**• 延迟(Delay)**：邻居可达性未知。Delay状态不是一个稳定的状态，而是一个延时等待状态。

**• 探测(Probe)**：邻居可达性未知,正在发送邻居请求探针以确认可达性.

**邻居状态迁移过程**

邻居状态的具体迁移过程如图所示：

![neigh_state][neigh_state]

下面以A、B两个邻居节点之间相互通信过程中A节点的邻居状态变化为例（假设A、B之前从未通信），说明邻居状态迁移的过程。

1、A先发送NS报文，并生成缓存条目，此时，邻居状态为Incomplete。

2、若B回复NA报文，则邻居状态由Incomplete变为Reachable，否则固定时间后邻居状态由Incomplete变为Empty，即删除表项。

3、经过邻居可达时间，邻居状态由Reachable（默认30s）变为Stale，即未知是否可达。

4、如果在Reachable状态，A收到B的非请求NA报文，且报文中携带的B的链路层地址和表项中不同，则邻居状态马上变为Stale。

5、在Stale状态若A要向B发送数据，则邻居状态由Stale变为Delay，并发送NS请求。

6、在经过一段固定时间后，邻居状态由Delay（默认5s）变为Probe（每隔1s发送一次NS报文，连续发送3次），其间若有NA应答，则邻居状态由Delay变为Reachable。

7、在Probe状态，A每隔1s发送单播NS，发送3次后，有应答则邻居状态变为Reachable，否则邻居状态变为Empty，即删除表项。

## odhcpd基本架构

源码目录结构非常清晰，见名知意

```sh
odhcpd
├── CMakeLists.txt
├── COPYING
├── README
└── src
    ├── config.c
    ├── dhcpv4.c
    ├── dhcpv4.h
    ├── dhcpv6.c
    ├── dhcpv6.h
    ├── dhcpv6-ia.c
    ├── ndp.c
    ├── netlink.c
    ├── odhcpd.c
    ├── odhcpd.h
    ├── router.c
    ├── router.h
    └── ubus.c
```

这里同时也给出一个示例配置(ra relay|ndp relay)

```sh
# cat /etc/config/dhcp
config dhcp
        option interface 'wan'
        option ifname 'wan1_4096'
        option master '1'
        option ra 'relay'
        option ndp 'relay'
        option dhcpv4 'disabled'
        option dhcpv6 'disabled'
 
config dhcp
        option interface 'lan'
        option ifname 'br0'
        # option ndproxy_slave '1'
        option ra 'relay'
        option ndp 'relay'
        option dhcpv4 'disabled'
        option dhcpv6 'disabled'
 
config odhcpd 'odhcpd'
        option legacy '0'
        option maindhcp '0'
        option leasefile '/tmp/hosts/odhcpd'
        option leasetrigger '/tmp/odhcpd-update'
        option loglevel '7'
```

由于重点是分析 RA relay 和 NPD relay，这两部分与 `route.c` 和 `ndp.c` 相关，据此去分析 `odhcpd` 的架构

首先找到整个程序的入口，在 `odhcpd.c` 的 `main` 函数中，可以看到首先调用了 `netlink_init`，当系统支持 ` ipv6` 时，会调用 `ndp_init`，最后会调用 `odhcpd_run`

```c
// odhcpd.c
int main(int argc, char **argv) {
	...
	/* 对netlink初始化
		借助libnl库，创建与netlink通信的套接字，使用了NETLINK_ROUTE协议族
		并将其加入uloop的事件循环中
	*/
	if (netlink_init())
		return 4;

    // 检查系统是否支持ipv6
	if (ipv6_enabled()) {
        // 初始化route，为netlilnk事件注册route回调
		if (router_init())
			return 4;

		if (dhcpv6_init())
			return 4;

        // 初始化ndp，为netlilnk事件注册ndp回调
		if (ndp_init())
			return 4;
	}

	...
	// 读取配置 注册事件循环
	odhcpd_run();
	return 0;
}
```

先来看一下几个 `init` 函数在做什么：

`netlink_init` 函数如下，创建了与 `netlink` 通信的套接字，记录套接字对应的描述符，设置 `netlink` 套接字的回调函数 `cb_rtnl_valid`，加入多个Netlink组接收特定类型的网络事件，注册事件。

> `odhcpd_register` 函数会将向 `uloop` 添加监听文件描述符，内部采用 `epoll` 模型，当套接字有数据可读，即调用回调函数处理

```c
// netlink.c
int netlink_init(void) {
	...

	rtnl_event.sock = create_socket(NETLINK_ROUTE);
	if (!rtnl_event.sock) {
		syslog(LOG_ERR, "Unable to open nl event socket: %m");
		goto err;
	}

	rtnl_event.ev.uloop.fd = nl_socket_get_fd(rtnl_event.sock);
	...

	// 此处设置了netlink的回调函数 cb_rtnl_valid
	nl_socket_modify_cb(rtnl_event.sock, NL_CB_VALID, NL_CB_CUSTOM,
			cb_rtnl_valid, NULL);

	/* Receive IPv4 address, IPv6 address, IPv6 routes and neighbor events */
    // 加入多个Netlink组 以接收特定类型的网络事件
	if (nl_socket_add_memberships(rtnl_event.sock, RTNLGRP_IPV4_IFADDR,
				RTNLGRP_IPV6_IFADDR, RTNLGRP_IPV6_ROUTE,
				RTNLGRP_NEIGH, RTNLGRP_LINK, 0))
		goto err;
	// 注册事件，通过uloop监听描述符
	odhcpd_register(&rtnl_event.ev);

	return 0;
	...
}
```

接下来看一下 `cb_rtnl_valid` 函数，这里接收了来自 `netlink` 的消息后，根据类型，调用对应的处理函数

```c
// netlink.c
static int cb_rtnl_valid(struct nl_msg *msg, _unused void *arg) {
	struct nlmsghdr *hdr = nlmsg_hdr(msg);
	int ret = NL_SKIP;
	bool add = false;

	switch (hdr->nlmsg_type) {
	case RTM_NEWLINK:
		ret = handle_rtm_link(hdr);
		break;

	case RTM_NEWROUTE:
		add = true;
		/* fall through */
	case RTM_DELROUTE:
		ret = handle_rtm_route(hdr, add);
		break;

	case RTM_NEWADDR:
		add = true;
		/* fall through */
	case RTM_DELADDR:
		ret = handle_rtm_addr(hdr, add);
		break;

	case RTM_NEWNEIGH:
		add = true;
		/* fall through */
	case RTM_DELNEIGH:
		ret = handle_rtm_neigh(hdr, add);
		break;

	default:
		break;
	}

	return ret;
}
```

`route_init` `ndp_init` 代码如下，可以发现都是调用 `netlink_add_netevent_handler` 函数，添加回调处理函数

这里没有贴出 `dhcpv6_init` 的代码，但可以推测也是调用 `netlink_add_netevent_handler` (不信自行验证)

```c
// router.c
static struct netevent_handler router_netevent_handler = { .cb = router_netevent_cb, };
int router_init(void) {
	...
	if (netlink_add_netevent_handler(&router_netevent_handler) < 0) {
		syslog(LOG_ERR, "Failed to add netevent handler");
		ret = -1;
	}
	...
}
```

```c
// ndp.c
static struct netevent_handler ndp_netevent_handler = { .cb = ndp_netevent_cb, };
int ndp_init(void) {
	...
	if (netlink_add_netevent_handler(&ndp_netevent_handler) < 0) {
		syslog(LOG_ERR, "Failed to add ndp netevent handler");
		ret = -1;
	}
	...
}
```

可以看到，这里是将各个 `handler` 加入到链表 `netevent_handler_list` 中了

```c
// netlink.c
int netlink_add_netevent_handler(struct netevent_handler *handler) {
	if (!handler->cb)
		return -1;

	list_add(&handler->head, &netevent_handler_list);

	return 0;
}
```

那么顺便看一下这个链表在哪里会有用，可以看到在 `call_netevent_handler_list` 函数中会调用每个 `handle` 的 `cb`，而该函数又会被多个函数调用，这里列出两个函数 `handle_rtm_route` `handle_rtm_neigh`，由此可见，在 `netlink` 中当某个事件发生时，就会触发上述添加的回调函数。是否还记得上面 `netlink_init` 中对套接字设置的回调函数 `cb_rtnl_valid` ，那里就是调用这些 `handle_xxx` 的入口。

> 比如路由或者邻居发生变化时，`handle_rtm_route` 或 `handle_rtm_neigh` 会调用 `call_netevent_handler_list` ，然后继续调用到 `handle->cb` 函数，即调用了 `router_netevent_cb` `ndp_netevent_cb`

```c
// netlink.c
static void call_netevent_handler_list(unsigned long event, struct netevent_handler_info *info) {
	struct netevent_handler *handler;

	list_for_each_entry(handler, &netevent_handler_list, head)
		handler->cb(event, info);
}

static int handle_rtm_route(struct nlmsghdr *hdr, bool add) {
    ...
    avl_for_each_element(&interfaces, iface, avl) {
        ...
        call_netevent_handler_list(add ? NETEV_ROUTE6_ADD : NETEV_ROUTE6_DEL, &event_info);
        ...
    }
    ...
}

static int handle_rtm_neigh(struct nlmsghdr *hdr, bool add) {
    ...
    avl_for_each_element(&interfaces, iface, avl) {
        ...
        call_netevent_handler_list(add ? NETEV_ROUTE6_ADD : NETEV_ROUTE6_DEL, &event_info);
        ...
    }
    ...
}
```

接下来在 `config.c` 的 `odhcpd_run` 中，下边调用了 `odhcp_reload`，该函数从 `/etc/config/dhcp` 读取配置完成配置初始化，最后遍历每一个 `interface` 重新加载服务，这里的每一个 `interface` 即对应了配置文件中的 `config/dhcp` 配置节，具体的接口就是配置项 `ifname`

```c
// config.c
void odhcpd_reload(void) {
	...
	if (!uci_load(uci, "dhcp", &dhcp)) {
		...
		/* 1. Global settings */
		uci_foreach_element(&dhcp->sections, e) {
			struct uci_section *s = uci_to_section(e);
			if (!strcmp(s->type, "odhcpd"))
				set_config(s);
		}

		/* 2. DHCP pools */
		uci_foreach_element(&dhcp->sections, e) {
			struct uci_section *s = uci_to_section(e);
			if (!strcmp(s->type, "dhcp"))
				set_interface(s);
		}
		...
	}
	...

	avl_for_each_element_safe(&interfaces, i, avl, tmp) {
		...
			reload_services(i);
		} else
			close_interface(i);
	}
	...
}
```

在 `reload_services` 函数中，对各个接口上的功能进行了设置

```c
// config.c
void reload_services(struct interface *iface) {
	if (iface->ifflags & IFF_RUNNING) {
		syslog(LOG_DEBUG, "Enabling services with %s running", iface->ifname);
		router_setup_interface(iface, iface->ra != MODE_DISABLED);
		dhcpv6_setup_interface(iface, iface->dhcpv6 != MODE_DISABLED);
		ndp_setup_interface(iface, iface->ndp != MODE_DISABLED);
		...
	} else {
		...
	}
}
```

这里还是看和 `router/ndp` 相关的两个函数 `router_setup_interface` `ndp_setup_interface`，这里两个函数都是创建套接字，然后设置选项，设置过滤哪些类型的数据，设置套接字来数据时的回调函数，将套接字对应的文件描述符加入 `uloop` 事件循环

```c
// router.c
int router_setup_interface(struct interface *iface, bool enable) {
	...

	if (!enable && iface->router_event.uloop.fd >= 0) {
		...
	} else if (enable) {
		...
		if (iface->router_event.uloop.fd < 0) {
			/* Open ICMPv6 socket */
			iface->router_event.uloop.fd = socket(AF_INET6, SOCK_RAW | SOCK_CLOEXEC,
								IPPROTO_ICMPV6);
			if (iface->router_event.uloop.fd < 0) {
				...
			}

			if (setsockopt(iface->router_event.uloop.fd, SOL_SOCKET, SO_BINDTODEVICE,
						iface->ifname, strlen(iface->ifname)) < 0) {
				...
			}

			/* Let the kernel compute our checksums */
			if (setsockopt(iface->router_event.uloop.fd, IPPROTO_RAW, IPV6_CHECKSUM,
						&val, sizeof(val)) < 0) {
				...
			}

			/* This is required by RFC 4861 */
			val = 255;
			if (setsockopt(iface->router_event.uloop.fd, IPPROTO_IPV6, IPV6_MULTICAST_HOPS,
						&val, sizeof(val)) < 0) {
				...
			}

			if (setsockopt(iface->router_event.uloop.fd, IPPROTO_IPV6, IPV6_UNICAST_HOPS,
						&val, sizeof(val)) < 0) {
				...
			}

			/* We need to know the source interface */
			val = 1;
			if (setsockopt(iface->router_event.uloop.fd, IPPROTO_IPV6, IPV6_RECVPKTINFO,
						&val, sizeof(val)) < 0) {
				...
			}

			if (setsockopt(iface->router_event.uloop.fd, IPPROTO_IPV6, IPV6_RECVHOPLIMIT,
						&val, sizeof(val)) < 0) {
				...
			}

			/* Don't loop back */
			val = 0;
			if (setsockopt(iface->router_event.uloop.fd, IPPROTO_IPV6, IPV6_MULTICAST_LOOP,
						&val, sizeof(val)) < 0) {
				...
			}

			/* Filter ICMPv6 package types */
			ICMP6_FILTER_SETBLOCKALL(&filt);
			ICMP6_FILTER_SETPASS(ND_ROUTER_ADVERT, &filt);
			ICMP6_FILTER_SETPASS(ND_ROUTER_SOLICIT, &filt);
			if (setsockopt(iface->router_event.uloop.fd, IPPROTO_ICMPV6, ICMP6_FILTER,
						&filt, sizeof(filt)) < 0) {
				...
			}
			// 将套接字加入uloop事件循环 并设置回调函数 handle_icmpv6
			iface->router_event.handle_dgram = handle_icmpv6;
			iface->ra_sent = 0;
			odhcpd_register(&iface->router_event);
		} else {
			...
		}

		memset(&mreq, 0, sizeof(mreq));
		mreq.ipv6mr_interface = iface->ifindex;
		inet_pton(AF_INET6, ALL_IPV6_ROUTERS, &mreq.ipv6mr_multiaddr);
		// 如果该接口RA配置为中继模式 且是主接口 则直接从主接口发送一次RS消息
		if (iface->ra == MODE_RELAY && iface->master) {
			inet_pton(AF_INET6, ALL_IPV6_NODES, &mreq.ipv6mr_multiaddr);
			forward_router_solicitation(iface);
		} else if (iface->ra == MODE_SERVER) {
			...
		}

		...
	}
out:
	...

	return ret;
}
```

```c
// ndp.c
int ndp_setup_interface(struct interface *iface, bool enable) {
	...

	snprintf(procbuf, sizeof(procbuf), "/proc/sys/net/ipv6/conf/%s/proxy_ndp", iface->ifname);
	procfd = open(procbuf, O_WRONLY);

	...

	if (enable) {
		struct sockaddr_ll ll;
		struct packet_mreq mreq;
		struct icmp6_filter filt;
		int val = 2;
		// 如果开启npd且是relay模式 则设置对应接口的 /proc/sys/net/ipv6/conf/<ifname>/proxy_ndp为1
		if (write(procfd, "1\n", 2) < 0) {}

		/* Open ICMPv6 socket */
		iface->ndp_ping_fd = socket(AF_INET6, SOCK_RAW | SOCK_CLOEXEC, IPPROTO_ICMPV6);
		...

		if (setsockopt(iface->ndp_ping_fd, SOL_SOCKET, SO_BINDTODEVICE,
			       iface->ifname, strlen(iface->ifname)) < 0) {
			...
		}

		if (setsockopt(iface->ndp_ping_fd, IPPROTO_RAW, IPV6_CHECKSUM,
				&val, sizeof(val)) < 0) {
			...
		}

		/* This is required by RFC 4861 */
		val = 255;
		if (setsockopt(iface->ndp_ping_fd, IPPROTO_IPV6, IPV6_MULTICAST_HOPS,
			       &val, sizeof(val)) < 0) {
			...
		}

		if (setsockopt(iface->ndp_ping_fd, IPPROTO_IPV6, IPV6_UNICAST_HOPS,
			       &val, sizeof(val)) < 0) {
			...
		}

		/* Filter all packages, we only want to send */
		ICMP6_FILTER_SETBLOCKALL(&filt);
		if (setsockopt(iface->ndp_ping_fd, IPPROTO_ICMPV6, ICMP6_FILTER,
			       &filt, sizeof(filt)) < 0) {
			...
		}


		iface->ndp_event.uloop.fd = socket(AF_PACKET, SOCK_DGRAM | SOCK_CLOEXEC, htons(ETH_P_IPV6));
		if (iface->ndp_event.uloop.fd < 0) {
			...
		}

		...

		if (setsockopt(iface->ndp_event.uloop.fd, SOL_SOCKET, SO_ATTACH_FILTER,
				&bpf_prog, sizeof(bpf_prog))) {
			...
		}

		...

		if (bind(iface->ndp_event.uloop.fd, (struct sockaddr*)&ll, sizeof(ll)) < 0) {
			...
		}

		...

		if (setsockopt(iface->ndp_event.uloop.fd, SOL_PACKET, PACKET_ADD_MEMBERSHIP,
				&mreq, sizeof(mreq)) < 0) {
			...
		}
		// 将套接字加入uloop事件循环 并设置回调函数 handle_solicit
		iface->ndp_event.handle_dgram = handle_solicit;
		odhcpd_register(&iface->ndp_event);

		...
	}

	...

 out:
	...

	return ret;
}
```

至此对于 `odhcpd` 的架构已经明了，整体框架是基于 `uloop` 的事件监听，当套接字有消息可读，就会调用对应的回调函数，在此基础上，`netlink` 向 `uloop` 添加事件监听，并在有消息时调用回调判断消息类型，调用注册到 `netlink` 事件链表中各个模块的函数，而 `router` `ndp` 等对配置的每个接口向 `uloop` 添加事件监听，并在有消息时调用回调函数。

> 即：
>
> `netlink` 会在消息到来调用回调 `cb_rtnl_valid`，根据消息类型调用各个模块的添加的回调函数 `xxx_netevent_cb`
>
> `router` 在 `netlink` 对应类型的消息到来会调用回调  `router_netevent_cb`， 对接口到来的消息会调用 `handle_icmpv6`
>
> `ndp` 在 `netlink` 对应类型的消息到来会调用回调  `ndp_netevent_cb`， 对接口到来的消息会调用 `handle_solicit`

## ROUTER

```c
// router.c
/**
 * router_netevent_cb - 处理网络事件的回调函数
 * @event: 发生的网络事件类型
 * @info: 包含事件相关详细信息的结构体指针
 *
 * 此函数根据传入的网络事件类型，执行相应的处理逻辑。主要处理的事件类型包括接口索引变更、
 * IPv6路由添加或删除、IPv6地址列表变更。对于不同的事件类型，执行不同的操作，如更新接口状态、
 * 触发路由更新或地址更新的定时器(trigger_router_advert)等。
 */
static void router_netevent_cb(unsigned long event, struct netevent_handler_info *info) {
	struct interface *iface;

	switch (event) {
	case NETEV_IFINDEX_CHANGE:
		/* 当网络接口索引发生变化时，注销并关闭该接口的文件描述符 */
		iface = info->iface;
		if (iface && iface->router_event.uloop.fd >= 0) {
			if (iface->router_event.uloop.registered)
				uloop_fd_delete(&iface->router_event.uloop);

			close(iface->router_event.uloop.fd);
			iface->router_event.uloop.fd = -1;
		}
		break;
	case NETEV_ROUTE6_ADD:
	case NETEV_ROUTE6_DEL:
		/* 当添加或删除IPv6路由时，针对服务器模式的接口更新定时器 */
		if (info->rt.dst_len)
			break;

		avl_for_each_element(&interfaces, iface, avl) {
			if (iface->ra == MODE_SERVER && !iface->master)
				uloop_timeout_set(&iface->timer_rs, 1000);
		}
		break;
	case NETEV_ADDR6LIST_CHANGE:
		/* 当IPv6地址列表发生变化时，对于服务器模式的接口更新定时器 */
		iface = info->iface;
		if (iface && iface->ra == MODE_SERVER && !iface->master)
			uloop_timeout_set(&iface->timer_rs, 1000);
		break;
	default:
		/* 对于未处理的事件类型，不做任何操作 */
		break;
	}
}
```

```c
// router.c
/* Event handler for incoming ICMPv6 packets */
/*
 * 处理IPv6的ICMP消息。
 * 
 * 参数:
 *  - addr: 源地址的指针。
 *  - data: 数据的指针，即ICMP报文的内容。
 *  - len: 数据长度。
 *  - iface: 当前接口的指针。
 *  - dest: 未使用的参数。
 */
static void handle_icmpv6(void *addr, void *data, size_t len,
		struct interface *iface, _unused void *dest) {
	struct icmp6_hdr *hdr = data; // ICMPv6报文头指针
	struct sockaddr_in6 *from = addr; // 源地址的详细信息

	// 验证ICMPv6报文的有效性，无效则直接返回
	if (!router_icmpv6_valid(addr, data, len))
		return;

	// 根据接口的角色（服务器模式或中继模式）来处理不同的ICMPv6类型
	if ((iface->ra == MODE_SERVER && !iface->master)) { /* 服务器模式 */
		// 对于路由器请求报文，发送路由器通告报文
		if (hdr->icmp6_type == ND_ROUTER_SOLICIT)
			send_router_advert(iface, &from->sin6_addr);
	} else if (iface->ra == MODE_RELAY) { /* 中继模式 */
		// 在中继模式下，根据报文类型和接口状态进行相应的处理
		if (hdr->icmp6_type == ND_ROUTER_SOLICIT && !iface->master) {
			// 转发路由器请求报文给其他中继接口
			struct interface *c;

			avl_for_each_element(&interfaces, c, avl) {
				if (!c->master || c->ra != MODE_RELAY)
					continue;

				forward_router_solicitation(c);
			}
		} else if (hdr->icmp6_type == ND_ROUTER_ADVERT && iface->master)
			// 转发路由器通告报文
			forward_router_advertisement(iface, data, len);
	}
}
```

## NDP

```c
// ndp.c
/*
 * NDP网络事件回调函数
 * 
 * 此函数处理有关IPv6邻居发现协议(NDP)的网络事件，例如地址添加/删除、邻居添加/删除。
 * 根据事件类型执行相应的配置或清理工作。
 *
 * @param event  发生的网络事件类型
 * @param info   包含事件详细信息的结构体指针，例如接口信息、地址或邻居信息
 */
static void ndp_netevent_cb(unsigned long event, struct netevent_handler_info *info) {
	struct interface *iface = info->iface;
	bool add = true; // 默认为添加操作

	// 如果接口不存在或NDP模式已禁用，则直接返回
	if (!iface || iface->ndp == MODE_DISABLED)
		return;

	switch (event) {
	case NETEV_ADDR6_DEL:
		// 地址删除操作，设置为删除模式，并清理邻居表
		add = false;
		netlink_dump_neigh_table(false);
		/* fall through */
	case NETEV_ADDR6_ADD:
		// 地址添加操作，为中继设置地址
		setup_addr_for_relaying(&info->addr.in6, iface, add);
		break;
	case NETEV_NEIGH6_DEL:
		// 邻居删除操作，设置为删除模式，并清理邻居表
		add = false;
		/* fall through */
	case NETEV_NEIGH6_ADD:
		// 邻居添加操作
		if (info->neigh.flags & NTF_PROXY) {
			// 处理代理邻居的情况
			if (add) {
				// 添加代理邻居，并设置相关路由
				netlink_setup_proxy_neigh(&info->neigh.dst.in6, iface->ifindex, false);
				setup_route(&info->neigh.dst.in6, iface, false);
				// 载入邻居表
				netlink_dump_neigh_table(false);
			}
			break;
		}

		// 非代理邻居的添加或删除操作
		if (add &&
		    !(info->neigh.state &
		      (NUD_REACHABLE|NUD_STALE|NUD_DELAY|NUD_PROBE|NUD_PERMANENT|NUD_NOARP)))
			break;

		// 设置中继地址和路由
		setup_addr_for_relaying(&info->neigh.dst.in6, iface, add);
		setup_route(&info->neigh.dst.in6, iface, add);

		// 删除操作时，清理邻居表
		if (!add)
			netlink_dump_neigh_table(false);
		break;
	default:
		// 未知事件，不处理
		break;
	}
}
```

```c
// ndp.c
/* Handle solicitations */
/*
 * 处理邻居请求（Neighbor Solicitation，NS）报文。
 * 
 * 参数：
 * addr - 发送请求的源地址。
 * data - 请求报文的数据指针。
 * len - 请求报文的长度。
 * iface - 接收到请求的网络接口。
 * dest - 未使用的参数。
 */
static void handle_solicit(void *addr, void *data, size_t len,
		struct interface *iface, _unused void *dest) {
	struct ip6_hdr *ip6 = data; // 指向IPv6头部。
	struct nd_neighbor_solicit *req = (struct nd_neighbor_solicit*)&ip6[1]; // 指向邻居请求结构体。
	struct sockaddr_ll *ll = addr; // 源地址的链路层地址。
	struct interface *c;
	char ipbuf[INET6_ADDRSTRLEN];
	uint8_t mac[6];

	/* 邻居请求用于重复地址检测（DAD） */
	bool ns_is_dad = IN6_IS_ADDR_UNSPECIFIED(&ip6->ip6_src);

	/* 如果接口不是中继模式，或者如果是外部接口且不是DAD请求，则不处理 */
	if (iface->ndp != MODE_RELAY || (iface->external && !ns_is_dad))
		return;

	// 校验报文总长度是否合法。
	if (len < sizeof(*ip6) + sizeof(*req))
		return; // 非法的总长度

	// 检查目标地址是否为链路本地、回环或组播地址，若是则不处理。
	if (IN6_IS_ADDR_LINKLOCAL(&req->nd_ns_target) ||
			IN6_IS_ADDR_LOOPBACK(&req->nd_ns_target) ||
			IN6_IS_ADDR_MULTICAST(&req->nd_ns_target))
		return; /* 无效的目标地址 */

	// 记录接收到的请求的目标地址。
	inet_ntop(AF_INET6, &req->nd_ns_target, ipbuf, sizeof(ipbuf));
	syslog(LOG_DEBUG, "Got a NS for %s on %s", ipbuf, iface->name);

	// 获取接口的MAC地址，并检查是否为回环报文。
	odhcpd_get_mac(iface, mac);
	if (!memcmp(ll->sll_addr, mac, sizeof(mac)))
		return; /* 回环报文，不处理 */

	// 遍历所有接口，对于中继模式的接口，如果不是当前接口，并且（是DAD请求或者不是外部接口），则发送ping6。
	avl_for_each_element(&interfaces, c, avl) {
		if (iface != c && c->ndp == MODE_RELAY &&
				(ns_is_dad || !c->external))
			ping6(&req->nd_ns_target, c);
	}

	/* 对于地址不是组播的NS请求，且源地址是链路本地地址的情况，手动发送邻居广告（NA）响应。
	 * 这是处理代理邻居的情况。 */
	if (!IN6_IS_ADDR_MULTICAST(&ip6->ip6_dst) &&
			IN6_IS_ADDR_LINKLOCAL(&ip6->ip6_src)) {
		bool is_proxy_neigh = netlink_get_interface_proxy_neigh(iface->ifindex,
				&req->nd_ns_target) == 1;

		if (is_proxy_neigh)
			send_na(&ip6->ip6_src, iface, &req->nd_ns_target, mac);
	}
}
```

[neigh_state]: data:image/png;base64,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
