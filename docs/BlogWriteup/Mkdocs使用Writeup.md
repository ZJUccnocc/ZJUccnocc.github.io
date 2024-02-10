# Mkdocs+Github.io配置Writeup

## 写在前面

本人的mkdocs文档仓库就在https://github.com/ZJUccnocc/ZJUccnocc.github.io

如果喜欢本人的配置文件可以直接去这里copy（这样就可以跳过下文的很多配置内容，但如果copy请务必标明复制来源！谢谢~）。

另外如果第七步的github Pages配置还有问题欢迎私信我~

联系方式见：https://zjuccnocc.github.io/Home/about/

## 第一步 安装Mkdocs

```shell
pip install mkdocs
```

- 需要安装好pip的功能，具体安装可以看[pip官网教程](https://pip-cn.readthedocs.io/en/latest/installing.html)

看到这样的Message表示安装成功：

```shell
Successfully installed click-8.1.7 colorama-0.4.6 ghp-import-2.1.0 jinja2-3.1.3 markdown-3.5.2 markupsafe-2.1.3 mergedeep-1.3.4 mkdocs-1.5.3 pathspec-0.12.1 platformdirs-4.1.0 pyyaml-6.0.1 pyyaml-env-tag-0.1 watchdog-3.0.0
```

> **重要备注**
>
> 如果你的电脑上还没有一个好用的Markdown编辑器的话，建议先安装一个，我本人使用的是Typora，还蛮好用的，就是要一次性付费（哭），如果你不想付费的话，也可以用VSC的markdown插件，就是需要你熟记各种样式的markdown代码罢了（对刚入门的使用者不太友好，但可以锻炼你徒手搓md的技能）

## 第二步 创建项目

恭喜你迈出了制作blog的第一步，下面我们来创建一个简单的项目。

### 创建项目

在Windows的命令行运行以下命令：

```shell
mkdocs new <my-project> #
cd <my-project>
```

- 将 `<my-project>` 替换为你希望你的项目被命名的名字，其实无伤大雅，主要就是方便输入就行，别起个长的要死的名字，每次进入都要敲半天。
- 注意你的执行目录最好选在一个好找的目录，比如Desktop，这样你的项目文件夹就会出现在执行目录下。

如果顺利的话你的执行目录下会出现一个这样的文件夹：

```
.
├─mkdocs.yml
└─docs
  └─index.md
```

- `mkdocs.yml` 是一个配置文件
- `docs` 文件夹是用来保存你的源文件的地方。

- 在刚安装完之后你的 `docs` 下只有一个文件叫 `index.md` ，这个文件里面记载了一些常用命令。

### 部署本地与预览Blog

接下来我们可以预览一下我们的Blog网页。但请注意，这一步需要在 `<my-project>` 目录下执行：

```
mkdocs serve
```

如果出现Message：

```
INFO    -  Building documentation...
INFO    -  Cleaning site directory
INFO    -  Documentation built in 0.41 seconds
INFO    -  [11:50:48] Watching paths for changes: 'docs', 'mkdocs.yml'
INFO    -  [11:50:48] Serving on http://127.0.0.1:8000/
```

表示你的服务器已经成功启动，并且在本地部署，接下来我们可以到 `http://127.0.0.1:8000/` 下预览我们的页面。可以看到我们的页面已经被成功渲染出来了。

聪明的你应该可以发现这个其实就是你的 `index.md` 里面的内容。因此我们可以知道 `doc` 目录下的文件最终会被 `mkdocs server` 指令渲染成一个部署在本地 `8000端口` 的页面。

另外，有个方便的技术支持，就是当你编辑了配置文件（ `mkdocs.yml` )，文档目录( `docs` )或主题目录( 后面接入主题之后会提到 )后，都会自动重新生成网页。

我们可以尝试更改 `<my-project>/docs/index.md` ，可以看到我们的网页也会随之更改。

### 更改Blog名字与域名

接下来我们可以尝试更改 `<my-project>/mkdocs.yml` 文件：

```yaml
site_name: CCnocc's Blog
```

需要特别说明的是，有一个配置属性 `site_url` 一开始不会出现在配置文件中，但是会默认为下方的属性。这个 `site_url` 需要在你部署到服务器上前更改为你的服务器域名，在你部署前可以设置成如下内容，后续部署到服务器的时候也会有更多说明。

