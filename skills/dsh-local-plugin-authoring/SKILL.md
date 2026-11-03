---
name: dsh-local-plugin-authoring
description: 在本机新建一个非 npm 的 DSH 插件并挂进 profile（含把新页面挂进「技能管理」内存源页签）——先读已装插件当契约样本，写 package.json（main + exports {"./client"}，页面型再声明 dsh.client），宿主 lib/index.js 用 execFileSync 调本机脚本并注册路由，客户端用 window.__ModuleLoader__.load 注册槽位或 sourceTypeId/pageId，最后向 ~/.dsh/profiles/desktop/cordis.patch.yml 追加 insert 条目并用 node --check、假 ctx 跑 apply()、createRequire 解析三重验收。当用户说「新建/写一个本地 DSH 插件」「不用发 npm 的插件」「给侧边栏/技能管理加个页面」「挂进 profile」「插件没生效」时使用。
whenToUse: 用户要做的是「本机 profile 里的手写插件」（node_modules 目录 + cordis.patch.yml 注册），而不是发布到插件市场（那条走 dsh-skill-market-publish），也不是改第三方插件源码后回提上游（那条走 dsh-plugin-upstream-contribute）。典型触发语：给技能管理加一个「发布」页、侧边栏底部加一个状态条、写个本地插件读本机脚本、插件挂进 profile 后怎么确认加载了。
version: 1.0.0
---

# 本地 DSH 插件手写与挂载（非 npm）

本技能只针对**本机手写插件**：文件直接落在 `~/.dsh/profiles/desktop/node_modules/<plugin-name>/`，不发布 npm，靠 profile 的 `cordis.patch.yml` 顶层数组 `insert` 条目被 loader 读入。验收全部用「不启服务也能跑」的机器可核手段。

## Steps

### A. 读契约样本（先看已装插件怎么写，不要凭记忆造 API）

1. 确认目标 profile 与插件目录：`ls ~/.dsh/profiles/desktop/node_modules/ | head -30`。下文以 `~/.dsh/profiles/desktop` 为 profile 根（写死前先核实本机实际 profile 名）。
2. 读宿主半契约样本（侧边栏/浮层型）：
   ```bash
   grep -n "^export\|^const inject\|^export function apply\|^export const" ~/.dsh/plugins/dsh-balance-panel/lib/index.js | head
   ```
   要点：`const inject = ["credentials", "webServer"]`，文件尾 `export { name, inject, apply }`。
3. 读路由注册写法：`sed -n '1285,1310p' ~/.dsh/plugins/dsh-balance-panel/lib/index.js`（`webServer.register({ path, kind, handler })`，handler 里 `sendJson(res, 200, ...)` / `sendJson(res, 500, { ok:false, error, message })`）。
4. 读客户端半契约样本：`head -40 ~/.dsh/plugins/dsh-balance-panel/lib/client.js`，再 `grep -n "slots.register\|slots.inject\|name: \"" .../lib/client.js | head -20`。要点：`window.__ModuleLoader__.load({ id, factory: (require) => {...} })`，槽位名形如 `sidebar.footer.action`。
5. 页面型插件（要往「技能管理」加页签）额外读内存源页契约：
   ```bash
   grep -n "MemorySourcePageFrame\|installMemorySourceUI" ~/.dsh/profiles/desktop/node_modules/dsh-mnemon/lib/client.js | head -5
   grep -n "exports\." ~/.dsh/profiles/desktop/node_modules/dsh-mnemon/lib/client.js | tail -12
   ```
   证据里 `installMemorySourceUI`（约 10229 行）与 `MemorySourcePageFrame`（约 10534 行，导出于 `exports.MemorySourcePageFrame`）就是页签注册与外壳复用的公开出口；行号随版本变，用 grep 重新定位。
6. 页面型还要读 `dsh.client` 声明格式（不要自己发明键名）：
   ```bash
   python -c "import json;d=json.load(open(r'C:/Users/<user>/.dsh/profiles/desktop/node_modules/dsh-mnemon-source-skills/package.json',encoding='utf-8'));print(json.dumps(d.get('dsh'),ensure_ascii=False,indent=1))"
   ```
   证据输出：`{"client": {"platform": "web", "external": ["dsh-mnemon/client"]}}`。

### B. 写插件三件套

