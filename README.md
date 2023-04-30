## 说明

语言：中文 | [English](https://github.com/deemoprobe/hugo-papermod2/blob/master/static/README_EN.md#description)

> 该主题根据Hugo PaperMod主题修改而来: <https://github.com/adityatelange/hugo-PaperMod>

中文版初稿见博主：[Sulv's Blog](https://www.sulvblog.cn)，本人在原博主GitHub项目基础上进行了个人定制和优化（UI基本没大改，主要是静态文件优化，删除了评论留言等功能，使之更适合纯文本技术博客）

### [博客DEMO](https://deemoprobe.github.io/)

## 本地运行

① 用`git clone`的方式拉取代码至桌面，此时会在桌面生成hugo-papermod2目录

② 进入到hugo-papermod2目录，输入`git submodule update --init`，表示拉取themes/下的hugo-PaperMod子模块，里面放的是官方主题，拉取来源是`.gitmodules`文件的链接，详细操作可参考：[Git子模块](https://git-scm.com/book/zh/v2/Git-%E5%B7%A5%E5%85%B7-%E5%AD%90%E6%A8%A1%E5%9D%97)

③ 把目录定位到hugo-papermod2下，在终端输入`hugo server -D`，在浏览器输入：localhost:1313 即可看到现成的博客模板。

## 修改优化

优化前可以通过：[PageSpeed Insights](https://pagespeed.web.dev/)测试一下自己站点的性能，根据分析结果进行优化。

![20230501032318](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20230501032318.png)

### 修改字体

本来为了网站的加载时间，是不打算定制的启用汉语字体的，但这款字体实在是太好看了，既美观又不花里胡哨，推荐：[LXGW WenKai / 霞鹜文楷](https://github.com/lxgw/LxgwWenKai)，将tff字体文件下载后放在`static/fonts`文件夹下，然后在文件`css/extended/font.css`中使用即可

![20230501031153](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20230501031153.png)

```css
/* 在文件css/extended/font.css中优化 */
font-display: swap; /* 在定制字体加载前启用备选字体，避免因字体未加载完全而白屏的情况 */

/* 备选字体在css/extended/blank.css文件指定 "PingFang SC", "Microsoft YaHei"为备选*/
font-family: LXGWWenKai-Regular, "PingFang SC", "Microsoft YaHei", sans-serif;
```

模板内部有许多个人信息需要自行修改，可以参考原博主的建站教程：[hugo博客搭建 | PaperMod主题](https://www.sulvblog.cn/posts/blog/build_hugo/)

根据网页性能，适当修改`layouts/partials/footer.html`、`assets\css\extended\fonts.css`、`assets\css\extended\blank.css`等文件中的css和js文件外链，选择合适的CDN外链。

![20230501033237](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20230501033237.png)

这是我优化后，除了字体文件基本在1s左右能完成博客的正常访问，由于github.io本来就慢，毕竟是白嫖，勉强可以接受，后续将持续优化。暂时没找到合适的CDN存放字体文件和GIF，主要是这俩首次加载比较慢。

### 配置静态资源缓存时间

## 发布博客文章

```bash
# 生成新文章，可指定content下任意路径，不指定则直接在content目录下生成
hugo new posts/tech/file.md
# Markdown渲染为HTML，--cleanDestinationDir参数含义是生成静态博客的时清除部分用不上的static内容
hugo -F --cleanDestinationDir

# 同步到远程GitHub Pages仓库
cd public
# 下面两步仅需在首次git初始化执行
# git init
# git remote add origin https://github.com/deemoprobe/deemoprobe.github.io.git
git add -A
git commit -m "modify"
git push -u origin master
```

## shortcodes使用方法

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

## 小表情和图标的使用

搜狗输入法"win+."快捷键可以调出输入法的表情，从中选择想要的即可使用🍕🍔🍟🌭🍿🚗🦽🚉，这些表情可以用在博客的任意位置：评论区、留言区、页面展示、菜单栏等等。

## 可能遇到的问题

1. 有些使用者会部署到github，可能遇到跨系统的问题，如提示`LF will be replaced by CRLF in ******`，这时输入命令：`git config core.autocrlf false`，解决换行符自动转换的问题。

2. 国内访问时`jquery.min.js`和`font-awesome.min.css`两个文件加载特别慢，这时候需要更换这俩文件的CDN链接，网上直接搜即可，我暂时使用的是`https://cdn.staticfile.org/jquery/3.6.3/jquery.min.js`和`https://cdn.staticfile.org/font-awesome/4.7.0/css/font-awesome.min.css`，不挂VPN情况下加载时长在300ms以内，勉强可以使用。
