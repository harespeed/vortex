# How-to：从零构建与运行（C++ / 零包管理器 / PQC-ready）

> 状态：设计阶段的操作手册。**应用本身的命令标注为“目标”，尚未实现。**  
> 目标平台：macOS（Apple Silicon / Intel）优先，Linux 次之。  
> 原则：不用包管理器；只允许 OpenSSL 与 SQLite amalgamation；所有下载都校验 SHA-256。

---

## 0. 目录与数据布局

```text
home-agent/
  CMakeLists.txt
  src/                 # 目标：agentd / agentctl
  third_party/
    openssl/           # 源码构建产物（静态库 + 头文件）
    sqlite/            # sqlite3.c / sqlite3.h
  data/
    app.db             # SQLite
    runs/<run_id>/     # 每个 run 的工作区
    artifacts/         # 大输出、文件
  keys/                # 仅公钥与 wrapped key；私钥在 Keychain/TPM
  backups/
  docs/
```

权限：`data/` 与 `keys/` 为 `0700`；`app.db` 为 `0600`；备份目录单独加密。

---

## 1. 工具链

| 工具 | 版本 | 检查 |
|---|---|---|
| C++ 编译器 | Apple Clang 16+ / GCC 12+ | `c++ --version` |
| CMake | 3.25+ | `cmake --version` |
| Ninja | 任意 | `ninja --version` |
| Perl | 任意（OpenSSL 构建需要） | `perl -v` |
| jq / curl | 任意（CVE 检查用） | `jq --version` |

这些是**构建工具**，不是运行时依赖。

---

## 2. 构建 OpenSSL 3.5.10（静态、裁剪）

> 为什么是 3.5.10：3.5.0–3.5.9 受 CVE-2026-107493 / 35190 / 88823 / 124709 影响；3.5.8 之前的 CMS/QUIC 问题也需要修。以官方发布页为准。

```bash
mkdir -p third_party/src && cd third_party/src

# 从官方发布页下载源码，并复制该页公布的 SHA-256
curl -LO https://www.openssl.org/source/openssl-3.5.10.tar.gz
shasum -a 256 openssl-3.5.10.tar.gz
# 与 https://openssl-library.org/source/ 上的值逐字比对；不要凭记忆写哈希

tar xzf openssl-3.5.10.tar.gz
cd openssl-3.5.10

./Configure \
  --prefix="$PWD/../../openssl" \
  --openssldir="$PWD/../../openssl/ssl" \
  no-shared no-module \
  no-quic no-cms no-cmp no-dtls \
  no-ssl3 no-comp no-weak-ssl-ciphers \
  -O2 -fPIC

make -j"$(sysctl -n hw.ncpu 2>/dev/null || nproc)"
make install_sw
```

验证：

```bash
third_party/openssl/bin/openssl version -a
third_party/openssl/bin/openssl list -kem-algorithms | grep -i ML-KEM
third_party/openssl/bin/openssl list -signature-algorithms | grep -i ML-DSA
# 若该子命令不存在，用 `openssl list -public-key-methods` 或 `-providers -verbose` 检查
```

应看到 `ML-KEM-768`、`ML-DSA-65` 等。

**注意**：

- `no-quic` 关闭 CVE-2026-18798 / 124709 的代码路径。
- `no-cms` 关闭 CVE-2026-63072 的代码路径。
- `no-dtls` 关闭 DTLS 放大问题。
- 如果 `3.5.10` 尚未镜像到你的源，使用官方最新 3.5.x 并核对公告；不要用 3.5.8 以下。

---

## 3. 构建 SQLite 3.54.1（amalgamation）

> 3.54.0 的 FTS5/session 有 CVE-2026-119373/119374/119379；3.54.1（2026-10-16）修复。我们不启用 FTS5/session，但仍升到 3.54.1+。

```bash
cd third_party/src
# 从 https://sqlite.org/download.html 获取 3.54.1 的 amalgamation
# 文件名形如 sqlite-amalgamation-3540100.zip
curl -LO https://sqlite.org/2026/sqlite-amalgamation-3540100.zip
shasum -a 256 sqlite-amalgamation-3540100.zip   # 与官网核对
unzip -q sqlite-amalgamation-3540100.zip
cp sqlite-amalgamation-3540100/sqlite3.c ../sqlite/
cp sqlite-amalgamation-3540100/sqlite3.h ../sqlite/
```

编译选项（写进 CMake，不单独编译）：

