# uniapp-vite8-compatibility

uni-app `@dcloudio/*` 编译链在 **Vite 8（rolldown 内核）** 下的兼容补丁集。uni-app 官方编译器目前仍按 vite 5 时代的假设生成配置（terser 压缩透传、esbuild 选项、alias customResolver、CJS-only 入口等），直接配 vite 8 会构建失败。本仓库用 3 个补丁完成适配，可在任何 uni-app CLI 工程中复用。

```
uniapp-vite8-compatibility/
├── README.md
└── patches/
    ├── @dcloudio__uni-cli-shared@3.0.0-5020420260813003.patch
    ├── @dcloudio__uni-mp-vite@3.0.0-5020420260813003.patch
    └── @dcloudio__vite-plugin-uni@3.0.0-5020420260813003.patch
```

## 补丁针对的版本

| 包 | 锁定版本 |
| ---- | ---- |
| `@dcloudio/uni-cli-shared` | `3.0.0-5020420260813003` |
| `@dcloudio/uni-mp-vite` | `3.0.0-5020420260813003` |
| `@dcloudio/vite-plugin-uni` | `3.0.0-5020420260813003` |
| `vite` | `^8`（验证于 8.3.x） |
| `vitest` | `^5` |

补丁后可用vitest测试，也可正常导入uni插件，构建产物主要验证微信小程序端下正确性

### 其他版本能否应用？

**可以，但需重验。** 补丁本体是标准 git unified diff：只要目标文件的上下文行没被上游改动，相邻版本通常可直接应用（`git apply` 容忍行号偏移）。注意事项：

- 各包管理器的登记键建议同步改为实际安装的版本（pnpm 的 `patchedDependencies` 键也支持版本范围）；补丁文件名不必与登记键一致，但按 `<scope>__<name>@<version>.patch` 命名最不易混淆。
- 每次升级 `@dcloudio/*` 或 `vite` 后，先用 `git apply --check <patch>` 预检，再以完整构建通过为准；上下文变了就按后文「升级与维护」重新生成补丁。
- 若上游已原生适配 vite 8，优先弃用补丁。

## 补丁内容

### 1. `@dcloudio/uni-mp-vite` — `this.resolve` 解绑

`dist/plugins/mainJs.js`：`globalComponentOptions.resolve` 由 `this.resolve` 改为 `(...args) => this.resolve(...args)`。原样传引用，后续调用时 `this` 已丢失，vite 8 下直接报错。

### 2. `@dcloudio/uni-cli-shared` — `uni:json` 放行 node_modules

`dist/vite/plugins/json.js`：id 含 `node_modules` 时直接 `return`，把依赖内的 json 交还 vite 原生 json 插件处理，避免被 uni 预处理后在 rolldown 下按 JS 解析失败。

### 3. `@dcloudio/vite-plugin-uni` — 五处适配

1. `dist/configResolved/plugins/json.js`：同上放行 node_modules 的 json；项目内 json 的输出由 `JSON.stringify(jsonObj)` 改为 `` `export default ${JSON.stringify(jsonObj ?? null)}` ``（rolldown 下必须是合法 ESM 模块）。
2. `dist/config/index.js`：移除已废弃的 `esbuild` 选项（vite 8 的 JS 转换器归 oxc/rolldown，保留会冲突）。
3. `dist/config/resolve.js`：alias 去掉 `customResolver` 与函数 replacement，改为纯 string 条目（`@` / `~@` → 源码根目录），走原生 ViteAlias。损失的能力（uts 模块 / 加密模块 / 独立分包 root 解析）mp-weixin 均不涉及；无扩展名导入由 `resolve.extensions`（含 `.vue` / `.json`）兜底。
4. `dist/configResolved/index.js`：移除 Windows 下重挂 `customResolver` 的 workaround（vite#3331 早已修复）。
5. 新增 `dist/index.mjs` 并在 `package.json` 增加 `exports`：本包是 CJS（`exports.default = uniPlugin`），`"type":"module"` 工程由 Node ESM 直载 vite.config 时，default import 拿到的是 `module.exports` 整体；经包装转发后 `import uni from '@dcloudio/vite-plugin-uni'` 才能拿到插件工厂函数。`exports` 同时附带 `"./package.json"` 子路径，防 exports 封闭拦截。

