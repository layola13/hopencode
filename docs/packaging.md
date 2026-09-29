# 打包说明（mini · headless server）

`mini` 分支只保留后端 headless server：`opencode serve` + `opencode run`（非交互）。
无 TUI、无桌面/Web 前端、无内嵌 Web UI。UI 路由保留，但运行时走上游代理（无内嵌文件可 serve）。

## 三种产物

| 产物 | 打法 | 体积 |
| --- | --- | --- |
| 单平台可执行文件（含 bun runtime） | `bun run script/build.ts --single --skip-install`（在 `packages/opencode` 下） | 约 154 MB（`dist/opencode-<os>-<arch>/bin/opencode[.exe]`） |
| 纯 JS bundle（minify，无 runtime、无 webui） | 见下文“复现 JS 体积” | 约 13.7 MB（249 文件；其中 wasm 约 4.2 MB） |
| 体积历史 | 删除 `generate` 命令 + `prettier` 后 | 20.7 MB → 13.7 MB（−34%） |

> `script/build.ts` 已移除 Web UI 内嵌逻辑（原来会先 `vite build` 打 `packages/app` 再以 `opencode-web-ui.gen.ts` 形式编入二进制）。

## 复现 JS 体积

在 `packages/opencode` 下用 `Bun.build`（与 `script/build.ts` 同配置，不含 compile）：

```ts
await Bun.build({
  conditions: ["bun", "node"],
  tsconfig: "./tsconfig.json",
  external: ["node-gyp", "opencode-web-ui.gen.ts"],
  format: "esm",
  minify: true,
  sourcemap: "none",
  splitting: true,
  target: "bun",
  outdir: "<tmp>/jsbundle-headless",
  entrypoints: ["./src/index.ts"],
  define: {
    FFF_LIBC: JSON.stringify("gnu"),
    OPENCODE_VERSION: "'0.0.0-mini'",
    OPENCODE_CHANNEL: "'mini'",
    OPENCODE_LIBC: "'glibc'",
  },
})
```

`opencode-web-ui.gen.ts` 只需 external 即可：`src/server/shared/ui.ts` 对它做动态 `import(...).catch(() => null)`，
缺失时自动降级走上游代理，不影响打包。

## 体积构成（源码字节归因，min 后 JS 约 9.5 MB + wasm 4.2 MB）

| 来源 | 源码体积 | 说明 |
| --- | --- | --- |
| `effect` | 3.5 MB | 框架本身，全站构建于此，动不了 |
| 自家代码（opencode 2.1 + core 1.1 + llm/schema/…） | 约 3.5 MB | headless 核心 |
| `zod` | 0.8 MB | schema 校验，动不了 |
| 模型 Provider（`@ai-sdk/openai`、`@ai-sdk/anthropic`、`ai`、`openai`、gitlab/venice 等） | 合计约 2.5 MB | 产品本身，删即砍模型支持 |
| `@npmcli/arborist` | 0.4 MB | 插件 npm 安装（`core/src/npm.ts` 运行时动态 import） |
| smithy/aws、google-auth、MCP SDK、drizzle、`ajv`、`fastify`、otel 等 | 各 0.2–0.3 MB | 运行时刻依赖 |
| wasm | 4.2 MB | photon 图片处理 1.8、tree-sitter-bash 1.3、powershell 0.9、core 0.2 |

### 刻意没砍的两项（有实证）

- **`typescript`（8.7 MB 源码）：`packages/codemode/src/interpreter/runtime.ts` 用 `transpileModule` 做类型剥离。
  试过切 `Bun.Transpiler`，但它会在编译期拒绝 `const` 重赋值，而 V8/`transpileModule` 是放行到运行时再抛可捕获的
  `TypeError`——`codemode` 的 parity 测试（`const c = 1; c = 2` 必须能被 `catch` 到 `TypeError`）直接挂掉。
  解释器对 JS 语义 fidelity 要求高，此路不通，已回滚（codemode 263/263 全绿）。
- **`prettier`：已随 `generate` 命令删除。** `opencode generate` 只是打印 OpenAPI spec 的 dev 工具，
  与 server 无关；删命令 + 去依赖，无行为损失。

### 2 MB 是否可能

按当前结构不可能：仅 `effect` + `zod` + 自家代码 min 后就超过 3 MB，再加 Provider 和 4.2 MB wasm。
真要往 2 MB 走，需要架构级改动（Provider 按需外置、wasm 懒加载、替换 Effect），每一项都是大手术。

## 依赖闭包（13 个包）

`opencode`、`core`、`llm`、`plugin`、`protocol`、`schema`、`sdk`（含 `sdk/js`）、`server`、
`script`、`codemode`、`http-recorder`、`effect-drizzle-sqlite`、`effect-sqlite-node`。
其余包（app/desktop/web/ui/session-ui/storybook/enterprise/console/stats/slack/client/sdk-next/
httpapi-codegen/function/tui/cli）已全部删除；`containers/`（CI 构建镜像）保留。

保留但与 UI 历史相关的残留：`@opencode-ai/plugin` 的 `./tui` 导出（对外 API）、server 的 tui HttpApi
分组（纯 schema）、`@opentui/*` peerDeps。`plugin` 的 `./tui` 仍被 `opencode` 的 `plugin/tui` 胶水引用过，
该胶水已删除，导出本身仅为兼容保留。

## 验证

```sh
bun install
bun run typecheck                      # packages/opencode，0 错误
bun test --timeout 30000 test/cli/help/help-snapshots.test.ts   # 33 快照
bun test --timeout 30000 test/cli/run/run-process.test.ts        # headless 端到端 13/13
bun test --only-failures test          # packages/codemode，263/263（在 packages/codemode 下）
```