```yaml
site_url: https://localhost:8000/
```

在这一步中我们创建了一个可以部署在本地的项目，如果想让别人能够访问你的Blog，你还需要将你的内容部署到服务器上。如果你还想要一个好看的域名，你需要去买一个并做好备案工作。

## 第三步 添加页面

### 增加一个md文件

我们接下来尝试再添加一个页面到我们的blog中。

首先我们从官方文档下尝试拷贝一份About说明，同样我们的执行目录应该在 `<my-project>` 下：

```
curl 'https://jaspervdj.be/lorem-markdownum/markdown.txt' > docs/about.md
```

- 如果遇到问题了你也可以自己写一个 `about.md` 文档，用于存放一些说明。

### 配置 `nav` 属性

然后我们在 `mkdocs.yml` 文件中添加一个 `nav` 属性：

```yaml
......
nav:
  - Home: index.md
  - About: about.md
```

这样你就可以在网页上看到你的About栏目和你的Home栏目了。并且还会再栏目条的最右边找到 `Previous` 和 `Next` 两个按钮，用于在不同导航栏之间切换。

最后，值得一提的是Mkdocs还自带了搜索条功能，也在栏目条的右侧，大家可以自行尝试使用一下~

## 第四步 配置主题

如果希望使用别人做好的开源的主题的话，我们需要将主题文件下载到工程文件中并且做好配置。

### 尝试主题 `readthedocs` 

同样我们先编辑 `<my-project>/mkdocs.yml` ，添加一个 `theme` 属性，并赋值为 `readthedocs`

```yaml
......
theme: readthedocs
```

保存更改后我们即可在本地部署的页面上看到主题的更换，聪明的你一定已经发现了所谓"主题"，其实就是别人配置好的一系列渲染规则。使用不同的主题可以将同一份md文件渲染成不同的样式。

如果想要更多主题可以去网上找找，或者直接找认识的使用mkdocs框架的佬问问能不能直接copy一份主题配置文件，一般会有github上开源的主题配置文件供使用。

### 改变页面图标

默认情况下， `MkDocs` 使用 `MkDoc收藏夹图标` 。要使用不同的图标，请在 `docs` 目录下创建 `img` 文件夹，并将自定义的 `favicon.ico` 复制到该目录中。 `MkDocs` 将自动检测并使用该文件作为您的收藏夹图标。如果自动检测失效的话，我们可以编辑 `mkdocs.yml` 文档，增加一个 `site_favicon` 属性。

```yaml
site_favicon: docs/img/favicon.ico
```

### 最美主题 material

在我们的 `<my-project>` 目录下执行命令：

```shell
pip install mkdocs-material
```

安装完毕后我们同样编辑 `<my-project>/mkdocs.yml` 的 `theme` 属性，赋值为 `material` 后即可看到我们的页面主题发生了更改。这也是我们大多数的实验指导所使用的主题。

#### 主题配置

