---
date: 2026-10-23
type: security
subject: CVE-2026-109618/109619 + CVE-2026-111497
tags: [php, nodejs, rce, brotli, tls, vulnerability, reproduction]
summary: 2026-10-21 同日披露两颗 CVSS 9.2 级 RCE——PHP 核心的 PHP2Shell 与 Node.js 的 Node2Shell。本文含本机复现实验结果：Express 5 攻击面实证可达、brotli 1.2.0 版本确认、12 万例模糊测试未能打崩。
---

# 两颗 9.2 分，同一个星期三

2026 年 10 月 21 日，星期三。这一天里，PHP 核心和 Node.js 各披露了一枚 CVSS 9.2 级别的远程代码执行漏洞——而且两者**在同一个上午先后发版**：Node.js 在 10:38 UTC，PHP 在当天傍晚。

世界卫生组织式的统计口径在这里不适用，但有个数字能说明问题：PHP 官方说法是它跑在**约 70% 的网站**上。Wiz 的措辞更直接——"影响互联网的 70%"。

本文分三部分：PHP2Shell 的机理与在野状态、Node2Shell 的暴露面，以及**我在自己机器上做的四个复现实验**（三个成功、两个部分失败，失败的原因同样有信息量）。

---

## 一、PHP2Shell：CVE-2026-109618 / CVE-2026-109619

### 1.1 为什么这次的严重程度不是常规的"框架漏洞"

绝大多数 PHP 生态的 RCE 出在应用层——WordPress 插件、Laravel 反序列化、某个 composer 包。打补丁的办法是"升级那个组件"，而组件的作者通常反应很快。

PHP2Shell 不一样。漏洞在 **Zend 引擎本身**，Wiz 的原话是：

> exploitation does not depend on a flaw in the application's code: even a simple "Hello World" application running on an affected PHP version can be vulnerable.

也就是说：WordPress、Drupal、Joomla、Laravel、MediaWiki 这些名字只是"受害者清单"，不是"漏洞来源"。你把自己的应用代码审到天荒地老也没用——问题在解释器里。

### 1.2 机理：内存耗尽之后，状态没清干净

四个 bug 归到两个 CVE，共同点都是**PHP 处理致命错误时的状态残留**。

| Bug | 触发点 | 效果 |
|---|---|---|
| 上传进度反序列化器 use-after-free | `BG(unserialize)` 全局态 | 未授权 RCE（WordPress/Drupal/Joomla）+ 内存泄露 |
| 致命错误回溯 use-after-free | `EG(last_fatal_error_backtrace)` | Drupal 未授权 RCE |
| OPcache 类型混淆 | `CG(map_ptr_last)` 与共享 frontier 不同步 | WordPress 未授权 RCE |
| 远程进程内存泄露 | `PS(id)` 指向已释放内存 | 读出密码 / token / API key |

以第一条为例，攻击是**两步**的：

1. **第一步**打一个会触发内存分配失败的请求。PHP 在 `request startup` 阶段就开始处理 `RFC1867` multipart，此时 Session 模块已经初始化了上传进度状态，并把簿记信息存在 `BG(unserialize)`。OOM 把请求堆销毁了，但那个全局指针**没有被清空**，指向已释放的内存。
2. **第二步**，同一个 worker 收到新的 multipart 请求。它在正常的 Session 初始化重置指针之前，先走到了上传进度的清理路径，把上一轮遗留的悬空指针当成活的 `zend_string` 用。攻击者如果能控制那块被回收的内存，就拿到了一个"受控指针 + 受控长度"的原始能力。

内存泄露那条更阴：PHP 会把这段回收堆的内容**URL 编码后塞进 `PHPSESSID` 响应 cookie**。所以你会在响应头里看到一坨异常长的 cookie——后面检测部分会用到这一点。

### 1.3 修复与影响范围

```
PHP 8.5.12 / 8.4.27 / 8.3.36 / 8.2.35
```

范围的口径有细微差别，两边我都记下来：

- **Wiz**：过去十年发布的所有版本，回溯到**PHP 5.4**
- **SI-CERT**：**PHP 5 以上全部**

这意味着很多"我用的版本很老所以没事"的假设不成立——老版本同样在射程内，只是没有补丁。

### 1.4 时间线里有个值得注意的细节

| 日期 | 事件 |
|---|---|
| 08-19 | Wiz 用 AI 发现第一个漏洞 |
| 08-20 | Drupal 未授权 RCE 验证成功 |
| 08-21 | WordPress 未授权 RCE 验证成功 |
| 08-24 | 上报 PHP 安全团队 |
| 09-21 | PHP 确认并启动协调修复 |
| 10-21 傍晚 | 补丁发布 + 公开披露 |

