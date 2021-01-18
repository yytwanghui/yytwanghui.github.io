---
title: hexo insert images
date: 2019-06-01 15:13:14
categories: 
- Hexo博客
tags: 
- hexo
- 生成
- 本地图片
---

<center><font size=4 color="red">本文是一篇关于Hexo如何生成本地图片的教程</font></center>

<!--more-->

# hexo生成博文插入本地图片

## 步骤

把主页配置文件_config.yml 里的post_asset_folder:这个选项设置为true

在你的hexo目录下执行这样一句话npm install hexo-asset-image --save，这是下载安装一个可以上传本地图片的插件，来自dalao：dalao的git

等待一小段时间后，再运行hexo new "xxxx"来生成md博文时，/source/_posts文件夹内除了xxxx.md文件还有一个同名的文件夹 （当然也可以自己手动建）

最后在xxxx.md中想引入图片时，先把图片复制到xxxx这个文件夹中，然后只需要在xxxx.md中按照markdown的格式引入图片：

`![你想输入的替代文字](xxxx/图片名.jpg)`

> 注意： xxxx是这个md文件的名字，也是同名文件夹的名字。只需要有文件夹名字即可，不需要有什么绝对路径。你想引入的图片就只需要放入xxxx这个文件夹内就好了，很像引用相对路径。

最后检查一下，hexo g生成页面后，进入public\2017\02\26\index.html文件中查看相关字段，可以发现，html标签内的语句是`<img src="2017/02/26/xxxx/图片名.jpg">`，而不是`<img src="xxxx/图片名.jpg>`。这很重要，关乎你的网页是否可以真正加载你想插入的图片。

## 遇到问题

遇到没能成功显示的问题，查看网页，结果是 `<img src="2017/02/26/2017-02-23-c/图片名.jpg">` 的形式，然后查看 hexo 的目录，发现路径是 /public/2017/02/26/c/a.jpg。

之前在 md 文件中引用时写作：

`![你想输入的替代文字](xxxx/图片名.jpg)`

改为：

`![](a.jpg)`

就可以了。

效果查看 这篇文章 的图片显示。

## 为什么

看上面的效果，出错原因是 /public/2017/02/26/c/ 文件夹中没有生成保存图片的同名文件夹，图片直接被拷贝到这个文件夹下了。

突然想起从 jekyll 转移到 hexo 的时候更改过文章路径与标题解析方式…

不知道有没有关联。

## hexo 修改文章路径

产生的链接和两条属性相关。

permalink: :year/:month/:day/:title/ 链接格式

new_post_name: :title.md 解析标题方式

之前文章的命名方式都是 YYYY-MM-DD-title （jekyll 的文章格式），比如文件 2018-05-30-hexo-image.md 。

hexo 渲染时解析博文的标题为 2018-05-30-hexo-image 。

当如上设置 permalink 后，路径就是 http://hqweay.cn/2018/05/30/2018-05-30-hexo-image/ 这样。

时间出现了两次好诡异啊。

可以修改 permalink： ：title/ ，链接就直接用标题来区分。

效果是：http://hqweay.cn/2018-05-30-hexo-image/

但是我想导入评论，链接最好和之前 jekyll 的一致。

于是就想办法改改后面。

hexo 提供了 new_post_name 。这条属性是定义 hexo 对文章标题的解析方式的。

默认为 new_post_name: :title.md 。

我们修改为 new_post_name: :year-:month-:day-:title.md 。

对比文件命名，很容易理解。

这样，hexo 就会把自动提取出文件的命名中的文章标题，而不是直接把文件的命名解析为标题。

路径就为：

http://hqweay.cn/2018/05/30/hexo-image/


