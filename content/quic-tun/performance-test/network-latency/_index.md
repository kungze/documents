---
"title": 网络延迟
---

在这个测试场景中，我们将额外测试公网环境下，quic-tun 对网络延迟的影响。
[测试工具](https://github.com/kungze/quic-tun/tree/master/tests/latency)是
我们针对这个场景专门编写的。

## 准备工作

* 在 server 虚机

  启动 TCP server

  ```console
  $ ./server
  The server listen on 0.0.0.0:15676
  ```

  打开一个新的终端，启动 quictun-server

  ```console
  $ ./quictun-server
  I0624 09:15:29.223140    1515 server.go:30] "Server endpoint start up successful" listen address="[::]:7500"
  ```

* 在 client 虚机

  启动 quictun-client

  ```console
  $ ./quictun-client --server-endpoint 192.168.26.129:7500 --token-source tcp:127.0.0.1:5201 --insecure-skip-verify true
  I0624 09:17:30.926905    1679 client.go:35] "Client endpoint start up successful" listen address="127.0.0.1:6500"
  ```

  `192.168.26.129` 是 server 虚机的 IP 地址，在公网场景，这个就是公网 IP。

  打开一个新的终端，运行 `client` python 脚本测试网络延迟

  * 测试直接 TCP 传输的网络延迟

  ```console
  ./client --server-host 192.168.26.131
  ```

  * 测试通过 quic-tun 传输的网络延迟

  ```console
  ./client --server-host 127.0.0.1 --server-port 6500
  ```

## 测试结果

### 丢包率设置为 0.0% (本地网络)

* TCP

```console
$ ./client --server-host 192.168.26.131
First packet latency: 0.7572174072265625 ms
Total latency: 499.89843368530273 ms
```

* quic-tun

```console
$ ./client --server-host 127.0.0.1 --server-port 6500
First packet latency: 7.899284362792969 ms
Total latency: 591.1564826965332 ms
```

### 丢包率设置为 1.0% (本地网络)

* TCP

```console
$ ./client --server-host 192.168.26.131
First packet latency: 0.6091594696044922 ms
Total latency: 4290.04430770874 ms
```

* quic-tun

```console
$ ./client --server-host 127.0.0.1 --server-port 6500
First packet latency: 8.286714553833008 ms
Total latency: 1201.0939121246338 ms
```

### 丢包率设置为 0.0% (公网网络)

* TCP

```console
$ ./client --server-host 47.111.149.1 --server-port 5201
First packet latency: 24.95884895324707 ms
Total latency: 25493.980407714844 ms
```

* quic-tun

```console
$ ./client --server-host 127.0.0.1 --server-port 6500
First packet latency: 100.67296028137207 ms
Total latency: 24987.539291381836 ms
```

### 丢包率设置为 1.0% (公网网络)

* TCP

```console
$ ./client --server-host 47.111.149.1 --server-port 5201
First packet latency: 23.789167404174805 ms
Total latency: 28489.194869995117 ms
```

* quic-tun

```console
$ ./client --server-host 127.0.0.1 --server-port 6500
First packet latency: 103.04689407348633 ms
Total latency: 27247.18403816223 ms
```

### 丢包率设置为 2.0% (公网网络)

* TCP

```console
$ ./client --server-host 47.111.149.1 --server-port 5201
First packet latency: 36.420583724975586 ms
Total latency: 33720.38745880127 ms
```

* quic-tun

```console
$ ./client --server-host 127.0.0.1 --server-port 6500
First packet latency: 100.87323188781738 ms
Total latency: 27528.08117866516 ms
```

### 结果分析

从上面的测试结果可以看出在本地存在丢包的网络环境中，quic-tun 对改善网络延迟有突出的表现，没有丢包是改善不明显。
在公网环境中，quic-tun 对网络延迟的改善并不明显。另外需要特别注意的是在使用 quic-tun 的情况下第一个包的网络延迟
特别的长，这是因为 quic-tun 隧道的建立是由第一个包触发的，第一个包的传输需要等待隧道建立成功。从这里也可以看出
quic-tun 不适合短连接 TCP 应用，如频繁且短暂的 http 请求应用，但是对于长链接的 http 应用我相信 quic-tun 也能对
其有很大优化。
