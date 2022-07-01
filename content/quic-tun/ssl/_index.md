---
"title": SSL
---

SSL（安全套接字层）在 TCP 协议中是可选的，但是 QUIC 协议中 SSL 是强制存在的。在该章节我们专门介绍 SSL 在 `quic-tun` 中的应用。
该文档默认您已经掌握 SSL 涉及到的概念：公钥，私钥，证书，CA。SSL 的功能主要有两个：数据加密、认证。该文档主要针对第二点，介绍
一下 SSL 认证功能在 quic-tun 中的应用。

## 相关参数

* `--ca-file` 指定 CA 证书，在 `quictun-client` 需要是为 `quictun-server` 的公钥签名的 CA 的证书；在 `quictun-server` 需
  要是为 `quictun-client` 的公钥签名的证书
* `--cert-file` 指定证书文件（经过 CA 签名的公钥）
* `--key-file` 指定私钥文件
* `--verify-client` 这个参数是 `quictun-server` 特有的，用于指定 `quictun-server` 是否验证 `quictun-client`，如果该值为 True，那
  么 `quictun-client` 启动时必须有 `--cert-file` 和 `--key-file` 参数。
* `--insecure-skip-verify` 这个参数是 `quictun-client` 特有的，用于指定是否验证 `quictun-server` 的证书，这个值为 False 时表明需要
  验证 `quictun-server` 的证书，这时候 `quictun-client` 的 `--ca-file` 参数是必须要传的。

QUIC 协议中加密是强制要求的，但是认证是可选的。而加密工作依赖于服务端的的私钥和证书，在 quic-tun 中指定私钥和证书的参数分别
是 `--key-file` 和 `--cert-file`。如果在启动 `quictun-server` 是不指定这两个参数，`quictun-server` 会自动创建一个临时的
私钥和证书并进行自签名。
