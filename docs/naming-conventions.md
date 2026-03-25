# 命名规范

## 文件夹命名

- 使用 **kebab-case**：全小写，单词之间用 `-` 连接
- 示例：`code-review`、`smart-commit`、`test-generator`

## Skill vs Command 的区别

| 类型 | 定位 | 示例 |
|------|------|------|
| Skill | 可复用的能力模块，描述 AI 应具备的能力 | `code-review`、`simplify` |
| Command | 可执行的具体指令，完成特定任务 | `commit`、`deploy-check` |

## 文件命名

| 文件 | 说明 |
|------|------|
| `README.md` | 使用说明文档 |
| `skill.md` | Skill 的 prompt 内容 |
| `command.md` | Command 的 prompt 内容 |
| `examples/example-01.md` | 示例文件，按序号命名 |