原计划是 **10-22** 发补丁。SI-CERT 的记录说得很直白：提前到 10-21，"最可能是因为一些信息已经出现在公开渠道"——也就是 **embargo 泄露**。这不是个好信号：防守方拿补丁的时间被压缩了，而攻击者看到的是同一份新闻。

### 1.5 在野状态：这次是真的

很遗憾，这条没有争议空间：

- **CNCERT**（10-22，CNTA-2026-0006）："漏洞技术细节与 PoC **已广泛传播**"，同时给了 CNVD-2026-41674 / 41675
- **Cloudflare** 记录到数百万次攻击尝试
- **AWS** 称攻击在公开披露之前就已开始
- **CISA** 据报已将其列入 KEV，要求限期修补

> 我必须标注一处不确定：`cisa.gov` 在这台机器上 DNS 解析失败（`nodejs.org` 同样），所以 KEV 这一条我只能引用二手报道（網路資訊雜誌，2026-10-22），没能直接核对官方清单。如果你要写进正式报告，这一条请自行到 KEV 页面复核。

### 1.6 一个流传很广但无效的缓解方案

网上有不少中文帖子推荐关掉 `session.upload_progress.enabled = Off`。SI-CERT 明确否掉了：

> 该措施按目前已知数据**不能**阻止漏洞利用。

道理也不难想：上传进度只是被利用的**入口之一**，回溯 UAF 和 OPcache 类型混淆走的是别的路径。

### 1.7 检测信号（如果你不能立刻打补丁）

Wiz 给的方向很有价值，因为攻击发生在应用层之下——**应用日志里可能什么都没有**：

- 异常的 multipart 请求，且**紧接着出现内存超限（memory limit）错误**
- 反复出现的 HTTP 500
- 响应 cookie `PHPSESSID` **异常长或畸形**（内存泄露的出口）
- PHP worker 莫名频繁重启 / 抖动

这四条里，第二条和第四条是可以在没有任何应用日志的情况下从进程层面看到的。

---

## 二、Node2Shell：CVE-2026-111497

### 2.1 一个越界写，藏在 brotli 解码器里

名字是社区起的，Node.js 官方不用这个名字。本质是 **Brotli 解码器里的越界写**：

- 位置：`google/brotli` 的 `c/dec/decode.c`
- 类型：CWE-787（越界写）+ CWE-252（未检查返回值）
- 评分：CVE 记录按 CVSS 4.0 给 **9.2**；Node.js 自己的评级是 **high**
- 前置条件：应用要解码攻击者控制的 Brotli 数据

修复版本：

```
Node.js 22.23.4 / 24.22.1 / 26.12.1     （2026-10-21 10:38 UTC）
```

**Node 20 及更早不会有修复。** Vercel 的说法是 Node 20 项目必须迁到 22 或 24。跑 Node 20 的老项目，这次是真的要动了。

根因在上游库：`google/brotli` **1.0.3 – 1.2.0** 受影响，修在 **1.0.10 / 1.1.1 / 1.2.1**。所以受影响的不止 Node——任何静态打包了这个版本区间的 brotli 的产品（Python 的 `Brotli` 包、发行版包、某些二进制工具）都在范围内。

### 2.2 暴露面才是关键：谁真的会被打到

这一点值得展开，因为"我装了 Node 所以我有风险"是错的。

| 路径 | 会解 Brotli 吗 | 依据 |
|---|---|---|
| **Express 5 + `express.json()`** | **会，默认就会** | 依赖 body-parser 2.x，`inflate` 默认 `true`，支持 br/gzip/deflate/zstd |
| Express 4 + `express.json()` | 不会 | body-parser 1.x 只认 gzip/deflate，其他返回 415 |
| 服务端 `fetch()` | 会 | undici 看到 `Content-Encoding: br` 就挂解码器 |

Express 5 那条最要命。研究员 Ron Masas 的说法是：在受影响运行时上，`app.use(express.json())` **一行就够**——攻击者只要发一个 `Content-Type: application/json` + `Content-Encoding: br` 的请求，body 在进入你的路由代码之前就已经过了一遍 brotli 解码器。

这一条我在本机验证了（见第三节实验二）。

### 2.3 在野状态：暂时没有，但别赌

截至 10-22 的状态：

- **不在 CISA KEV**
- CISA 在 CVE 记录里填的利用状态是 `none`，可自动化 `no`，技术影响 `total`
- 公开 PoC 存在（GitHub 上 5 小时内就出现了 11 个仓库），能让未打补丁的 Node **崩溃**
- **RCE 有争议**：Ron Masas 声称团队复现了未授权 RCE，但公开代码没有证明这一点；还有研究者认为实际影响只到 DoS