7. 目录：`mkdir -p ~/.dsh/profiles/desktop/node_modules/<plugin-name>/lib`。
8. `package.json`（`main` 指宿主入口，`exports["./client"]` 指浏览器半，这是宿主读客户端文件的唯一路径；`private: true` + `type: module`）：
   ```json
   {
     "name": "dsh-skill-publisher",
     "version": "0.1.0",
     "private": true,
     "description": "<一句话>",
     "type": "module",
     "main": "lib/index.js",
     "exports": {
       ".": "./lib/index.js",
       "./client": "./lib/client.js",
       "./package.json": "./package.json"
     },
     "dsh": { "client": { "platform": "web", "external": ["dsh-mnemon/client"] } }
   }
   ```
   `dsh.client` 段**只对页面型客户端**（复用 dsh-mnemon 外壳）必需；`external` 里的模块名必须是 profile 内可解析的包名。
9. 宿主 `lib/index.js` 头部：
   ```js
   import { execFileSync } from "<Node 子进程模块>";
   export const name = "<plugin-name>";
   export const inject = ["webServer"]; // 需要凭据再加 "credentials"
   ```
   数据来源一律是本机脚本/文件，用 `execFileSync(process.execPath | "python", [script, ...args], { encoding: "utf-8", timeout: ... })`，不要在插件里缓存或猜状态；错误路径统一 `sendJson(res, 500, { ok:false, error:"scan-failed", message: String(e && e.message || e) })`。
10. 宿主路由：`ctx.webServer.register({ path: "/api/<plugin-name>/scan", kind: "exact", handler })`，用 `ctx.effect(() => () => {})` 收口清理；路由集合按用途定（证据里是 `scan`(GET) / `publish`(POST) / `history`(GET)，守卫型是 `GET /api/patch-guard` + 两个 POST）。
11. 客户端 `lib/client.js` 骨架（槽位型）：
    ```js
    window.__ModuleLoader__.load({
      id: "<plugin-name>",
      factory: (require) => {
        // slots.inject / slots.register({ name: "sidebar.footer.action" }, ...)
      },
    });
    ```
12. 客户端页面型（在「技能管理」加页签）：`installMemorySourceUI(ctx, { sourceTypeId: "skills", pageId: "publish", order: 820, title: "发布", render })`，`render` 内复用 `MemorySourcePageFrame` 外壳；模块从 `require("dsh-mnemon/client")` 取 `{ installMemorySourceUI, MemorySourcePageFrame }`（此即 `dsh.client.external` 声明的作用）。不要改 `dsh-mnemon` 源码——新页面只在本插件里注册。
13. 数据全部走本插件宿主路由（例如页面只 fetch `/api/skill-publisher/scan`），不要在客户端直连本机脚本。

### C. 挂进 profile

14. 备份：`cp ~/.dsh/profiles/desktop/cordis.patch.yml ~/.dsh/profiles/desktop/cordis.patch.yml.bak-before-<tag>`，先 `tail -5` 确认现有条目形态。
15. 追加（文件是**顶层 YAML 数组**，不是 map；已有 `- insert:` 条目就再追加一条同级）：
    ```bash
    cd ~/.dsh/profiles/desktop && printf '\n# <中文用途说明>（<plugin-name>，非 npm 包）\n- insert:\n    - id: <short-id>\n      name: <plugin-name>\n' >> cordis.patch.yml && tail -4 cordis.patch.yml
    ```

### D. 三重机器可核验收（不启服务）

16. 语法门禁：`cd ~/.dsh/profiles/desktop/node_modules/<plugin-name> && node --check lib/index.js && node --check lib/client.js && echo "syntax ok"`。
17. 假 ctx 跑 `apply()` 收路由（**不要**真的启服务）：
    ```bash
    cd ~/.dsh/profiles/desktop/node_modules/<plugin-name> && node --input-type=module -e "
    import('./lib/index.js').then((m) => {
      const routes = [];
      const ctx = { webServer: { register: (r) => { routes.push(r.path + ' ' + r.kind); return () => {}; } }, effect: (fn) => { fn(); }, logger: { warn: () => {} } };
      m.apply(ctx);
      console.log('routes:', routes.join(' | '));
    });"
    ```
