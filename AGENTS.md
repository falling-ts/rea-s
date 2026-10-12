# rea-s 工作区说明（AGENTS.md）

本仓库 `rea-s` 是 **rea** 的上层工作空间，用于在本机组织 rea 相关的工作、
技能配置与参考资料。

## 目录结构

| 路径          | 用途                                               |
| ------------- | -------------------------------------------------- |
| `.agents/`    | 存放技能（skills）、代理配置等文件                 |
| `docs/`       | 工作区文档、笔记、设计说明                         |
| `refs/`       | 外部参考仓库（以 git submodule 形式引入）          |
| `refs/rea/`   | **rea** 项目本体（git submodule，见下方说明）      |

## rea 子模块

`refs/rea` 通过 git submodule 引入上游项目
[`morluto/rea`](https://github.com/morluto/rea.git)：

```bash
# 首次拉取后初始化子模块
git submodule update --init --recursive

# 更新 rea 到上游最新
git submodule update --remote --recursive
```

## 协作约定

- 修改文件请使用编辑工具，命令与脚本仅用于校验。
- 思考与输出使用中文；代码、标识符与符号保持原样。
- 校验脚本优先内联执行，避免遗留临时文件。
