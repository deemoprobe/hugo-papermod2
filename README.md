语言：中文 | [English](https://github.com/deemoprobe/hugo-papermod2/blob/main/static/README_EN.md)

> 该主题根据Hugo PaperMod主题修改而来: <https://github.com/adityatelange/hugo-PaperMod>

### [博客DEMO](https://deemoprobe.github.io/)

## 1. git clone 拉取代码

① 用`git clone`的方式拉取代码至桌面，此时会在桌面生成hugo-papermod2目录

② 进入到hugo-papermod2目录，输入`git submodule update --init`，表示拉取themes/hugo-PaperMod/下的子模块，里面放的是官方主题

## 2. 启动界面

③ 把目录定位到hugo-papermod2下，在终端输入`hugo server -D`，在浏览器输入：localhost:1313 即可看到现成的博客模板。

## 3. 修改信息

模板内部有许多个人信息需要自己配置，请耐心修改完，可以参考博主的建站教程：[https://YOUR_DOMAIN/posts/blog/](https://YOUR_DOMAIN/posts/blog/)

## 4. Hugo博客交流群

🎉🎉 YOUR_QQ 🎉🎉

## 5. shortcodes使用方法

`bilibili: {{< bilibili BV1Fh411e7ZH(填 bvid) >}}`

`youtube: {{< youtube w7Ft2ymGmfc >}}`

`ppt: {{< ppt src="网址" >}}`

`douban: {{< douban src="网址" >}}`

```bash
# 文章内链卡片
# 末尾要加 md，只能填写相对路径，如下
{{< innerlink src="posts/tech/mysql_1.md" >}}
```

```bash
gallery:

{{< gallery >}}
{{< figure src="https://YOUR_DOMAIN/posts/read/structural_thinking/0.png" >}}
{{< figure src="https://YOUR_DOMAIN/posts/read/structural_thinking/0.png" >}}
{{< /gallery >}}
```

## 5. 可能遇到的问题

1. 有些使用者会部署到github，可能遇到跨系统的问题，如提示`LF will be replaced by CRLF in ******`，这时输入命令：`git config core.autocrlf false`，解决换行符自动转换的问题。
