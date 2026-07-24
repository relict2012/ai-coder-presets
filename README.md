# ai-coder-presets

Claude Code + CCSwitch 个人预设仓库。 存放自定义Skill、MCP服务配置、Agent行为规则、系统提示、编码规范、可复用代码片段与部署脚本。 



### 目录结构

```
ai-coder-presets/
├─ skills/                # 本地Skill文本（CCSwitch导入使用）
│  ├─ SKILL.md
│  └─ SkillName/
│     ├─ data/            # 使用数据
│     ├─ references/      # 引用清单 
│     ├─ scripts/         # 执行脚本
│     └─ SKILL.md
├─ mcp-config/            # MCP服务配置片段、启动脚本
│  ├─ xxx.json
│  └─ xxx2.ps1
├─ agent-rules/           # 全局系统提示、行为约束
│  ├─ xxx.md
│  └─ xxx2.md
├─ templates/             # 代码/配置模板
│  ├─ xxx.xml
│  ├─ xxx2.json
│  ├─ xxx3.cs
│  └─ xxx4.ps1
├─ workspace-claude-md/   # 各个项目claude.md模板
│  └─ xxx2.claude.md
└─ docs/
   ├─ mcp-enable-guide.md
   └─ skill-usage-notes.md
```

## 备忘录
### Skills
1. SKILL.md正文：是路由器，不是仓库,不应该在 `SKILL.md` 正文中添加太多内容，除非整个Skill本身就很小
2. 在对应的skill上添加`disable-model-invocation: true`,可以阻止Agent主动触发技能，只允许用户手动触发
3. 
