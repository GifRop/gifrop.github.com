---
title: 打造自己的 dns 记录服务 
date: 2021-12-12 22:44:48
tags: [dns, log4j]
---

2021年12月9日，Apache Log4j2 爆出核弹级漏洞，该漏洞可以远程代码执行，一旦被攻击者利用，将造成严重危害。各大公司均连夜进行应急处理和推送业务修复。<!-- more -->

在漏洞处理和修复过程中，有个 dnslog 被广泛提及，主要是用于严重业务是否存在漏洞，代码或者命令是否真实执行，因为其使用 dns 查询请求的 payload 在受影响的用户端执行，dnslog 服务端记录 dns 请求，也被叫 dns 回显平台。  

这里，我们使用 go 开发一个 dns 记录服务，帮助业务快速自查风险。  

首先，定义一个 DNS 结构，用于存储真实服务解析。  

```go
type DNS struct {
	Port   int     // 需要绑定的端口，一般为53
	Domain string   // dns 解析服务提供泛解析的域名
	Ip     net.IP   // 域名需要被解析到的ip
}
```

绑定端口进行 udp 数据解析, 我们通过 dns 相关信息知道，dns 请求数据最长为 512 字节：  

```go
func (dns *DNS) Start() { 	
  conn, err := net.ListenUDP("udp", &net.UDPAddr{Port: dns.Port})
	if err != nil {
		panic(err)
	}
	defer conn.Close()
  for {
		buf := make([]byte, 512)
		_, addr, _ := conn.ReadFromUDP(buf)
}
```

接着解包 dns 数据，这里需要使用到包 `golang.org/x/net/dns/dnsmessage` 。  

```go
		if err := msg.Unpack(buf); err != nil {
      // 解包失败丢弃，重新获取
			continue
		}
    // 过滤掉无效请求
		if len(msg.Questions) == 0 {
			continue
		}
    // 在协程里进行处理
		go dns.ServerDNS(addr, conn, msg)
```

在处理的时候，我们首先梳理下，需要做哪些事情：  

1. 拿到 client ip，用于定位哪个服务发起了这个域名的 dns 查询。
2. 正常返回 dns 结果，主要为 A 记录和 AAAA 记录。
3. 记录查询请求的域名信息、类型信息、clientip 进行存储。

那我们开始吧，先拿到client ip:

```go
func (dns *DNS) ServerDNS(addr *net.UDPAddr, conn *net.UDPConn, msg dnsmessage.Message) {
	req := msg.Questions[0]
	ip := addr.IP
	port := addr.Port
```

接着让正常请求能够返回，获取请求类型，判断是否是 A 或者 AAAA ，根据类型进行处理：

```go
	var reqTypeStr = req.Type.String()
	var reqNameStr = req.Name.String()
	var reqType = req.Type
	var reqName, _ = dnsmessage.NewName(reqNameStr)

	fmt.Printf("[%s] reqName: [%s] clientip: [%s]:[%d]\n",
		reqTypeStr, reqNameStr, ip, port)

	var resource dnsmessage.Resource
	switch reqType {
	case dnsmessage.TypeA:
    // 解析所有后缀为 www.baidu.com 的域名
		if strings.HasSuffix(reqNameStr, dns.Domain+".") {
			ipv4 := dns.Ip.To4()
			if ipv4 != nil {
				resource = dnsmessage.Resource{
					Header: dnsmessage.ResourceHeader{
						Name:  reqName,
						Class: dnsmessage.ClassINET,
						TTL:   600,
					},
					Body: &dnsmessage.AResource{
						A: [4]byte{ipv4[0], ipv4[1], ipv4[2], ipv4[3]},
					},
				}
			}
	default:
		return
	}
	msg.Response = true
	msg.Answers = append(msg.Answers, resource)
	go dns.SendAnswers(addr, conn, msg)
}

func (dns *DNS) SendAnswers(addr *net.UDPAddr, conn *net.UDPConn, msg dnsmessage.Message) {
	packed, err := msg.Pack()
	if err != nil {
		fmt.Println(err)
		return
	}
	if _, err := conn.WriteToUDP(packed, addr); err != nil {
		fmt.Println(err)
		return
	}
}
```

我们来试试：

```go
func main(){
  ipAddr := net.ParseIP("1.1.1.1")
	dns := &DNS{Domain: "www.baidu.com", Port: 8080, Ip: ipAddr}
	dns.Start()
}
```

使用 dig 进行验证：

```shell
└─(17:01:07)──> dig @127.0.0.1 -p 8080 111.www.baidu.com                     9 ↵ ──(星期日 21年12月12日)─┘

; <<>> DiG 9.10.6 <<>> @127.0.0.1 -p 8080 111.www.baidu.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 48203
;; flags: qr rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;111.www.baidu.com.		IN	A

;; ANSWER SECTION:
111.www.baidu.com.	600	IN	A	1.1.1.1

;; Query time: 1 msec
;; SERVER: 127.0.0.1#8080(127.0.0.1)
;; WHEN: Sun Dec 12 17:01:39 CST 2021
;; MSG SIZE  rcvd: 62
```

我们看到，111.www.baidu.com 的 A 记录已经被正常的解析到了 1.1.1.1 这个 ip 地址。  

下面来搞定 AAAA 记录：

```go
	case dnsmessage.TypeAAAA:
    // 解析所有后缀为 www.baidu.com 的域名
		if strings.HasSuffix(reqNameStr, dns.Domain+".") {
			ipv6 := dns.Ip.To16()
			if ipv6 != nil {
				var ipV6 [16]byte
				copy(ipV6[:], ipv6)
				resource = dnsmessage.Resource{
					Header: dnsmessage.ResourceHeader{
						Name:  reqName,
						Class: dnsmessage.ClassINET,
						TTL:   600,
					},
					Body: &dnsmessage.AAAAResource{
						AAAA: ipV6,
					},
				}
			} 
```

这样服务就已经的对域名的 A 和 AAAA 记录了：

```shell
[TypeA] reqName: [111.www.baidu.com.] clientip: [127.0.0.1]:[60065]
[TypeAAAA] reqName: [111.www.baidu.com.] clientip: [127.0.0.1]:[50532]
```

后续只需要进行 dns 记录的存储和查询。在公司内部，也可以将 dnslog 的请求，全部转发到该服务，即可感知到所有被外部检查的存在漏洞的服务了。同时提供查询和删除的管理界面，也能给业务进行自查，帮助业务更快的修复安全漏洞。  

最后，祈福永无BUG，无安全漏洞！！！

