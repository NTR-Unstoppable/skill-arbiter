# Skill Arbiter — 安装规则（中文）

将此内容插入 `~/.claude/CLAUDE.md` 的**最顶部**（在所有其他规则之前）。

不加此规则：拦截率 ≈ 50%。加了：≈ 80%。

---

## 0. Skill 调用拦截规则（最高优先级，先于所有其他规则）

**禁止直接调用除 `skill-arbiter` 之外的任何 skill。**

### 正确流程

```
用户请求 → skill-arbiter（仲裁） → arbiter 选择并调用目标 skill
```

### 禁止行为

```
用户请求 → 直接调用 nature-polishing / cnki-search / code-review / ...
         或其他任何非 skill-arbiter 的 skill  ← 违规
```

### 唯一豁免

**`skill-arbiter` 自身**是唯一豁免项。调用 `skill-arbiter` 不需要先经过仲裁。

### 违规后果

如果直接调用了非 arbiter 的 skill，用户会看到未经仲裁的 skill 执行结果，
且无法享受 arbiter 的冲突检测和多 skill 对比。
