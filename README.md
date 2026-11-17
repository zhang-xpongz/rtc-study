# rtc-study

WebRTC / 音视频 / RTC 专项复习与长期学习仓库。

- 站点：https://zhang-xpongz.github.io/rtc-study/
- 内容：复习计划、每日记录、源码卡片、基础知识、系统实践、算法题解
- 工具：MkDocs Material + GitHub Pages

## 1. 仓库模型

**一个仓库，按内容类型分目录，不按日期堆文件。**

```
rtc-study/
├── README.md
├── mkdocs.yml
├── requirements.txt
├── .github/workflows/pages.yml
└── docs/
    ├── index.md                 # 首页
    ├── plan/                    # 计划：3 周、逐日、打卡
    ├── daily/                   # 每日记录：YYYY-MM/YYYY-MM-DD.md
    ├── source/                  # 源码走读：地图、卡片、单测
    ├── fundamentals/            # 基础：音频、视频、封装、编解码
    ├── systems/                 # 系统：编译、调试、SRS、mediasoup
    ├── algorithms/              # 算法题解
    ├── assets/                  # 图片、图表
    ├── tpl/                    # 模板
    └── 写作约定.md
```

约定：

- `main` 只放 Markdown 和站点配置，不放构建产物。
- `gh-pages` 由 GitHub Actions 自动生成，不手动提交。
- 每日记录一页一个文件；源码一模块一张卡片；算法一题一个文件。
- 文件名保留中文，标题清晰优先；目录名用英文，便于脚本和 URL。

## 2. 本地预览

```bash
cd rtc-study
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
mkdocs serve
```

打开 http://127.0.0.1:8000/。

构建静态站点：

```bash
mkdocs build
# 产物在 site/
```

## 3. GitHub Pages 首次配置

1. 在 GitHub 创建**空仓库** `rtc-study`（不要勾 README / .gitignore）。
2. 推送：

```bash
git remote add origin git@github.com:zhang-xpongz/rtc-study.git
git push -u origin main
```

3. 打开仓库 **Settings → Pages**：
   - Source 选 **Deploy from a branch**
   - Branch 选 **gh-pages** / **root**
4. 等 Actions 跑完，访问：

```
https://zhang-xpongz.github.io/rtc-study/
```

之后每次 push 到 `main`，Actions 会自动 `mkdocs gh-deploy` 更新 `gh-pages`。

## 4. 每日更新流程

```bash
cd rtc-study
git pull

# 1) 新增当天记录
cp docs/templates/每日记录.md docs/daily/2026-11/2026-11-18.md
# 编辑内容

# 2) 如果读了源码 / 做了算法，也一起加
# docs/source/xxx.md
# docs/algorithms/xxx.md

# 3) 提交
git add docs
git commit -m "docs(daily): 2026-11-18 GCC 与弱网"
git push
```

提交信息建议：

| 类型 | 示例 |
|---|---|
| 每日记录 | `docs(daily): 2026-11-18 GCC 与弱网` |
| 源码卡片 | `docs(source): add VideoStreamBufferController card` |
| 算法题 | `docs(algo): add LeetCode 209/56` |
| 计划调整 | `docs(plan): adjust D5 schedule` |
| 站点配置 | `chore(site): enable mermaid` |

## 5. 隐私边界

**这个仓库是公开的，只放技术学习内容。**

以下内容不要提交，放在私有仓库或本地 `private/`（已在 `.gitignore` 中忽略）：

- 简历、薪资期望、offer 对比
- 投递公司清单、面试反馈
- 个人联系方式、证件信息

## 6. 长期扩展

- 每周日把 `daily/` 里的高频问题提炼到 `fundamentals/` 或 `source/`。
- 每完成一个模块，在 `source/` 加一张卡片，并在首页更新进度。
- 面试前用站内搜索复习，不重新翻聊天记录。
