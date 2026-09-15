# lorelum-rehearsal — 个人 Practice 沉淀仓库

原为 react-web-craft 0.1.0 的安装路径演练库（该演练内容已归档至 `archive/react-web-craft-rehearsal`，正式版在 [lorelum/lorelum-packs](https://github.com/lorelum/lorelum-packs)）。本仓库现为 AzMilabo 的个人 Practice 积累池：随手记录真实工程决策，经使用验证后，待主库晋升机制就绪时从中筛选提交上游。

## 目录约定

- `inbox/` — 低门槛草稿：任务中的决策随时记「决策 → 做法 → 结果」，不要求格式合规。
- `packs/<pack>/` — 正式内容：`pack.yaml` + `README.md` + `practices/**/*.md`，必须通过 `lore format` / `lore validate`（0 诊断）。
- `templates/` — 新建 pack / practice 的拷贝起点（字段以 format schema 为准）。

## 工作流（每周一个节拍）

1. 随手捕获 → `inbox/`。
2. 批量晋升：inbox → `packs/<pack>/practices/`，跑 `lore format` + `lore validate`，Practice ID 与 anti-pattern ID 查重，隔离 store 安装后做检索自测（每篇 2-3 个真实查询，近邻干扰能分对）。
3. 发版：打不可变 tag `<pack>-v<x.y.z>`，并在 `.lorelum/registry.yaml` 登记 release。**tag 绝不重指，改动一律发新版本。**
4. 安装回路验证：

   ```sh
   lore pack install <pack>@<ver> --registry AzMilabo/lorelum-rehearsal --store-root <隔离目录>
   lore pack list --details
   lore query "<当时真实会说的话>"
   ```

## 写作规则

- 一篇 Practice 只拥有一个决策，写明例外与边界。
- canonical 用英文；`stage` / `tech_stack` 为自由字符串（用于检索过滤），`severity` 取 `info | warn | critical`（默认 warn）。
- 每篇必须有 **Evidence** 小节：PR / 事故 / 决策记录链接 + 实际结果 —— 这是日后主库筛选的依据。
- 写之前先查官方 catalog 查重；对官方既有 Practice 的改进想法单独标注 `relates-to: <pack>.<id>`，晋升时走修订通道而非新增。
- License 对齐主库：CC-BY-4.0。

## 状态标记

在各自 pack 的 README 中维护每篇状态：`draft → validated → offered`。只把 `validated` 及以上的内容对外提供。
