# GitHub Pages 发布说明

这个目录已经是一个最小可发布版本，核心文件如下：

- `index.html`
- `.nojekyll`
- `tactile_force_datasets_survey_zh.md`

## 最简单的发布方式

1. 新建一个 GitHub 仓库。
2. 把这个目录中的所有文件上传到仓库根目录。
3. 在 GitHub 仓库中打开 `Settings` -> `Pages`。
4. 在 `Build and deployment` 里选择：
   - `Source`: `Deploy from a branch`
   - `Branch`: `main`
   - `Folder`: `/ (root)`
5. 保存后，等待 GitHub Pages 部署完成。

## 发布后的链接形式

通常会是：

`https://<你的用户名>.github.io/<仓库名>/`

## 说明

- 页面入口文件是 `index.html`。
- `.nojekyll` 用于避免 GitHub Pages 对静态文件做额外处理。
- 如果你已经有一个仓库，也可以把这几个文件放进 `docs/` 目录，然后在 Pages 里把发布目录改成 `docs`。
