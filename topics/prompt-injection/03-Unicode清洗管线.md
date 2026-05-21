# 03 Unicode 清洗管线

> 防御 prompt-injection 的字符级闸门。处理那些"用户/模型/审计人员看不见，但模型能读懂"的隐藏字符攻击。

## 3.1 为什么需要 Unicode 清洗

LLM 用 BPE 分词器处理 Unicode 文本。这意味着：

- **零宽字符**（zero-width space U+200B、zero-width joiner U+200D 等）会进入 token 流；
- **双向控制字符**（RLO/LRO/PDI/RLI U+2066-2069 等）可能改变模型对"文本顺序"的理解；
- **Unicode Tag 字符**（U+E0000-E007F）肉眼完全不可见，但模型能识别；
- **Private Use Area**（U+E000-F8FF、U+F0000-FFFFD 等）可以放任意约定字符；
- **变音符号叠加**（如 `á̂...` 多层组合）能让显示看起来正常但 token 化结果不同。

攻击模型：

> "用户在 GitHub issue 里粘了一段看起来正常的文本，但其中插了 Tag 字符 + ASCII Smuggling 编码的指令。模型读完之后执行了攻击者的指令，用户全程不知道发生了什么。"

这不是理论威胁。`sanitization.ts:10-14` 直接引用了 HackerOne #3086545：

```ts
/**
 * The vulnerability was demonstrated in HackerOne report #3086545 targeting
 * Claude Desktop's MCP (Model Context Protocol) implementation, where attackers
 * could inject hidden instructions using Unicode Tag characters that would be
 * executed by Claude but remain invisible to users.
 *
 * Reference: https://embracethered.com/blog/posts/2024/hiding-and-finding-text-with-unicode-tags/
 */
```

也就是说——这套防御**是事后补丁**。Anthropic 在 Claude Desktop 上被 HackerOne 报告过这个漏洞，Claude Code 在源码层显式标注了"这是为了修这个洞的"。

## 3.2 `sanitization.ts` 91 行逐行精读

整个文件就两个导出函数：`partiallySanitizeUnicode` 和 `recursivelySanitizeUnicode`。我们逐段拆。

### 3.2.1 文件头注释（1-23 行）

```ts
/**
 * Unicode Sanitization for Hidden Character Attack Mitigation
 *
 * This module implements security measures against Unicode-based hidden character attacks,
 * specifically targeting ASCII Smuggling and Hidden Prompt Injection vulnerabilities.
 * These attacks use invisible Unicode characters (such as Tag characters, format controls,
 * private use areas, and noncharacters) to hide malicious instructions that are invisible
 * to users but processed by AI models.
 *
 * The vulnerability was demonstrated in HackerOne report #3086545 targeting Claude Desktop's
 * MCP (Model Context Protocol) implementation, where attackers could inject hidden instructions
 * using Unicode Tag characters that would be executed by Claude but remain invisible to users.
 *
 * Reference: https://embracethered.com/blog/posts/2024/hiding-and-finding-text-with-unicode-tags/
 *
 * This implementation provides comprehensive protection by:
 * 1. Applying NFKC Unicode normalization to handle composed character sequences
 * 2. Removing dangerous Unicode categories while preserving legitimate text and formatting
 * 3. Supporting recursive sanitization of complex nested data structures
 * 4. Maintaining performance with efficient regex processing
 *
 * The sanitization is always enabled to protect against these attacks.
 */
```

四个设计承诺：
1. **NFKC 标准化**——处理"看起来一样但编码不同"的字符
2. **按 Unicode 类别移除**——而不是逐个枚举字符
3. **递归处理嵌套结构**——对象 / 数组 / 字符串都过
4. **性能足够好**——能在每个用户输入路径上执行

注意最后一句 "The sanitization is always enabled"——**没有 feature gate**。这条防御是常开的。

### 3.2.2 `partiallySanitizeUnicode` 主体（25-65 行）