## 配套调整（补丁之外，建议）

### vite.config.ts

```ts
import uni from "@dcloudio/vite-plugin-uni";
import { defineConfig } from "vite";

export default defineConfig(({ mode }) => ({
  plugins: [uni()],
  build: {
    // uni-app 默认透传 terser（oxc压缩率更高）；
    // vite 8 中 minify: true 会归一化为内置 oxc
    minify: mode === "production",
    // rolldown 的插件耗时占比提示在 uni-app 构建下恒触发，属噪音
    rollupOptions: { checks: { pluginTimings: false } },
  },
}));
```

### 依赖覆盖（vue 必须 ≥ 3.5）

`@dcloudio/* 3.0.0-5020420260813003` 锁定的 vue / vite 插件是 vite 5 时代版本，直接安装会与 vite 8 冲突，必须用 overrides 抬版本。**vue 系列建议覆盖到 3.5 以上**（Bun 下必须）。

```jsonc
// npm：package.json "overrides"；yarn：package.json "resolutions"（同形）；
// pnpm：pnpm-workspace.yaml "overrides"（去掉外层大括号）
{
  "vite": "^8.0.0",
  "vue": "^3.5.0",
  "@vue/compiler-sfc": "^3.5.0",
  "@vue/server-renderer": "^3.5.0",
  "@vue/compiler-core": "^3.5.0",
  "@vue/compiler-dom": "^3.5.0",
  "@vue/compiler-ssr": "^3.5.0",
  "@vue/runtime-core": "^3.5.0",
  "@vue/runtime-dom": "^3.5.0",
  "@vue/reactivity": "^3.5.0",
  "@vue/shared": "^3.5.0",
  "@vitejs/plugin-vue": "^6.0.0",
  "@vitejs/plugin-vue-jsx": "^5.0.0",
  "@vitejs/plugin-legacy": "^7.0.0",
  "@vitejs/plugin-basic-ssl": "^2.0.0"
}
```

## 使用方法

### pnpm（推荐，补丁的开发环境）

1. 把 `patches/` 拷到工程根目录。
2. 登记补丁。pnpm ≥ 10 写在 `pnpm-workspace.yaml`；pnpm 9 写在 `package.json` 的 `"pnpm"` 字段：

```yaml
# pnpm-workspace.yaml
patchedDependencies:
  "@dcloudio/uni-cli-shared@3.0.0-5020420260813003": patches/@dcloudio__uni-cli-shared@3.0.0-5020420260813003.patch
  "@dcloudio/uni-mp-vite@3.0.0-5020420260813003": patches/@dcloudio__uni-mp-vite@3.0.0-5020420260813003.patch
  "@dcloudio/vite-plugin-uni@3.0.0-5020420260813003": patches/@dcloudio__vite-plugin-uni@3.0.0-5020420260813003.patch
```

3. `pnpm install`，补丁在安装期间自动应用。

后续修改补丁：`pnpm patch @dcloudio/vite-plugin-uni@3.0.0-5020420260813003` 在临时目录改文件，然后 `pnpm patch-commit <临时目录>` 重新生成并登记。

### npm（CLI v12 起，原生支持）

npm 已内置 `npm patch`：补丁存于 `patches/` 目录、登记在根 `package.json` 的 `patchedDependencies` 字段、内容 hash 记入 `package-lock.json`，安装期间应用——对传递依赖同样生效，且不受 `--ignore-scripts` 影响。任何补丁问题（应用失败 / 无匹配 / hash 不符）默认硬错误中断安装。