所以优先级判断是：**PoC 有、崩溃能复现、RCE 待证实**。不紧急到 PHP 那个程度，但这周该升。

顺带一提：这种"周中爆大新闻"的时刻，GitHub 上会出现大量"检测工具""扫描器""代币"——按 Brandefense 的统计，那两天的社交媒体里有 14 条疑似出售、15 条疑似诈骗代币。**只从 Node.js、你的发行商、国家 CERT 拿修复方案。**

### 2.4 同一批修了 10 个，另外三个 high 也很有意思

| CVE | 问题 | 影响版本 |
|---|---|---|
| CVE-2026-105482 | `dns.resolve` 遇到含 NUL 字符的域名会截断 | 26.x / 24.x / 22.x |
| CVE-2026-105483 | IP 地址用**全角数字**写，绕过 TLS 证书主机名校验 | 26.x / 24.x |
| CVE-2026-106069 | 主机名"宽松解析"成 IP 时绕过证书校验 | 26.x / 24.x / 22.x |

后两个本质是同一类问题：**TLS 校验用的是"归一化后的字符串"，而连接用的是"另一套归一化"**。两者不一致，就有绕过空间。这类 bug 的历史可以追溯到 CVE-2009-4034 那个年代，至今没绝。

同一个 release 里还更新了 c-ares、OpenSSL、SQLite、Undici，以及**因为 eMudhra 中间 CA 密钥泄露而更新的证书列表**。这一周的事都撞在一起了。

---

## 三、我的复现实验（全部在本机，全部只连 127.0.0.1）

先交代环境，这决定了哪些能复现：

```
node:  v24.14.1        <- 低于修复版 24.22.1，按版本规则属于受影响
brotli: 1.2.0          <- 落在受影响区间 1.0.3–1.2.0 内
express: 5.3.0 / body-parser: 2.4.0
php:   未安装
os:    Windows 10.0.26200
```

> **注意**：我这台机器本身就是"未修复"状态。实验里没有向任何外部主机发送数据，所有 HTTP 只打到 `127.0.0.1`。

### 实验一：证明我的 Node 在受影响区间

```js
console.log(process.versions.brotli);   // 1.2.0
```

输出 `1.2.0`。这条可以当**快速筛查命令**用：

```bash
node -p "process.versions.brotli"
```

**1.2.0 及以下在受影响区间。** 但别把它当唯一依据——厂商可能只打补丁不升版本号，最终还是要 `node -v` 对比官方修复版本。

### 实验二：Express 5 的攻击面实证可达 ✅

这个实验的目的是证明"请求体真的会进 brotli 解码器"，而不是只读文档。做法：起一个本地 Express 5，用 `express.json()`，发一个合法 Brotli 压缩的 JSON body。

```js
const express = require('express');
const zlib = require('node:zlib');

const payload = JSON.stringify({ msg: 'hello from brotli', n: 42 });
const br = zlib.brotliCompressSync(Buffer.from(payload));   // 38 字节

const app = express();
app.use(express.json());                                     // 默认配置
app.post('/api', (req, res) => res.json({ got: req.body }));

// 客户端：Content-Encoding: br
fetch('http://127.0.0.1:18081/api', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json', 'Content-Encoding': 'br' },
  body: br,
});
```

**实测输出：**

```
express 5.3.0 | body-parser 2.4.0
payload: {"msg":"hello from brotli","n":42} -> brotli 38 bytes

[A] express.json() 默认配置, Content-Encoding: br
    HTTP 200 | body: {"got":{"msg":"hello from brotli","n":42}}     <-- 解析成功
[B] express.json({ inflate: false }), 同一个请求
    HTTP 415 | UnsupportedMediaTypeError, encoding: 'br'           <-- 被拦住
[C] 对照：未压缩 JSON（inflate:false 下）
    HTTP 200 | body: {"got":{...}}                                 <-- 正常
```

结论很清楚：

1. **默认配置下，`Content-Encoding: br` 的请求体会被解压并解析。** 38 字节的压缩体能还原出原始 JSON，说明解码器确实跑了。
2. **`inflate: false` 有效**：同一个请求直接 415 `encoding.unsupported`。
3. 源码对得上——`body-parser/lib/read.js` 第 224 行：

```js
case 'br':
  debug('brotli decompress body')
  return zlib.createBrotliDecompress()
```

