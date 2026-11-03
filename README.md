# Nginx For Android (Bionic)

![GitHub Workflow Status](https://img.shields.io/github/actions/workflow/status/Ruixu-Account/Nginx-For-Android/build.yml?style=flat-square)
![Nginx Version](https://img.shields.io/badge/Nginx-1.30.6-green?style=flat-square)
![OpenSSL Version](https://img.shields.io/badge/OpenSSL-3.5.0-blue?style=flat-square)
![NDK Version](https://img.shields.io/badge/NDK-r27c-orange?style=flat-square)

本项目提供了一个基于 Android NDK 交叉编译原生 **Nginx** 的 GitHub Actions 工作流。编译目标锁定在 **Android API 24 (bionic)**，完全依靠静态链接，无需 Root，即可在 Android 沙箱或 Termux 环境中直接拉起。

> **Warning**: 本项目包含了大量针对 Nginx 祖传 `auto/` 构建脚本、Clang 18 严格语法、以及 Android Bionic C 库残缺特性的“外科手术级”补丁。阅读源码前请做好血压升高的准备。

## ✨ 特性 (Features)

*   **纯静态链接**：不依赖系统任何 `.so`（甚至没有 `NEEDED` 头），扔进任何 ARM64 / x86_64 的 Android 环境直接跑。
*   **HTTP/3 支持**：内嵌 OpenSSL 3.5.0，开启 QUIC 和 HTTP/3 模块，完美对接 Cloudflare CDN。
*   **全功能网关**：包含 Stream 四层代理、反向代理、Rewrite 等常用模块，可作为本地微服务网关。
*   **极致轻量**：由于启用了 Actions 缓存（NDK & PCRE2），单架构编译仅需 **1分半钟**。
*   **PQC支持**：由于使用了最新的 OpenSSL 3.5.0，所以支持 **X25519MLKEM768**。

## 📦 产物 (Artifacts)

每次触发 Workflow，你将获得以下打包文件（以 `1.30.6` 为例）：
*   `nginx-1.30.6-arm64-v8a.tar.gz`
*   `nginx-1.30.6-x86_64.tar.gz`

压缩包内包含编译好的 `sbin/nginx`、`conf/`、`logs/` 和 `html/` 目录。

## 🚀 在 Android 上运行

本项目编译时默认将 `prefix` 写死为了 `/data/data/com.nginx.app/nginx`。为了适配不同的运行环境（如 Termux、外部存储或不同包名的 App），启动时强烈建议使用 `-p` 参数动态指定当前的绝对路径。

```bash
# 1. 创建目录并解压
mkdir -p /data/data/com.nginx.app/nginx
tar -zxf nginx-1.30.6-arm64-v8a.tar.gz -C /data/data/com.nginx.app/nginx
cd /data/data/com.nginx.app/nginx

# 2. 赋予执行权限
chmod +x sbin/nginx

# 3. 验证版本与编译参数
./sbin/nginx -V

# 4. 启动 Nginx（关键：使用 -p 覆盖编译时的硬编码路径）
# -p 指定 prefix 目录，-c 指定配置文件路径
./sbin/nginx -p $(pwd) -c conf/nginx.conf

# 5. 重载或停止 Nginx（同样需要带上 -p 参数）
./sbin/nginx -p $(pwd) -s reload
./sbin/nginx -p $(pwd) -s stop
```

(注：如果你在 App 沙箱内通过 ProcessBuilder 运行，直接传入 -p 和应用的私有目录绝对路径即可。)

---

🤬 底层排雷记录 (The "Fuck Bionic" Section)

这绝不是一次轻松愉快的编译。如果你尝试用原版 Nginx 源码直接上 NDK 交叉编译，你会遇到以下所有报错，本项目已经在 YAML 中将其一一物理超度：

1. C compiler is not found
*  原因：交叉编译无法在 x86 机器上运行 ARM 测试程序，Nginx 探测脚本判定编译器损坏。
*  解法：Python 脚本强行替换 auto/feature 中的运行测试逻辑（改为 case "skip"）。
2. can not define uint32_t / cannot detect system byte ordering
*   原因：Clang 18 严格模式下 sys/types.h 不默认包含 <stdint.h>；且无法运行 ARM 二进制检测大小端。
*   解法：重写 auto/types/typedef 和 auto/endianness，直接硬编码强制输出 little endian 和 4 bytes。
3. unknown type name 'cpu_set_t'
*   原因：Bionic 将其藏在了 _GNU_SOURCE 宏后面。
*   解法：在 CFLAGS 中暴力注入 -D_GNU_SOURCE，并向 ngx_setaffinity.h 强行注入 <sched.h>。
4. call to undeclared function 'glob'
*   原因：Nginx 的 ngx_files.c 忘了引入 <glob.h>，在老版本 GCC 下能过，Clang 18 直接报错。
*   解法：用 Python 强行在最前面注入 #include <glob.h>。
5. token is not a valid binary operator in a preprocessor subexpression
*   原因：Android 内核 UAPI 头文件将 EPOLLRDHUP 定义成了带类型转换的 (__poll_t) 0x2000，C 预处理器解析直接崩溃。
*   解法：用 sed 将 ngx_epoll_module.c 里的 #if 全部改成 #if 1，并强行注入 -DEPOLLRDHUP=0x2000。
6. unable to find library -lpthread
*   原因：Bionic 早就把 libpthread 和 libdl 合并进了 libc.so。Nginx 的 auto/unix 却非要塞 -lpthread。
*   解法：在 make 前，用 sed 把 objs/Makefile 里的 -lpthread 和 -ldl 全部物理扬了。
7. Segmentation fault 与 TLS segment underaligned 的双重夹击
*   原因：Android 16 的 Bionic Linker 强制要求 TLS 段 64 字节对齐，但 NDK r27c 的 LLD 链接器根本不支持 tls-align 参数。
*   解法：在 CI 里用 Python 脚本读取 ELF 的 e_phoff，暴力定位 PT_TLS 段，强制把 p_align 改成 64。Fuck Bionic Linker.
🛠️ 构建环境

* Host: ubuntu-latest
* NDK: r27c (27.3.13750724)
* Target API: 24 (Android 7.0)
* 依赖库: OpenSSL 3.5.0, PCRE2 10.44

📜 免责声明 (Disclaimer)

本仓库仅供极客折腾与学习交流使用。请勿用于生产环境的公网大流量服务（除非你确信你的 Android 平板散热良好，且准备好了随时被 SELinux 制裁）。