18. 实调一个 handler 端到端（构造最小 `req/res`，看 status 与 body 能否 `JSON.parse`）：
    ```bash
    cd ~/.dsh/profiles/desktop/node_modules/<plugin-name> && node --input-type=module -e "
    import('./lib/index.js').then(async (m) => {
      let handler = null;
      const ctx = { webServer: { register: (r) => { if (r.path.endsWith('/scan')) handler = r.handler; return () => {}; } }, effect: (f) => f() };
      m.apply(ctx);
      const res = { statusCode: 0, headers: {}, setHeader() {}, end(b) { this.body = b; } };
      const t = Date.now(); await handler({ method: 'GET', url: '/api/<plugin-name>/scan' }, res);
      console.log('status', res.statusCode, 'ms', Date.now() - t);
      console.log('ok', JSON.parse(res.body).ok);"
    ```
19. 注册确实读入：从 profile 目录用 `createRequire` 解析插件名 + 数 patch 条目（`node -e` 里不要用顶层 await，写成静态 import + `.then`）：
    ```bash
    cd ~/.dsh/profiles/desktop && node --input-type=module -e "
    import { createRequire } from 'node:module';
    import { readFileSync } from 'node:fs';
    const req = createRequire(process.cwd() + '/package.json');
    console.log('resolve:', req.resolve('<plugin-name>'));   // 应指向 .../lib/index.js
    const txt = readFileSync('cordis.patch.yml', 'utf-8');
    console.log('entries:', (txt.match(/^- /gm) || []).length, 'last:', JSON.stringify(txt.trim().split('\n').slice(-3)));"
    ```

## Pitfalls

- **`cordis.patch.yml` 是顶层数组**：追加 `- insert:` 同级条目，别写成 `insert:` 下的 map 子键，也别覆盖已有条目。改前必须备份（证据里的做法是 `.bak-before-<tag>`）。
- **不能靠 `dsh --dump-config` 验证**：证据中 `dsh --profile desktop --dump-config` 直接报 `profile "desktop" is managed exclusively by the Electron application`。验证改走 `createRequire` + patch 条目计数。
- **没有 PyYAML**：`import yaml` 会 `ModuleNotFoundError`。校验 YAML 只能用 node 侧的行匹配/正则计数，或先 `pip install pyyaml` 再说。
- **`node -e` 里顶层 await 会炸**（`error at evalModuleEntryPoint`）：用 `import('./lib/index.js').then(...)` 或静态 import + `.then`。
- **工具名写错**：证据里一次 `bash`（不存在）被拒，只有 `ops_bash`；Windows 下走 bash 语法时保持 `cd ~/.dsh/...` 这类 POSIX 路径，路径含空格要整体加引号（如 `"/d/deepseek hns/resources/runtime/cli/bin/dsh.cmd"`）。
- **别改 `dsh-mnemon` 源码加页签**：页面注册一律走本插件 `installMemorySourceUI` + `MemorySourcePageFrame`，`dsh-mnemon` 只作为 `require("dsh-mnemon/client")` 的 external 依赖。
- **客户端/宿主的生效口径不同**：客户端（`lib/client.js`）改完**刷新页面即生效**；宿主（`lib/index.js`，新增/修改路由、新增插件条目）**必须重启 DSH**。没重启前不要说 UI 已验证。
- **插件文件是宿主副本**：`~/.dsh/plugins/.../lib/client.js` 与源工程里的同名文件可能不同步（证据里该文件注释明确写着「改完源文件要同步过来」）。以 profile 下 `node_modules` 里实际被加载的那份为准。
- **本机补丁守卫**：手写了 `node_modules` 里的第三方文件后要跑 `python ...\_pub\patch_guard.py record`，否则下次 `status` 会把自改报成 drift。

## Verification

- `node --check` 对 `lib/index.js` 与 `lib/client.js` 均返回 0。
- 假 ctx 跑 `apply()` 打印出全部路由 `path` + `kind`，无异常、`logger.warn` 未触发。
- 实调一个 handler 得到 `status 200` 且 `JSON.parse(res.body)` 成功（证据基线：scan 796 ms 返回「可发 5 / 跳过 10 / 审查 0 / 包 29 KB」）。
- `createRequire(profile/package.json).resolve("<plugin-name>")` 解析到 `.../node_modules/<plugin-name>/lib/index.js`；patch 条目计数比改前 +1，末条就是新 insert。
- 收口时明确写：客户端改动刷新即生效，宿主改动需重启 DSH；未重启则不宣称 UI 已验证（证据里的唯一未验证项就是这个）。
