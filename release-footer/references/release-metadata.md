# 发布元数据与缓存

## 数据约定

推荐在静态目录生成 release.json（如 public/release.json）：

```json
{
  "version": "1.2.3",
  "commit": "abc1234",
  "commitFull": "完整的真实 Git SHA"
}
```

此处仅是结构示例，不是可直接发布的数据。版本取自项目权威版本来源，例如 package.json。UI 回退版本也应从该来源生成或导入，避免版本常量多处手工更新。

若项目明确配置发布阶段，可在元数据中添加可选 `stage`（如 `beta`、`lts`）和 `stageLabel`（如 `RC.2`）；没有显式配置时由版本后缀推断，规则见 [release-stages.md](release-stages.md)。阶段与版本、SHA 应来自同一份发布配置和源码快照。

元数据缺失时显示 local 或“开发版”，不能假装展示有效提交。校验 version 和 commit 的类型、格式；版本校验应支持项目实际使用的预发布语义版本。可验证返回版本与预期版本一致，发现不同则保留回退状态，避免旧元数据覆盖当前版本。

## 浏览器加载

读取静态元数据时，请求 URL 使用当前构建的版本，而不是先读取元数据来决定版本：

```js
fetch(`/release.json?v=${encodeURIComponent(appVersion)}`, {
  cache: 'no-store',
  signal: controller.signal,
});
```

appVersion 来自构建时的版本来源。React effect 在卸载时 abort；请求失败时保留明确的回退显示。适配现有数据加载机制，不要求每个框架都用客户端 fetch。

曾遇到的现象是健康检查已返回新版本，但固定 /release.json 地址仍返回旧版本；加查询参数后返回新值。这能证明按 URL 区分的缓存参与其中，不能仅凭 Server: nginx 确定是哪一层缓存。

前端 no-store 不能保证中间代理一定绕过缓存。版本化 URL 要求代理缓存键包含查询参数；若不包含，使用带版本的文件路径，例如 /releases/1.2.3.json，或在有权限时调整缓存策略。服务器可为元数据响应设置 Cache-Control: no-store。不要把多个矛盾的 Cache-Control 值叠加当成可靠修复。已有页面和 JS 自身的缓存也需要独立判断。

## 打包与 Docker

- 打包前检查相关源码无未提交或遗漏的新文件；不要忽略未跟踪源码而声称发布已包含它。
- 固定完整 HEAD SHA，从该 SHA 读取版本、导出源码和生成短 SHA，避免运行期间 HEAD 变化导致不一致。
- 使用临时目录和 git archive；在导出的静态目录写入 release.json。元数据不依赖服务器具有 .git。
- 保留项目必需隐藏配置，排除凭据、环境变量文件、node_modules、旧发布包与构建缓存。
- 发布包与校验文件放入被忽略的 releases/ 或项目约定目录。文件名采用项目名称与版本。避免覆盖既有包。
- 注入元数据后再完成最终构建验证；确认框架复制静态文件进入 standalone 或其他生产输出。
- 检查 .dockerignore 不会排除 release.json，Docker COPY 包含生成元数据。仅 git clone/pull 不会获得这个被忽略文件：CI 或直接从 Git 部署时必须先从 CI 提交 SHA 生成它再构建。
- 临时目录清理必须针对明确创建的目录；脚本不得重置或清理用户工作区。
- 脚本失败时停止，不推送未经验证的产物。使用临时输出并在校验成功后移入发布目录，可避免半成品占用版本名。

## 故障定位

依次比对：压缩包中 release.json → 容器内部文件 → 健康检查 → 元数据 URL（普通地址和带版本地址）→ 页面显示。
用实际结果区分旧构建目录、旧容器、静态文件遗漏、浏览器缓存和中间代理缓存。不要一开始就要求 --no-cache 全量 Docker 重建。