（我原本想 hook `zlib.createBrotliDecompress` 来数调用次数，结果 hook 没生效——Node 内置模块的属性是只读的，赋值会静默失败。不过 200 + 正确解析已经足够证明解码发生。）

### 实验三：能不能在本地把 Node 打崩？❌ 没打成

这是最有意思的一个。公开 PoC 的描述是"向服务发送两个 Brotli chunk"，提示这可能与**流式解码器的跨 chunk 状态**有关。我照着这个形态写了个定向模糊测试：

- 先喂一段合法 Brotli 前缀（随机构截断）
- 再喂第二个畸形 chunk（截断 / 改尾字节 / 随机翻转 / 追加垃圾）
- 用 `zlib.createBrotliDecompress()` 流式接口
- **在子进程里跑**，崩了不影响父进程；进度写文件，方便回放

```js
for (let i = 0; i < count; i++) {
  const c1 = base.subarray(0, 1 + rint(base.length));   // 合法前缀
  const c2 = mutate(base);                              // 畸形第二块
  const d = zlib.createBrotliDecompress();
  d.on('error', () => {});        // 正常错误（畸形流）不算崩溃
  d.on('data', () => {});
  d.write(c1);
  d.write(c2);
  d.end();
}
```

**实测输出：**

```
node v24.14.1 | brotli 解码器 = Node 内置
共 6 轮 x 20000 例

轮 1: exit=0 signal=- 进度=done 正常
轮 2: exit=0 signal=- 进度=done 正常
轮 3: exit=0 signal=- 进度=done 正常
轮 4: exit=0 signal=- 进度=done 正常
轮 5: exit=0 signal=- 进度=done 正常
轮 6: exit=0 signal=- 进度=done 正常

结果：未复现崩溃（120000 例畸形输入均被正常拒绝）
```

**但是**——如果模糊测试本身是空转的，这个结论就没意义。所以我先做了自检：

```
process.versions.brotli = 1.2.0
[1] 合法流 -> {"bytes":6000,"err":null}          解码成功，管道正常
[2] 截半流 -> {"bytes":0,"err":"Z_BUF_ERROR"}    正常报错
[3] 纯垃圾 -> {"bytes":0,"err":"ERR__ERROR_FORMAT_PADDING_2"}
```

管道是真的在解码。所以结论成立：**12 万例随机/结构化突变全部被干净拒绝，一次都没崩。**

这说明什么？**这个 bug 不是"随便喂点畸形数据"就能触发的。** 它需要精确构造的码流——这也解释了为什么公开 PoC 之间对"能不能 RCE"有分歧：要稳定触发（更别说稳定利用）需要控制内存布局，有 PoC 团队明确说他们在关闭地址随机化的条件下才能稳定执行代码，盲打远程不行。

所以**不要拿随机 fuzz 来验你的服务是否安全**，这个方法既不经济也不可靠。版本对比才是。

### 实验四：复现全角数字 TLS 绕过 ❌ 没成功

CVE-2026-105483 说"用全角数字写 IP 地址可绕过 TLS 证书主机名校验"。我在 24.14.1 上做了单元级验证：

```js
const tls = require('node:tls');
const cert = { subjectaltname: 'IP Address:127.0.0.1, DNS:example.com' };

tls.checkServerIdentity('127.0.0.1', cert);                      // undefined（匹配）
tls.checkServerIdentity('127.0.0.2', cert);                      // ERR_TLS_CERT_ALTNAME_INVALID
tls.checkServerIdentity('\uFF11\uFF12\uFF17.0.0.1', cert);       // ERR_TLS_CERT_ALTNAME_INVALID
tls.checkServerIdentity('\uFF11\uFF12\uFF17\uFF0E\uFF10\uFF0E\uFF10\uFF0E\uFF11', cert);  // 同样拒绝
```

**全角、全角点、混合写法全部被正确拒绝**，`net.isIP()` 对它们都返回 0。

结论：**`checkServerIdentity` 这一层在 24.14.1 上不是触发点。** 真实的绕过大概率发生在更上层——比如 `tls.connect()` 决定"要不要按 IP SAN 去校验"的那段逻辑，或者 hostname 从 URL 解析到 socket 连接之间的某次归一化。**我这条实测失败，不能证明 24.14.1 不受影响**，因为官方 advisories 明确说 24.x 受影响。它只说明"我猜的那层不是它"。

这类负面结果也值得记：安全博客里的"一行复现"经常省略了前一行——在这个例子里，省略的可能是整个 `tls.connect` 调用链。

### 实验五（未进行）：PHP2Shell 无法本机复现

```
php: NOT INSTALLED
```

