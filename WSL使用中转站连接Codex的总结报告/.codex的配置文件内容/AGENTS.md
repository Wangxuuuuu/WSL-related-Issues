# AGENTS.md (Global)

## Communication
- 默认使用中文回复；仅在用户明确要求时切换语言。
- 先获取上下文再动手；不确定需求时先澄清。

## Working Style
- 优先最小改动，直接修复根因，避免过度设计。
- 先读后改：先定位相关文件与调用链，再实施修改。
- 变更后做最小可行验证；无法本地验证时明确说明并给出可执行命令。

## JavaScript / TypeScript Conventions
- 文件名使用小写并以下划线分隔。
- 定义方法优先使用 const，不要使用 function 声明式写法。
- 注释使用单行注释，避免无意义注释。
- 第三方库通过 @/libs 统一导入；第三方导入在上、项目导入在下，中间空一行。

## Defaults / Fallbacks
- 禁止对内部配置、环境变量使用 || 或类似方式兜底。
- 仅对外部不可信输入做必要的默认值与防御处理。

## Tooling Preferences
- 优先用只读命令获取上下文（如 rg、ls、cat）。
- 需要安装依赖、构建时，先说明影响并与用户确认。

## Safety
- 非用户明确要求，不执行破坏性操作（如 rm、git reset --hard）。
- 不因"看起来更整洁"而修改无关代码。

## Git 操作规范
- **禁止破坏性 git 命令**：git checkout <file>、git reset --hard、git clean -f、git restore <file> 等会丢弃修改的命令，除非用户明确授权。
- **文件修复优先级**：Read + Edit/Write > sed/awk（需验证）> Python 脚本 > git 命令（仅查询）。
- **批量修改前验证**：使用 sed/awk 前先预览或在测试文件验证，修改后立即检查语法。
- **检查依赖关系**：修改文件前用 Grep 搜索导入关系，确保不破坏其他文件。
- **错误恢复原则**：操作失误时不要用 git 回滚，先用 git diff 查看丢失内容，手动恢复或请求用户帮助。

## Priority
- 若项目内存在更具体的 AGENTS.md / 项目规则文件，以项目规则优先。