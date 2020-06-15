---
layout: post
title: VSCode 新增 Emmet 缩写
tags: [vscode]
---

### 1. 打开 vscode settings 界面

![vscode-settings](/assets/img/posts/2020/vscode-emmet-snippets/vscode-settings.jpg "vscode-settings")

### 2. 给 Emmet 添加自定义 snippet 配置文件路径（注意添加的实际上是文件夹位置）

![add-extension-path](/assets/img/posts/2020/vscode-emmet-snippets/add-extension-path.jpg "add extension path")

选中 Emmet，点击配置按钮，然后新增如下代码：

```
"emmet.extensionsPath": "~/Library/Application Support/Code/User/snippets"
```

默认就有 snippets 文件夹并且是空的，就直接拿来用了

### 3. 新建 snippets.json 文件

路径配置好了，接下来就新建配置文件，在 snippets 文件夹底下新建 snippets.json 文件。

最后就是具体的自定义 snippets 配置了，大概的格式如下：

```
{
  "html": {
    "snippets": {
      "rtr": "<Router>$1</Router>",
      "rt": "<Route path=\"$1\"></Route>",
    }
  }
}
```

***`$1` 表示缩写创建完成后光标停留的位置***