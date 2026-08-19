# nginx-gmssl-build

国密版 nginx 双架构编译产物流水线（**B 方案：官方 nginx 分支 + GmSSL 密码库**）。

> 与 A 方案（tengine-gm-build：Tengine + Tongsuo）的区别：
> - A 方案是 OpenSSL 生态（1.1.1 兼容层），Tengine 自带 NTLS 模块
> - B 方案是 **GmSSL 官方 nginx 分支**（纯 GmSSL TLS 后端，替换 OpenSSL），国标 TLCP 原生支持，政务/金融认可度高
> - **B 方案无 RSA 通道**：nginx-gmssl 后端只服务国密协议，普通浏览器（RSA/TLS 国际套件）入口需另配常规 nginx 实例

## 组合

| 组件 | 版本 | 说明 |
|---|---|---|
| [nginx-gmssl](https://github.com/GmSSL/nginx-gmssl) | commit `7c70da7`（基于 nginx 1.30.2） | GmSSL 官方 nginx 分支，OpenSSL 后端替换为 GmSSL TLS 接口 |
| [GmSSL](https://github.com/guanzhi/GmSSL) | v3.2.0 | 北大开源国密密码库（SM2/SM3/SM4/TLCP） |
| 协议 | TLCP / TLS1.2 / TLS1.3 | 每个 server 只能启用一个协议（GmSSL 后端限制） |

## 产物（GitHub Actions Artifacts）

| Artifact | 目标服务器 | 编译容器 | glibc |
|---|---|---|---|
| `nginx-gmssl-x86_64` | CentOS 7.x（x86） | centos:7.9.2009 | 2.17 |
| `nginx-gmssl-aarch64` | 麒麟 V10 / UOS（ARM） | almalinux:8 | 2.28 |

产物为 `/usr/local/nginx` 完整目录的 tar.gz 包。

## 触发构建

```bash
gh workflow run build.yml
gh run watch
```

## 服务器部署

```bash
# 1. 拷贝对应架构的产物包到服务器
# 2. 解压到 /usr/local
tar xzf nginx-gmssl-x86_64-centos7-glibc217.tar.gz -C /usr/local

# 3. 验证可执行
/usr/local/nginx/sbin/nginx -V

# 4. 放置证书并配置（TLCP 双证书示例）
# 关键配置：
#   ssl_protocols TLCP;                       # 单协议模式
#   ssl_certificate     /path/double_certs.pem;   # 签名证书+加密证书合并文件
#   ssl_certificate_key /path/double_keys.pem;    # 两个私钥合并文件
#   （证书按 KeyUsage 扩展自动区分角色：digitalSignature=签名，keyEncipherment=加密）

# 5. 启动
/usr/local/nginx/sbin/nginx
```

## 证书要求

- TLCP 需要 SM2 **签名证书 + 加密证书**两套，KeyUsage 扩展是角色判定依据：
  - 签名证书：`keyUsage=critical,digitalSignature,nonRepudiation`
  - 加密证书：`keyUsage=critical,keyEncipherment,keyAgreement`
- 自签测试证书可用本仓库 `test-certs/` 下的（含 KeyUsage 扩展，`double_certs.pem`/`double_keys.pem` 为合并格式）
- 正式证书向 CA（CFCA/阿里云等）申请国密双证书；**普通浏览器通道需另外部署常规 nginx（RSA 证书）**

## 验证国密握手

```bash
# GmSSL 自带 TLCP 客户端（国标互通）
gmssl tlcp_client -host localhost -port 8445 -cacert cacert.pem
# 输出 Connection established! 即成功
```

## 注意事项

- **已知上游 bug（重要）**：nginx-gmssl 当前版本 TLCP 握手会导致 worker 崩溃（客户端报 Connection reset by peer），官方测试证书/正式证书均复现，见 [GmSSL/nginx-gmssl issue #2](https://github.com/GmSSL/nginx-gmssl/issues/2)。流水线握手自测为软断言（WARN 不阻断构建）。**生产使用前需等待上游修复或换用 A 方案（Tengine + Tongsuo）**。官方 CI 目前也只做 `nginx -t` 不做握手，与本现象一致。
- 产物动态依赖：`libgmssl`（rpath 已嵌入 /usr/local/lib）、`libpcre2-8`、`libz`——部署服务器需有对应库（编译容器内已装 /usr/local，服务器上如缺失需从产物包或系统源补齐）
- nginx-gmssl 启动时校验编译期/运行期 GmSSL 版本一致，不可混用不同版本的 libgmssl
- 本仓库测试证书为 localhost 自签，仅用于演练，勿用于生产
