# Contributing

本仓库目前主要用于作者维护 ClinkZ-WoT 技术文章，也欢迎通过 Issue 或
Pull Request 指出事实错误、表达问题和可验证的反例。

## 内容类型

- `articles/`：已经进入正式写作流程的文章；
- `drafts/`：未完成的探索稿和素材；
- `assets/`：封面、图表和插图；
- `templates/`：写作模板；
- 根目录治理文档：AI 规则、事实策略、编辑规范和内容计划。

## 文件命名

文章路径：

```text
articles/<series>/<number>-<english-slug>.md
```

示例：

```text
articles/02-runtime-architecture/005-why-bindings-cannot-dispatch-handlers.md
```

文件名使用稳定的英文 slug，文章标题可以使用中文。

## 修改流程

1. 读取 `AGENTS.md` 和 `SOURCE_POLICY.md`；
2. 确认文章记录的主项目 commit；
3. 对技术结论进行证据检查；
4. 修改正文；
5. 更新 `updated` 日期；
6. 必要时更新修订说明；
7. 更新 `CONTENT_PLAN.md` 和 `PROJECT_STATE.md`。

## 错误报告

报告技术错误时，请尽量提供：

- 文章路径和段落；
- 错误陈述；
- ClinkZ-WoT 对应文件、源码、测试或 commit；
- 建议的更准确表述。

## 图片

- 技术图优先保存可编辑源文件；
- 导出图片放在 `assets/diagrams/` 或 `assets/images/`；
- 文件名使用英文；
- 图片必须与文章记录的架构基线一致；
- 不要在图片中把 planned 组件画成 implemented；
- 图片应包含 alt text 或正文说明。

## 发布同步

知乎发布后，回填：

```yaml
publication:
  platform: "zhihu"
  published_at: "YYYY-MM-DD"
  canonical_url: "https://zhuanlan.zhihu.com/..."
```

GitHub Markdown 是可版本化源稿，但知乎上的排版可能做适配性调整。技术结论
应保持一致。