```ts
export function partiallySanitizeUnicode(prompt: string): string {
  let current = prompt
  let previous = ''
  let iterations = 0
  const MAX_ITERATIONS = 10 // Safety limit to prevent infinite loops

  // Iteratively sanitize until no more changes occur or max iterations reached
  while (current !== previous && iterations < MAX_ITERATIONS) {
    previous = current

    // Apply NFKC normalization to handle composed character sequences
    current = current.normalize('NFKC')

    // Remove dangerous Unicode categories using explicit character ranges

    // Method 1: Strip dangerous Unicode property classes
    // This is the primary defence and is the solution that is widely used in OSS libraries.
    current = current.replace(/[\p{Cf}\p{Co}\p{Cn}]/gu, '')

    // Method 2: Explicit character ranges. There are some subtle issues with the above method
    // failing in certain environments that don't support regexes for unicode property classes,
    // so we also implement a fallback that strips out some specifically known dangerous ranges.
    current = current
      .replace(/[​-‏]/g, '') // Zero-width spaces, LTR/RTL marks
      .replace(/[‪-‮]/g, '') // Directional formatting characters
      .replace(/[⁦-⁩]/g, '') // Directional isolates
      .replace(/[﻿]/g, '') // Byte order mark
      .replace(/[-]/g, '') // Basic Multilingual Plane private use

    iterations++
  }

  // If we hit max iterations, crash loudly.
  if (iterations >= MAX_ITERATIONS) {
    throw new Error(
      `Unicode sanitization reached maximum iterations (${MAX_ITERATIONS}) for input: ${prompt.slice(0, 100)}`,
    )
  }

  return current
}
```

逐段拆解：

#### 不动点迭代（line 26-32）

```ts
let current = prompt
let previous = ''
let iterations = 0
const MAX_ITERATIONS = 10

while (current !== previous && iterations < MAX_ITERATIONS) {
  previous = current
  // ... 处理 ...
  iterations++
}
```

为什么需要循环？因为 NFKC + 删字符这两步本身不是**幂等独立**的：

- NFKC 可能把组合字符**分解或合并**，产生新的字符；
- 删字符可能让相邻字符**重新组合**成新的复合字符；
- 上一轮残留的字符可能被这一轮处理掉。

举例：

```
原文：a​́
          ↑     ↑
       ZWSP   组合重音
```

NFKC 不会把它合并成 `á`，因为 ZWSP 在中间。但**先删掉 ZWSP** 再 NFKC，`a` 和 `́` 就能组合成 `á`。如果只跑一遍可能漏掉。所以要迭代直到稳定。

`MAX_ITERATIONS = 10` 是安全阀。注释（57-62 行）说：

> If we hit max iterations, crash loudly. This should only ever happen if there is a bug or if someone purposefully created a deeply nested unicode string.

正常输入 1-2 轮就稳定。攻击者构造的深度嵌套 Unicode 序列才会触发 throw。**Fail loudly** 是好习惯——攻击企图能被检测到，而不是被静默吞掉。

#### NFKC 标准化（line 36）

```ts
current = current.normalize('NFKC')
```

Unicode 有 4 种标准化形式：NFC / NFD / NFKC / NFKD。NFKC 是 **compatibility composed**：

- **Compatibility**：把"看起来一样"的字符变成"编码也一样"。例如：
  - `①` (U+2460) → `1`
  - `Ⅴ` (U+2164) → `V`
  - `（` (U+FF08) → `(`
  - `ﬁ` (U+FB01) → `fi`
- **Composed**：组合字符 → 单一码点
  - `a` + `́` → `á`

为什么选 NFKC 而不是 NFC？因为 **NFKC 把 ASCII 等价的全角/特殊形式也归一化成 ASCII**。攻击者用 `Ⅴ` 假装是 `V` 来绕过关键词匹配——NFKC 直接把它变成真 `V`。

#### Method 1：按 Unicode 类别批量删除（line 42）

```ts
current = current.replace(/[\p{Cf}\p{Co}\p{Cn}]/gu, '')
```

正则用了 Unicode 属性类（需要 `u` flag）。三个类：

- **`\p{Cf}`** — Format 类：包括 ZWSP、ZWJ、ZWNJ、Bidi 控制符、Unicode Tag chars (U+E0000+) 等。**这是 Tag-char 攻击的主战场。**
- **`\p{Co}`** — Private Use 类：U+E000-F8FF、U+F0000-FFFFD、U+100000-10FFFD。攻击者可以在这些区段约定任意字符做秘密通信。
- **`\p{Cn}`** — Unassigned 类：还没分配含义的码点。攻击者用这些"未来字符"做隐写。