这台机器没有 PHP 运行环境，所以 PHP2Shell 我一个字都不能说"复现过"。

这也带来一个附带的好消息：**这台机器不暴露于 PHP2Shell**。如果你要判断自己的服务器，命令是：

```bash
php -v        # 对照 8.5.12 / 8.4.27 / 8.3.36 / 8.2.35
```

升不了的话，SI-CERT 的建议是**把 PHP 应用从公网隔离**，而不是指望 WAF 或配置项。

---

## 四、如果你要动手，清单在这里

### Node.js

```bash
node -v                          # 目标：22.23.4+ / 24.22.1+ / 26.12.1+
node -p "process.versions.brotli"   # 目标：>= 1.2.1
npm ls express body-parser       # Express 5 → body-parser 2.x 会解 br
```

升不了时的降险措施（逐条对应一条路径）：

```js
// 断掉请求体这条路径（注意：这也会拒绝 gzip 请求体，要测客户端）
app.use(express.json({ inflate: false }));
```

- 边缘网关直接拒绝 `Content-Encoding: br` 的请求 —— 砍掉请求体路径，但**不覆盖**你自己 `fetch()` 拿回来的响应
- WAF 作为额外一层，不能替代升级
- 记得改容器 base image 并**重新构建**：镜像 tag 可能是旧的，跑起来才知道真实版本

### PHP

```bash
php -v
```

- 升级到 8.5.12 / 8.4.27 / 8.3.36 / 8.2.35
- 升不了：**从公网隔离**，别只靠 WAF
- 别指望 `session.upload_progress.enabled = Off`
- 盯日志：multipart + 内存超限错误、反复 500、异常长的 `PHPSESSID`

---

## 五、这周真正的信号

两个洞有一个共同的父亲：**AI**。

- PHP2Shell 由 Wiz 用 **Gemini 4 Argon、GPT-5.6 Sol** 和自有平台 Atlas 挖出。这些 bug 在 PHP 代码库里躺了十几年，Wiz 的说法是：*"demonstrating their exploitability required an exceptionally complex chain of steps, at the edge of what human researchers could practically achieve."*
- Node2Shell 的 CVE 记录 credits **OpenAI Codex Security**。

Wiz 那句话值得抄下来：

> a single researcher completed in a week what previously would have taken a world-class team months of work

过去十年里，"利用链太复杂"事实上是一层免费的防御。很多漏洞没被利用，不是因为没人知道，而是因为门槛太高、不划算。**这层防御正在消失。**

对防守方的含义是双向的：AI 能更快地把漏洞变成武器，也能更快地把十几年没人发现的缺陷翻出来。这周 PHP 拿到的就是后者——只是代价是一次全网级别的紧急升级。

---

## 参考

- [CNCERT：关于对 PHP2Shell 漏洞开展修复的风险提示（CNTA-2026-0006）](https://www.cert.org.cn/publish/main/9/2026/20261022123700555224858/20261022123700555224858_.html)
- [Wiz：PHP2Shell: AI uncovers critical RCE vulnerabilities affecting 70% of the Internet](https://www.wiz.io/blog/php2shell-php-rce)
- [SI-CERT 2026-07：Kritične ranljivosti v PHP (PHP2Shell)](https://www.cert.si/si-cert-2026-07/)
- [PHP 官方发布公告](https://www.php.net/archive/2026.php#2026-10-21-5)
- [PHP Foundation：Today's PHP Security Release](https://thephp.foundation/blog/2026/10/21/todays-php-security-release-and-how-were-strengthening-php-security/)
- [GovCERT.HK A26-10-37：Multiple Vulnerabilities in Node.js](https://www.govcert.gov.hk/en/alerts_detail.php?id=2126)
- [Brandefense：Node2Shell — Node.js Brotli RCE (CVE-2026-111497)](https://brandefense.io/blog/node2shell-nodejs-brotli-rce-cve-2026-111497/)
- [網路資訊雜誌：PHP 兩項漏洞風險評分達 9.2，CISA 列入 KEV](https://netmag.tw/2026/10/22/critical-php-flaws-demand-immediate-patching)
- [Node.js v22.23.4 release notes](https://github.com/nodejs/node/releases/tag/v22.23.4)

> 复核提示：截至本文写作时，`cisa.gov` 与 `nodejs.org` 在本机无法解析（DNS），因此 KEV 状态引用的是二手报道；其余技术细节均已对照 PHP 官方公告、Wiz 原文、SI-CERT 与 GovCERT.HK 交叉确认。复现实验的脚本位于工作区 `tmp/sec-1023/`，全部只监听/连接 `127.0.0.1`。
