# rea-s

rea 的上层工作空间，用于组织 rea 相关的工作、技能配置与参考资料。

## 结构

- `.agents/`：技能与代理配置文件
- `docs/`：文档与笔记
- `refs/`：外部参考仓库
  - `refs/rea/`：上游 [morluto/rea](https://github.com/morluto/rea.git) 子模块

## 快速开始

```bash
# 克隆本仓库（含子模块）
git clone --recurse-submodules git@github.com:falling-ts/rea-s.git

# 或克隆后初始化子模块
git submodule update --init --recursive
```

## 更新 rea 子模块

```bash
git submodule update --remote --recursive
git add refs/rea
git commit -m "chore: 更新 rea 子模块"
```

## 说明

- 远程仓库：`git@github.com:falling-ts/rea-s.git`
- rea 通过 submodule 形式引用，不修改其内容（如需改动请在上游进行或另建分支）。