这一行很短，但杀伤力极大。它**精确切除三大类隐写攻击载体**，而不需要枚举具体码点。

#### Method 2：显式范围兜底（line 47-52）

```ts
current = current
  .replace(/[​-‏]/g, '') // Zero-width spaces, LTR/RTL marks
  .replace(/[‪-‮]/g, '') // Directional formatting characters
  .replace(/[⁦-⁩]/g, '') // Directional isolates
  .replace(/[﻿]/g, '') // Byte order mark
  .replace(/[-]/g, '') // Basic Multilingual Plane private use
```

注释解释为什么需要：

> some subtle issues with the above method failing in certain environments that don't support regexes for unicode property classes

也就是说：**Method 1 的 `\p{Cf}` 在某些 JS 引擎可能不工作**（Node 版本、Bun、嵌入式 JS 等）。这一段是 fallback，显式枚举已知最危险的 5 段范围：

| 范围 | 含义 | 攻击形态 |
|---|---|---|
| U+200B-200F | ZWSP / ZWNJ / ZWJ / LRM / RLM | 零宽字符隐写、单词边界混淆 |
| U+202A-202E | LRE/RLE/PDF/LRO/RLO | 双向覆盖攻击（[Trojan Source](https://trojansource.codes/)） |
| U+2066-2069 | LRI/RLI/FSI/PDI | 隔离符攻击（Trojan Source 第 2 代） |
| U+FEFF | BOM | 不可见 token 漂移 |
| U+E000-F8FF | BMP Private Use | 自定义隐写编码 |

注意：这些都是 `\p{Cf}` 和 `\p{Co}` 的子集——Method 2 是冗余备份，确保即使 Method 1 失效也能切掉最危险的几段。

### 3.2.3 `recursivelySanitizeUnicode`（67-91 行）

```ts
export function recursivelySanitizeUnicode(value: string): string
export function recursivelySanitizeUnicode<T>(value: T[]): T[]
export function recursivelySanitizeUnicode<T extends object>(value: T): T
export function recursivelySanitizeUnicode<T>(value: T): T
export function recursivelySanitizeUnicode(value: unknown): unknown {
  if (typeof value === 'string') {
    return partiallySanitizeUnicode(value)
  }

  if (Array.isArray(value)) {
    return value.map(recursivelySanitizeUnicode)
  }

  if (value !== null && typeof value === 'object') {
    const sanitized: Record<string, unknown> = {}
    for (const [key, val] of Object.entries(value)) {
      sanitized[recursivelySanitizeUnicode(key)] =
        recursivelySanitizeUnicode(val)
    }
    return sanitized
  }

  // Return other primitive values (numbers, booleans, null, undefined) unchanged
  return value
}
```

要点：

1. **TypeScript 重载签名**：调用方知道返回类型与输入类型一致（string → string、T[] → T[] 等），不丢类型信息；
2. **递归三态**：字符串 → 清洗；数组 → 逐元素递归；对象 → 键 + 值都递归；
3. **键也清洗**（line 83）——一个常被忽视的细节。攻击者可能在 JSON object 的 key 里放隐藏字符，这里也覆盖到了；
4. **基础值跳过**：number / boolean / null / undefined 直接返回（没有清洗空间）。

## 3.3 这套清洗在哪些路径上跑

`recursivelySanitizeUnicode` 和 `partiallySanitizeUnicode` 在源码里的调用点：

| 调用点 | 文件:行 | 角色 |
|---|---|---|
| MCP tools 列表 | `services/mcp/client.ts:1758` | 第三方 MCP server 返回的工具描述 |
| MCP prompts 列表 | `services/mcp/client.ts:2051` | 第三方 MCP server 返回的 prompt 描述 |
| Deep link 解析 | `utils/deepLink/parseDeepLink.ts:141` | 来自 OS 的 deep link 参数（可能由其他 app 构造） |
| `/tag` 命令 | `commands/tag/tag.tsx:82` | 用户输入的 tag 名称 |

注意一个关键观察：**这个清洗不是在用户消息入口跑的**。普通用户输入（在 prompt 框里打字）**没有**走这套清洗。

为什么？分析下来有两个解释：

1. **信任边界**：用户在自己机器上打字、粘贴的内容算"自己人"，不视为攻击源；
2. **可见性假设**：终端会渲染零宽字符为可见占位符（虽然不完整），用户至少有机会察觉。

但 **MCP / deep link / tag 命令**就不一样——这些是**外部输入**：

- MCP server 是用户从 marketplace 安装的第三方代码（可能不可信）；
- Deep link 是其他应用通过 OS 发过来的（可能被攻击者构造）；
- `/tag` 命令的参数虽然由用户输入，但可能用户从某网页复制了带隐藏字符的文本。

所以清洗只覆盖"半信任"和"不信任"的入口。这是基于**威胁模型**而不是"全量覆盖"的实用主义设计。

## 3.4 三个关键设计判断

读完源码可以提炼几个设计判断：

### 判断 1：常开，不要 feature gate

文件头注释最后一句：

> The sanitization is always enabled to protect against these attacks.

很多 Claude Code 的防御机制有 feature gate（`tengu_chair_sermon` 等）。但**这一条没有**——因为它针对的是真实存在的零日漏洞，关掉它就会暴露已知 CVE。这种"反 CVE"的安全机制不该有 kill switch。

### 判断 2：迭代到不动点，且 fail loudly

```ts
if (iterations >= MAX_ITERATIONS) {
  throw new Error(...)
}
```

为什么 throw 而不是返回当前结果？因为**到达 10 轮还没稳定 = 不正常**，要么是 bug 要么是恶意构造。**让上层错误处理拦截**比静默放过更安全——后者会让攻击者通过设计深度嵌套的 Unicode 来探测/绕过清洗。

### 判断 3：Method 1 + Method 2 双保险

按 Unicode 类别（Method 1）的覆盖更广更新更稳，但依赖 JS 引擎对 `\p{}` 的支持。显式范围（Method 2）覆盖窄但兼容性 100%。两层叠加：

- 现代环境：Method 1 已经全部清掉，Method 2 是 no-op；
- 旧环境：Method 1 可能失效，Method 2 兜住已知最危险的几段。

这是面向真实工程环境（不是控制好的实验室）的成熟做法。

## 3.5 局限和未覆盖的攻击向量

诚实评估：

### 局限 1：不覆盖普通 user prompt 路径

如 3.3 所述，用户在 prompt 框里打字不走清洗。如果用户粘贴了带 Tag chars 的文本，模型还是会看到原始字符。

**为什么这样设计**：清洗会改变用户原文，破坏可逆性（用户可能想要的就是 unicode 原文）。在用户路径上做静默修改有可用性代价。

### 局限 2：不清洗工具输出

FileRead 输出、Bash 输出、Web 抓取内容**不**走 sanitization。这些里面的 Unicode 攻击不被这条管线拦截。

**为什么这样设计**：工具输出可能合法地包含特殊 Unicode（如 emoji、CJK 字符、源码里的零宽字符在某些 DSL 里有意义）。在这里清洗会破坏工具的合法用途。

### 局限 3：不覆盖 base64 / 编码后的指令

攻击者可以把指令 base64 编码再放到正常文本里，sanitization 完全看不到。但这种攻击需要模型主动 decode，门槛较高。

### 局限 4：不处理同形字（homoglyph）

`partiallySanitizeUnicode` 不替换"看起来像 ASCII 但是 Cyrillic 字母"这种同形字（如 Cyrillic `а` U+0430 vs Latin `a` U+0061）。NFKC 也不处理这类。

**为什么这样设计**：同形字在合法多语言文本里非常常见，替换会破坏正常中文/俄语/希腊语等内容。同形字攻击通常需要配合其他向量才能成功，单独不是高优先级。

## 3.6 一句话总结

- `sanitization.ts` 91 行实现了字符级的 prompt-injection 兜底；
- 是为了修补 HackerOne #3086545（Tag-char 攻击）而加的，**always-on 无 feature gate**；
- 用 NFKC + 三大类 Unicode 类别 + 显式范围三重防御；
- 迭代到不动点，超过 10 轮 fail loudly；
- 只覆盖 MCP / deep link / tag 等"外部输入"路径，不覆盖正常 user prompt 和工具输出——基于威胁模型而非全量覆盖。

下一篇 → [04 attachments 与 surfacer 预算](./04-attachments与surfacer预算.md)