复用本仓库补丁：npm 生成的补丁命名恰好与 pnpm 一致（`<name>@<version>.patch`，scope 的 `/` 替换为 `__`），文件放进 `patches/` 后登记即可：

```jsonc
// package.json（示意；推荐用 npm patch 命令自动生成登记，以生成条目为准）
{
  "patchedDependencies": {
    "@dcloudio/uni-cli-shared@3.0.0-5020420260813003": "patches/@dcloudio__uni-cli-shared@3.0.0-5020420260813003.patch",
    "@dcloudio/uni-mp-vite@3.0.0-5020420260813003": "patches/@dcloudio__uni-mp-vite@3.0.0-5020420260813003.patch",
    "@dcloudio/vite-plugin-uni@3.0.0-5020420260813003": "patches/@dcloudio__vite-plugin-uni@3.0.0-5020420260813003.patch"
  }
}
```

常用命令：`npm patch add <pkg>@<ver>`（开出临时编辑目录）→ 手动改文件 → `npm patch commit <编辑目录>`（生成补丁并登记）；另有 `npm patch update`（把补丁重定基到新版本）、`npm patch ls`、`npm patch rm`。

### yarn 2+（berry，含 yarn 4）

yarn 用 `patch:` 协议登记补丁，文件默认落 `.yarn/patches/`。把本仓库补丁迁移过去最省事的路径：

```bash
yarn patch @dcloudio/vite-plugin-uni@3.0.0-5020420260813003
# 在打印出的临时目录里应用本仓库补丁：
git apply /path/to/uniapp-vite8-compatibility/patches/@dcloudio__vite-plugin-uni@3.0.0-5020420260813003.patch
yarn patch-commit -s <临时目录>
```

对三个包各做一遍。`patch-commit` 会自动生成补丁文件并改写 `package.json` 的依赖声明为 `patch:` 协议条目，无需手写。

### yarn 1 / 旧版 npm：patch-package

yarn 1 与 v12 之前的 npm 没有原生补丁能力，用 [patch-package](https://www.npmjs.com/package/patch-package)：

1. 补丁文件改名 —— patch-package 用 `+` 连接 scope 与包名：`@dcloudio__uni-cli-shared@…patch` → `@dcloudio+uni-cli-shared+3.0.0-5020420260813003.patch`，其余两个同理，仍放 `patches/`。
2. `package.json` 加 `"postinstall": "patch-package"`（并安装 `patch-package` 为 devDependency）。
3. 每次安装后自动应用；也可手动 `npx patch-package`。其底层用 `git apply`，可正常消费带 `index` 行的 git diff。

### Bun

Bun ≥ 1.3 提供 `bun patch <pkg>` / `bun patch --commit`，流程同上（编辑临时目录后提交登记）。

> ⚠️ **兼容性警告**：Bun 与 uni-app 编译链存在其他兼容性问题，**必须先把 vue 覆盖到 3.5 以上**（见前文「依赖覆盖」，`vue` 与全部 `@vue/*` 显式抬到 `^3.5.0`）才能正常工作；仅装补丁不做依赖覆盖时无法构建通过，（这个问题和补丁无关，原版也要覆盖版本）。

## 升级与维护

- 升级 `@dcloudio/*` 或 `vite` 后：
  1. `git apply --check patches/<补丁>`（或在 pnpm patch 临时目录里试应用）预检；
  2. 上下文未变 → 把登记键改成新版本（或改用版本范围键）即可；
  3. 上下文已变 → 用对应包管理器的官方流程重新生成补丁（以 pnpm 为例：`pnpm patch <pkg>@<新版本>` → 在临时目录套用同样的改动 → `pnpm patch-commit`）。
- 构建以 `uni build -p mp-weixin`（或你的目标平台）通过为准，产物用微信开发者工具导入验证。
- `@dcloudio/*` 上游若已原生适配 vite 8（补丁涉及的 `esbuild` 选项、alias、ESM 入口等均已修），应优先移除补丁回到官方实现。

## 许可证
- MIT