```text
-DSQLITE_THREADSAFE=1
-DSQLITE_OMIT_LOAD_EXTENSION
-DSQLITE_DQS=0
-DSQLITE_OMIT_DEPRECATED
-DSQLITE_OMIT_SHARED_CACHE
-DSQLITE_DEFAULT_FOREIGN_KEYS=1
-DSQLITE_MAX_VARIABLE_NUMBER=250000
```

不要定义 `SQLITE_ENABLE_FTS5`、`SQLITE_ENABLE_SESSION`。

验证：

```bash
echo 'PRAGMA compile_options;' | sqlite3 :memory: | grep -E 'FTS5|SESSION' || echo "FTS5/session disabled"
```

---

## 4. 构建 agentd（CMake 目标）

`CMakeLists.txt` 的关键点：

```cmake
cmake_minimum_required(VERSION 3.25)
project(home_agent LANGUAGES C CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(OPENSSL_ROOT_DIR "${CMAKE_SOURCE_DIR}/third_party/openssl")
find_package(OpenSSL 3.5 REQUIRED)

add_library(sqlite3 STATIC third_party/sqlite/sqlite3.c)
target_compile_definitions(sqlite3 PRIVATE
  SQLITE_THREADSAFE=1 SQLITE_OMIT_LOAD_EXTENSION SQLITE_DQS=0
  SQLITE_OMIT_DEPRECATED SQLITE_OMIT_SHARED_CACHE
  SQLITE_DEFAULT_FOREIGN_KEYS=1)

add_executable(agentd src/main.cpp ...)
target_link_libraries(agentd PRIVATE OpenSSL::SSL OpenSSL::Crypto sqlite3 pthread)

if (APPLE)
  target_compile_options(agentd PRIVATE -Wall -Wextra -Wpedantic
    -fstack-protector-strong -D_FORTIFY_SOURCE=3 -fPIE)
  target_link_options(agentd PRIVATE -pie -Wl,-dead_strip)
else()
  target_compile_options(agentd PRIVATE -Wall -Wextra -Wpedantic
    -fstack-protector-strong -D_FORTIFY_SOURCE=3 -fPIE)
  target_link_options(agentd PRIVATE -pie -Wl,-z,relro,-z,now)
endif()
```

构建：

```bash
cmake -S . -B build -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DOPENSSL_ROOT_DIR="$PWD/third_party/openssl"
cmake --build build -j
```

检查运行时依赖：

```bash
otool -L build/agentd      # macOS：应只有系统库（OpenSSL/SQLite 静态链接）
ldd build/agentd           # Linux：同上
```

---

## 5. 首次启动与账号

目标命令（待实现）：

```bash
./build/agentd --data ./data --listen 127.0.0.1:8443 --tls-cert keys/server.crt --tls-key keys/server.key

./build/agentctl user add --name admin --role admin
./build/agentctl user add --name kid1  --role kid
```

规则：

- 首个用户必须是 `admin`。
- 密码交互输入，不落 shell history；服务端 scrypt 哈希。
- 登录后发会话 Cookie（HttpOnly/SameSite=Lax/Secure）。
- 所有 POST 校验 CSRF。

---

## 6. TLS：私有 CA + hybrid 组

### 6.1 生成 CA 与服务器证书（ECDSA P-256）

```bash
mkdir -p keys && cd keys

openssl ecparam -name prime256v1 -genkey -noout -out ca.key
openssl req -x509 -new -key ca.key -sha256 -days 3650 \
  -subj "/CN=Home Agent CA" -out ca.crt

openssl ecparam -name prime256v1 -genkey -noout -out server.key
openssl req -new -key server.key -subj "/CN=agent.home" \
  -addext "subjectAltName=DNS:agent.home,IP:127.0.0.1,IP:192.168.1.10" \
  -out server.csr

openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -days 825 -sha256 \
  -extfile <(printf "subjectAltName=DNS:agent.home,IP:127.0.0.1,IP:192.168.1.10\n") \
  -out server.crt
```

权限：`chmod 600 ca.key server.key`；CA 私钥离线保存。

### 6.2 服务端配置

目标（实现时）：

```cpp
SSL_CTX_set_min_proto_version(ctx, TLS1_3_VERSION);
SSL_CTX_set1_groups_list(ctx, "X25519MLKEM768:X25519");
```

### 6.3 验证协商

```bash
openssl s_client -connect 127.0.0.1:8443 -tls1_3 \
  -groups X25519MLKEM768 -CAfile keys/ca.crt -brief
```

