# 长期记忆（跨会话）

## 工作区结构
- `/Users/jyb/code/hopeio/vscode/` 父仓库下挂多个独立扩展仓库：`vscode-git`、`vscode-database`，
  各有自己的 `.git`，改动后**各自**提交推送（父仓库不必同步）。

## vscode-git 约定
- 改 `src/**`、`package.json` 后必须跑 `cd vscode-git && pnpm run package:vsix`
  （tsc → check-webview → test → vsce package，四步全过）。
- 进度写 `CHANGELOG.md`，**不要**动 `package.json` 的 version（恒为 0.0.1）。
- webview 脚本/样式一律在 `resources/webview/*.{js,css}`，禁止 TS 模板内嵌 `<script>`。
- 架构与协议约定见 `vscode-git/docs/ARCHITECTURE.md`（含 8 条关键设计决策 + 6 道构建防线）。

## VS Code 扩展开发常识（踩坑后确认）
- **虚拟文档（TextDocumentContentProvider）的 languageId 由 VS Code 按 URI path 扩展名自动推断，
  且尊重 `files.associations`**。不要为 `git-diff:` / `git-patch:` 之类手写扩展名→languageId 映射表，
  否则会覆盖用户的 `files.associations` 配置（净负收益，已回退）。
- `onDidChange` 触发后 VS Code 原地重取 `provideTextDocumentContent`：可用于「原地切换内容」而无需改
  URI（改 URI 会导致另开 tab）。补丁「忽略空白」开关即据此实现（偏好不进 URI）。

## vscode-database 约定
- 改 `src/**`、`package.json`、`syntaxes/**`、`language-configuration.json` 后必须
  `cd vscode-database && pnpm run package:vsix`。
- pnpm 11：`nodeLinker` / `allowBuilds` 只写在 `pnpm-workspace.yaml`，不放 `.npmrc`。

## vscode-git 需求进展（20 条需求关键结论，跨会话速览）
- 已落地：1/2（worktree+packed-refs 监听）、3（写操作串行锁）、4（转义统一+构建防线）、18、
  20b（补丁视图「忽略空白」开关）、9（落盘缓存 `DiskJSONCache`）、LogOutputChannel 升级、
  新增真实 CI（修正文档漂移）。
- 文档漂移修复：ARCHITECTURE.md 声称有 `.github/workflows/ci.yml`，新建该文件使其成真
  （ubuntu/macos/windows × Node 20/22 跑 `pnpm run package:vsix`）。`.github` 被 `.vscodeignore` 排除不进 vsix。
- 已回退：20a（git-diff 语言映射）——VS Code 已按 URI 扩展名自动推断语言并尊重 `files.associations`。
- 已跳过：8（cat-file 进程池）——现有 `--no-walk` 批量 + 微缓存 + facets/refs 内存缓存已抹平 N+1，
  常驻进程池风险 > 收益。
- 建议收尾：未做仅剩 7（esbuild 打包）与 20c（折叠未变更区 / 原生 diff 忽略空白）。
  - 7 当前为 tsc 直出多文件，与 `test.cjs`（require 具体 `dist` 模块）强耦合；单文件 bundle 会破坏
    测试、且 node 运行时多文件无碍，收益有限 → 不值得为打包改测试架构。
  - 20c 需改用户全局 `diffEditor.*`（折叠未变更区 / 忽略空白），影响所有 diff 有副作用；且 vscode-git
    补丁视图是自定义虚拟文档（纯文本），不受益。
  - 结论：20 条需求高价值项已全部落地，建议收尾，不再硬做低价值 / 高风险项。
