# 多 agent 入口用「一份真身 + 薄壳」，不写三份重复内容

系统要同时被 Codex、Gemini、WorkBuddy 等不同 agent 读取，而各家认的入口文件名不同。曾考虑为每个 agent 各写一份完整规则，但重复文档一定会漂移，且历史上有过「入口文件说 A、实际规则是 B」的事故（改一处漏改一处）。因此真身只有一份，其余入口都只是指向它的薄壳。

实测事实（2026-10-13，本机）：Codex CLI 0.160.0 **默认**读取 `AGENTS.md`（全局 `~/.codex/AGENTS.md` + 项目逐级合并），无需配置；Gemini CLI 0.57.0 **默认不读** `AGENTS.md`，需要在 `settings.json` 里显式设置 `contextFileName`（或新版 `context.fileName`）为 `AGENTS.md` 才会读。所以薄壳策略不是零成本的——每个非默认 agent 都要一次性的配置动作。

## 修订：入口是三层，且入口文件有 8000 字符硬上限

**修正（2026-10-13 二次实测，推翻本节此前的推测）**：WorkBuddy **会**读项目级 `AGENTS.md`。此前判断"不读"是错的——那是检索方法本身失效导致的假阴性（见本节末尾）。

从 WorkBuddy 打包源码中直接读到的加载逻辑（`packages/workbuddy-server/src/prompts/user/sections/project-context-section.ts`）：

```js
var GUIDANCE_FILES = ["CODEBUDDY.md", ".codebuddy/CODEBUDDY.md", "AGENTS.md"];
var MAX_GUIDANCE_CHARS = 8e3;
async function readProjectGuidance(cwd) {
  for (const candidate of GUIDANCE_FILES) {
    const target = path.join(cwd, candidate);
    try {
      const raw = await fs.readFile(target, "utf8");
      return wrap("project_guidance", raw.length > 8000 ? raw.slice(0, 8000) + "\n[...too long, omitted...]" : raw);
    } catch { continue; }
  }
  return "";
}
```

四条必须写进设计的结论：

1. **`AGENTS.md` 属于自动层**——WorkBuddy 和 Codex 都无需任何配置就读。
2. **只加载第一个命中的文件**（循环里直接 `return`，不合并）。若包内同时存在 `CODEBUDDY.md`，它会盖住 `AGENTS.md`。因此**本包只放 `AGENTS.md`**，不放任何别名。
3. **只读工作目录根这一层**，不向上遍历、也不递归子目录。这把 ADR-0011 的「包根 = 工作目录」前提从约定升级成了源码级事实。
4. **入口文件有 8000 字符硬上限**，超出直接截断。这是全系统最紧的一条约束（Codex 是 32 KiB，更宽松），所以**入口文件按 8000 字符来写**，细节一律外置。

另有一条省字数的发现：WorkBuddy 会自动注入 `<project_layout>`（工作目录的目录快照，最多 40 个可见条目、忽略隐藏项、子树深度 6），因此**入口文件不需要罗列目录树**，那些字符留给规则本身。

## 三层入口

- **自动层**：Codex、WorkBuddy 默认读 `AGENTS.md`。
- **配置层**：Gemini CLI 需在 `.gemini/settings.json` 里把 `contextFileName` 指到 `AGENTS.md`。
- **兜底层**：豆包电脑版、智谱清言、Trae 等既不认 `AGENTS.md` 也没有可配置的等价入口。对它们，README 里给用户**一句话起步指令**——「请读 `AGENTS.md`，并严格按它的规则工作」。

## 取证方法记录（避免重走弯路）

- **有效**：解出 asar 头部的文件清单后，对原始字节做 `count`，并且**必须带对照组**——用 `SKILL.md` 验证方法有效（实测命中 227 次，`AGENTS.md` 命中 2 次且落在源码里）。
- **无效**：对 `app.asar` 直接用 `rg` 检索——连确定存在的 `SKILL.md` 都零命中，两次都给出假阴性，已弃用。