输出应包含：

```text
Negotiated TLS1.3 group: X25519MLKEM768
```

如果显示 `X25519`，说明回退发生：检查服务端 groups 列表、客户端支持、以及是否被中间层终止。

**注意**：证书签名仍是 ECDSA；hybrid 只保护密钥交换。若要全 PQC 认证，需要私有 ML-DSA CA，复杂度更高（见 architecture ADR-6）。

---

## 7. 沙箱

### 7.1 macOS：sandbox-exec

```bash
cat > sandbox/run.sb <<'EOF'
(version 1)
(deny default)
(allow process-exec (literal "/usr/bin/python3"))
(allow file-read* (subpath "/Users/me/home-agent/data/runs/RUN_ID"))
(allow file-write* (subpath "/Users/me/home-agent/data/runs/RUN_ID"))
EOF

sandbox-exec -f sandbox/run.sb -- /usr/bin/python3 -c 'print("ok")'
```

- 默认拒绝网络；需要网络的工具走独立 `http.fetch`。
- `sandbox-exec` 已 deprecated，但仍可用；若未来失效，改为“独立用户 + 网络扩展/防火墙”。
- 路径必须 `realpath` 后校验前缀，防符号链接逃逸。

### 7.2 Linux：unshare + seccomp

```bash
unshare -n -r --fork \
  /usr/bin/python3 -c 'print("no network")'
```

- `-n` 断网；`-r` 映射 root（用户命名空间）。
- seccomp 由实现里的 BPF 程序加载：默认拒绝，只允许 `read/write/exit/execve/brk/mmap` 等最小集合。
- 独立用户 + 只读根 + 每 run 临时目录。

### 7.3 逃逸测试（必须做）

| 测试 | 期望 |
|---|---|
| `cat ../../etc/passwd` | 拒绝 |
| 绝对路径 `/etc/passwd` | 拒绝 |
| 符号链接指向工作区外 | 拒绝（realpath 校验） |
| `curl https://example.com` | 拒绝（无网络） |
| 无限循环 | 30 s 后 kill |
| 分配 2 GB | 拒绝（RLIMIT_AS） |
| fork 炸弹 | 拒绝（RLIMIT_NPROC） |

---

## 8. PQC 信封加密

### 8.1 算法与格式

- KEM：X25519 + ML-KEM-768（hybrid）。
- KDF：HKDF-SHA256。
- AEAD：AES-256-GCM。
- 头部：`AGF1 | ver | alg | kem | key_id | nonce | wrapped_len | wrapped | ct_len | ct | tag`。
- AAD = 头部（防篡改）。

### 8.2 包裹流程

1. 随机 DEK（32 B）。
2. 临时 X25519 密钥对 + 接收方 X25519 公钥 → `ss1`。
3. ML-KEM-768 封装到接收方公钥 → `ct2`, `ss2`。
4. `KEK = HKDF-SHA256(ss1 || ss2, salt, info="AGF1/wrap")`。
5. `AES-256-GCM(KEK, nonce, DEK)` → `wrapped`, `tag`。
6. 存储：临时 X25519 公钥、`ct2`、nonce、tag、key_id。

解封按相反顺序；AEAD 失败即拒绝。

### 8.3 目标命令（待实现）

```bash
./build/agentctl keygen --alg hybrid-x25519-mlkem768 --out keys/household-1
./build/agentctl seal   --key keys/household-1.pub --in backups/runs.tgz --out backups/runs.tgz.agf
./build/agentctl open   --key keys/household-1.key --in backups/runs.tgz.agf --out backups/runs.tgz
./build/agentctl rotate --old keys/household-1.key --new keys/household-2.pub --rewrap data/artifacts
```

### 8.4 测试用例

| 测试 | 期望 |
|---|---|
| 正常 seal/open | 字节一致 |
| 改 1 bit 密文 | 拒绝 |
| 改头部 `key_id` | 拒绝 |
| 错误私钥 | 拒绝 |
| 旧 key 解新数据 | 拒绝 |
| 轮换后旧数据 | 仍可解（保留窗口内） |
| 1 GB 文件 | 时间/内存达标 |

---

## 9. 备份与恢复

### 9.1 备份

```bash
mkdir -p backups
sqlite3 data/app.db ".backup 'backups/app.db'"
tar -czf backups/runs.tgz data/runs data/artifacts
# 目标：信封加密
./build/agentctl seal --key keys/backup.pub --in backups/runs.tgz --out backups/runs.tgz.agf
```

