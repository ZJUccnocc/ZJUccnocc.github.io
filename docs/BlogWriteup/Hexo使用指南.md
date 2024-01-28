# Hexo使用指南

## 1 配置相关

具体配置的意义可以参考官方文档，写的还是很清楚的。其中主要要修改的部分除了 `theme` `title` `auther` 等以外，基本上没啥需要修改的，保留默认配置即可。

需要了解的是，在一个 `hexo` 框架的静态网页目录中，可能会存在着三种类型的配置文件（config文件），一个是在根目录下的 `_config.yml` 一个是在根目录下的需要使用者自己新建的文件 `_config.[theme_name].yml` ，还有一个是在 `./theme/[theme_name]/` 目录下的 `_config.yml` 文件。大致布局如下：

```
root
|---theme
|	|---theme_name
|		|---_config.yml
|---_config.yml
|---_config.theme_name.yml
```

因为在这三个文件中都可以对主题进行配置，所以我们需要了解他们的优先级：

1. `_config.yml` ：在这个文件中配置主题的优先级最高，配置方法是增加一个叫 `theme_config` 的关键字。
2. `_config.theme_name.yml` ：在这个文件中配置主题的优先级第二，可以直接在本文件下进行配置。
3. `./theme/theme_name/_config.yml` ：在这个文件下配置的优先级最弱，但是因为这个是安装hexo主题时就已经设置好的部分，所以为了避免对原主题的更改，以及方便区分和对校自己写的代码和原主题提供的代码，我们一般会选择用第二种方法来进行主题配置。

## 2 命令

### 2.1 建站前

#### init

```shell
hexo init [folder]
```

新建一个网站。如果没有设置 `folder` ，Hexo 默认在目前的文件夹建立网站。

#### new

```shell
hexo new [layout] <title>
```

新建一篇文章。如果没有设置 `layout` 的话，默认使用 `_config.yml` 中的 `default_layout` 参数代替。如果标题包含空格的话，请使用**引号**括起来。

| 参数           | 描述                                          |
| -------------- | --------------------------------------------- |
| -p / --path    | 自定义新文章的路径                            |
| -r / --replace | 如果存在同名文章则替换他                      |
| -s / --slug    | 文章的 Slug，作为新文章的文件名和发布后的 URL |

默认情况下，Hexo 会使用文章的标题来决定文章文件的路径。对于独立页面来说，Hexo 会创建一个以标题为名字的目录，并在目录中放置一个 `index.md` 文件。你可以使用 `--path` 参数来覆盖上述行为、自行决定文件的目录：

```shell
hexo new page --path about/me "About me"
```

以上命令会创建一个 `source/about/me.md` 文件，同时 Front Matter 中的 title 为 `"About me"`

#### generate

```shell
hexo generate  #hexo g
```

生成静态文件。

#### server

```shell
hexo server		#hexo s
```

启动服务器。默认情况下，访问网址为： `http://localhost:4000/`。

| 参数          | 描述                           |
| ------------- | ------------------------------ |
| -p / --port   | 重设端口                       |
| -l / --log    | 启动日记记录，使用覆盖记录格式 |
| -s / --static | 只使用静态文件                 |

#### deploy

```shell
hexo deploy		#hexo d
```

部署网站。

#### clean

```shell
hexo clean		#hexo cl
```

清除缓存文件 (`db.json`) 和已生成的静态文件 (`public`)。

在某些情况（尤其是更换主题后），如果发现您对站点的更改无论如何也不生效，您可能需要运行该命令。

## 3 基本操作