这个部分应该是花费时间和心思最多的部分，因为Material主题提供了非常多的配置选项，我们可以根据自己的需求进行配置。具体每一条内容是什么意思可以看注释，没注释清楚的我也不清楚了（[内容来源](https://zhuanlan.zhihu.com/p/672743170)，有部分更改）。

```yaml
theme:
  custom_dir: material/overrides # 自定义文件夹，对于个别页面，如果你不想使用主题的默认样式，可以在这里进行修改，使用里面的文件覆盖主题的默认文件。具体可以参考material官方文档。
  name: material # 主题名称，Material已经是最优秀的选择了，相信我。
  logo: static/images/logo.png # logo 图片
  language: Chinese # 默认语言
  features: # 功能
    - announce.dismiss # 可以叉掉公告的功能
    # - content.action.edit # 编辑按钮，会跳转到你的仓库里编辑当前页面内容
    # - content.action.view # 查看按钮，会跳转到你的仓库里查看当前页面内容
    - content.code.annotate # 代码注释，具体不清楚
    - content.code.copy # 复制代码按钮
    # - content.code.select # 选择代码按钮
    - content.tabs.link # 链接标签
    - content.tooltips # 不太清楚呢这个
    # - header.autohide # 自动隐藏header
    - navigation.expand # 默认展开导航栏
    - navigation.footer # 底部导航栏
    - navigation.indexes # 索引按钮可以直接触发文件，而不是只能点击其下属选项浏览，这个功能可以给对应的section提供很好的预览和导航功能
    # - navigation.instant # 瞬间加载 最好注释掉，多语言切换这个会导致跳回首页
    - navigation.instant.prefetch # 预加载
    - navigation.instant.progress # 进度条
    - navigation.path # 导航路径， 目前好像没啥用
    # - navigation.prune # 只构建可见的页面
    - navigation.sections # 导航栏的section
    - navigation.tabs # 顶级索引被作为tab
    - navigation.tabs.sticky # tab始终可见
    - navigation.top # 开启顶部导航栏
    - navigation.tracking # 导航栏跟踪
    - search.highlight # 搜索高亮
    - search.share # 搜索分享
    - search.suggest # 搜索建议
    - toc.follow # 目录跟踪-页面右侧的小目录
    # - toc.integrate # 目录跟踪集成到左侧大目录中
  palette:
    - media: "(prefers-color-scheme)" # 主题颜色
      scheme: slate
      primary: black
      accent: indigo
      toggle:
        icon: material/link
        name: Switch to light mode
    - media: "(prefers-color-scheme: light)" # 浅色
      scheme: default
      primary: indigo
      accent: indigo
      toggle:
        icon: material/toggle-switch
        name: Switch to dark mode
    - media: "(prefers-color-scheme: dark)" # 深色
      scheme: slate
      primary: black
      accent: indigo
      toggle:
        icon: material/toggle-switch-off
        name: Switch to system preference
  font: # 字体，大概率不需要换
    text: Roboto
    code: Roboto Mono
  favicon: docs/img/favicon.ico # 网站图标，因为你使用了主题会覆盖之前设置的site_favicon属性，所以在这里最好重新设置一下
  icon: # 一些用到的icon
    logo: logo
    previous: fontawesome/solid/angle-left
    next: fontawesome/solid/angle-right
    tag:
      default-tag: fontawesome/solid/tag
      hardware-tag: fontawesome/solid/microchip
      software-tag: fontawesome/solid/laptop-code
```

## 第五步 其他配置

### Markdown拓展

在 `mkdocs.yml` 内增加如下内容：

```yaml
......
markdown_extensions:
  - toc:
      permalink: true
      toc_depth: 4
  - meta
  - def_list
  - attr_list
  - md_in_html
  - sane_lists
  - admonition
  - pymdownx.keys
  - pymdownx.mark
  - pymdownx.tilde
  - pymdownx.critic
  - pymdownx.details
  - pymdownx.snippets
  - pymdownx.magiclink
  - pymdownx.superfences
  - pymdownx.inlinehilite
  - pymdownx.emoji:
      emoji_index: !!python/name:materialx.emoji.twemoji
      emoji_generator: !!python/name:materialx.emoji.to_svg
  - pymdownx.tabbed:
      alternate_style: true 
  - pymdownx.tasklist:
      custom_checkbox: true
  - pymdownx.highlight:
      anchor_linenums: true
  - pymdownx.arithmatex:
      generic: true
```

然后你就会发现你的代码能够根据你指定的编程语言高亮显示了。

除此之外你还可以设置你的代码段的标题，具体操作如下:

````
```python title="PythonTest.py"	//这一行是配置代码段属性用的，第一个参数指定代码段的编程语言
								//第二个属性就是代码段的标题
print("HelloWorld!")
```
````

实现效果如下：

```python title="PythonTest.py"
print("HelloWorld!")
```

### 基础配置

在这部分我可能会重复之前的配置，但是为了完整性，请自行确认是否需要添加这一条配置内容。

```yaml
# 项目信息
site_name: CCnocc's Blog 	# 项目名称
site_url: https://localhost:8000/ 
site_author: CCnocc 		# 作者
# 代码仓库信息
repo_name: ZJUCCnoccCode # 仓库名称
repo_url: https://github.com/ZJUCCnocc/BlogCode.git/ # 仓库地址
```

## 第六步 搭建网页

当你准备好将你本地部署的页面发布到网上后，你需要先按照如下步骤创建你的 `build文件` ，同样这些命令是在 `<my-project>` 目录下执行的：

```
mkdocs build
```

如果出现如下Message，说明你的build成功完成了：

```
INFO    -  Cleaning site directory
INFO    -  Building documentation to directory: D:\Desktop\my-project\site
INFO    -  Documentation built in 0.09 seconds
```

完成后你会在你的 `<my-project>` 目录下看到一个新的名为 `site` 的文件夹，这里面就是你的网页文件。

可以在 `site` 文件夹中看到你原先的md文件被**编译**成了 `index.html` 和 `about/index.html`。所以我们可以将`mkdocs build` 看作是一个编译命令，将我们的 `md文件` 渲染成 `html网页文件` 。

## 第七步 Github Pages部署

参考文献：[利用mkdocs部署静态网页至GitHub pages （相当于做一个个人网站）_mkdocs github pages-CSDN博客](https://blog.csdn.net/m0_63203517/article/details/127019819)

参考文献中存在写的不清楚的地方，我争取在我的Write up中补充完整。

### 创建仓库

在你的Github上创建一个名为 `<你的用户名>.github.io` 的仓库，比如我的仓库就是这样的：![image-20240206134714479](./Mkdocs%E4%BD%BF%E7%94%A8Writeupimg/image-20240206134714479.png)

### 克隆仓库到本地

```
git clone https://github.com/<你的用户名>/<你的用户名>.github.io.git
```

注意克隆的位置最好是你熟悉的位置。

### 复制内容

将你之前建好的网页文件夹复制到当前文件夹内（如果你按我的方法来，那么这个文件夹的名字应该就是`<你的用户名>.github.io`）。大概是如下的情况。补充一句，之前第六步的site其实只是为了你本地部署使用的文件，在上传到github pages远程部署的时候并不需要上传，因此这里并不需要复制site文件夹。

![image-20240206135242966](./Mkdocs%E4%BD%BF%E7%94%A8Writeupimg/image-20240206135242966.png)

### 编写Github Workflow

在 `<你的用户名>.github.io` 文件夹下执行：

```shell
$ mkdir .github
$ cd .github
$ mkdir workflows
$ cd workflows
```

在workflows文件夹中新建一个文件： `PublishMySite.yml` 

文件内容如下：

```yaml
name: publish site
on: # 在什么时候触发工作流
  push: # 在从本地main分支被push到GitHub仓库时
    branches:
      - main
  pull_request: # 在main分支合并别人提的pr时
    branches:
      - main
jobs: # 工作流的具体内容
  deploy:
    runs-on: ubuntu-latest # 创建一个新的云端虚拟机 使用最新Ubuntu系统
    steps:
      - uses: actions/checkout@v2 # 先checkout到main分支
      - uses: actions/setup-python@v2 # 再安装Python3和相关环境
        with:
          python-version: 3.x
      - run: pip install mkdocs-material # 使用pip包管理工具安装mkdocs-material
      - run: mkdocs gh-deploy --force # 使用mkdocs-material部署gh-pages分支
```

### 设置github pages

回到github pages。

![image-20240206140103666](./Mkdocs%E4%BD%BF%E7%94%A8Writeupimg/image-20240206140103666.png)

打开workflow的写权限![image-20240206140215146](./Mkdocs%E4%BD%BF%E7%94%A8Writeupimg/image-20240206140215146.png)

![image-20240206140344844](./Mkdocs%E4%BD%BF%E7%94%A8Writeupimg/image-20240206140344844.png)

除了上述这些设置，还可以看看这个界面有没有其他设置和本人的不一样，后续如果出现奇怪的bug可以考虑调整成本人的试试（）

接下来我们可以把自己写好的内容上传到仓库里了。

在 `<你的用户名>.github.io` 文件夹下执行：

```shell
git add .
git commit -m "first commit"
git push
```

现在你的内容就被上传到你的仓库里了。最后再设置一下你部署的文件夹就行啦~

![image-20240206140532795](./Mkdocs%E4%BD%BF%E7%94%A8Writeupimg/image-20240206140532795.png)

不出意外的话你的网页可以在如下图的网址访问啦~

![image-20240206140623346](./Mkdocs%E4%BD%BF%E7%94%A8Writeupimg/image-20240206140623346.png)

## 写在后面

最后，如果还是有出现问题，一般都是github pages部署的问题，欢迎联系本人debug~

联系方式见：https://zjuccnocc.github.io/Home/about/