保留 3 代；备份密钥离线保存（打印纸质恢复码或硬件密钥）。

### 9.2 恢复演练（每月）

```bash
# 停服务
sqlite3 backups/app.db "PRAGMA integrity_check;"
tar -tzf backups/runs.tgz >/dev/null
# 目标：解密并校验
./build/agentctl open --key keys/backup.key --in backups/runs.tgz.agf --out /tmp/runs.tgz
# 恢复到临时目录，启动 agentd，跑 3 条黄金任务
```

**没有演练过的备份 = 没有备份。**

---

## 10. CVE 监控与升级

### 10.1 每月检查

```bash
# OpenSSL 3.5 公告
curl -s https://openssl-library.org/news/vulnerabilities-3.5/ | \
  grep -oE 'CVE-[0-9]{4}-[0-9]+' | sort -u

# SQLite 发布日志
curl -s https://sqlite.org/releaselog/current.html | grep -iE 'CVE|overflow|use-after'

# NVD 关键词（按需调整日期）
for kw in OpenSSL ML-KEM ML-DSA SQLite; do
  curl -s "https://services.nvd.nist.gov/rest/json/cves/2.0?keywordSearch=$kw&pubStartDate=2026-01-01T00:00:00.000" \
    | jq -r '.vulnerabilities[]?.cve.id' | head
  sleep 7
done
```

### 10.2 升级流程

1. 记录当前版本与 `otool -L` / `ldd`。
2. 下载新版本源码 → 校验 SHA-256。
3. 重新构建 OpenSSL/SQLite/agentd。
4. 跑 20 条黄金集 + 沙箱逃逸测试 + TLS 协商测试。
5. 检查 `Negotiated TLS1.3 group` 仍为 `X25519MLKEM768`。
6. 备份 → 部署 → 观察 24 h。

**不要**在未跑 eval 的情况下升级模型或加密库。

---

## 11. 验证清单

- [ ] `openssl version` ≥ 3.5.10
- [ ] `openssl list -kem-algorithms` 含 ML-KEM-768
- [ ] `openssl list -signature-algorithms` 含 ML-DSA-65
- [ ] `s_client` 协商 `X25519MLKEM768`
- [ ] SQLite `PRAGMA compile_options` 无 FTS5/session
- [ ] `otool -L`/`ldd` 无第三方动态库
- [ ] 沙箱逃逸 7 项全部拒绝
- [ ] 杀进程后 run 可恢复
- [ ] 重复执行写工具不产生重复副作用
- [ ] 备份可解密、可恢复、`integrity_check` 通过
- [ ] 登录限速、CSRF、会话过期生效
- [ ] 两用户 private 记忆互不可见

---

## 12. 故障排查

| 现象 | 原因 | 处理 |
|---|---|---|
| CMake 找不到 OpenSSL 3.5 | 系统 OpenSSL 太旧 | 设 `-DOPENSSL_ROOT_DIR=...`；确认 `OpenSSLConfig.cmake` |
| `s_client` 只协商 X25519 | 客户端/服务端未启用 hybrid | 检查 groups 列表与 OpenSSL 版本 |
| `sandbox-exec: operation not permitted` | profile 路径/权限 | `realpath`；允许工作区子路径；检查 SIP |
| SQLite `database is locked` | 多写者/长事务 | 单写者；`busy_timeout`；短事务 |
| 启动报缺少 ML-KEM | OpenSSL 构建裁剪过度 | 重新构建，确认未 `no-ml-kem`（不存在该选项，但确认 provider 加载） |
| 备份解密失败 | 密钥/key_id 不匹配 | 检查头部与 key registry；不要重试破坏数据 |
| 模型调用 429 | 限流 | 退避 + 暂停；不要静默切换模型 |

---

## 13. 参考

- OpenSSL 3.5 vulnerabilities: https://openssl-library.org/news/vulnerabilities-3.5/
- OpenSSL 3.5 release notes: https://openssl-library.org/news/openssl-3.5-notes/
- SQLite download: https://sqlite.org/download.html
- SQLite release log: https://sqlite.org/releaselog/current.html
- NIST FIPS 203/204/205: https://csrc.nist.gov/
- liboqs security advisory（为什么不用）: https://github.com/open-quantum-safe/liboqs/security/advisories/GHSA-f2v9-5498-2vpp
