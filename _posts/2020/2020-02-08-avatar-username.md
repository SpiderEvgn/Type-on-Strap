---
layout: post
title: 根据用户名生成头像图片
date: 2020-02-08
tags: rails
---

### 0. 引言

介绍一种简单又直接的在 Rails 根据用户名字生成头像的方法。核心代码来自于 [https://github.com/getcampo/campo](https://github.com/getcampo/campo)，感谢 Rei 的开源项目。

### 1. 直接看代码

```ruby
# 定义了一个颜色数组用来随机生成头像背景色
DEFAULT_AVATAR_COLORS = %w(
  #007bff
  #6610f2
  #6f42c1
  #e83e8c
  #dc3545
  #fd7e14
  #ffc107
  #28a745
  #20c997
  #17a2b8
)

after_commit :generate_default_avatar, on: [:create]

def generate_default_avatar
  # 头像的临时地址
  temp_path = "#{Rails.root}/tmp/#{id}_default_avatar.png"
  # 用 Rails 的 system 方法运行 shell 脚本
  # 用 imagemagick 的 convert 方法生成图片
  # -font STYaoti_XM 字体必须是系统支持的，见下一节说明
  system(*%W(convert -size 120x120 -annotate 0 #{username} -font STYaoti_XM -fill white -pointsize 70 -gravity Center xc:#{DEFAULT_AVATAR_COLORS.sample} #{temp_path}))
  # 用 activestorage 的 attach 方法保存头像
  avatar.attach(io: File.open(temp_path), filename: "patient_id_#{id}.png")
  # 删除临时头像
  FileUtils.rm temp_path
end
```

### 2. 字体

笔者在实际操作中遇到一个很坑的问题：当 `username` 是英文字母或数字时可以正常显示，但是中文字显示不出来，只有背景色。

这其实是系统字体的问题，必须在『ImageMagic』注册它能够识别的中文字体。用以下命令可以查看当前支持的字体：

```ssh
$ convert -list font

Path: System Fonts
  Font: DejaVu-Sans
    family: DejaVu Sans
    style: Normal
    stretch: Normal
    weight: 400
    glyphs: /usr/share/fonts/truetype/dejavu/DejaVuSans.ttf
  Font: DejaVu-Sans-Bold
    family: DejaVu Sans
    style: Normal
    stretch: Normal
    weight: 700
    glyphs: /usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf
  ...
  Font: STYaoti_XM
    family: STYaoti
    style: Normal
    stretch: Normal
    weight: 400
    glyphs: /usr/share/fonts/truetype/custom/HuaWenYaoTi.ttf
```

可见，只要下载一个中文字体的 `ttf` 文件存入相应位置就行了，笔者下载了一个 `HuaWenYaoTi.ttf` 字体文件并保存到了 `/usr/share/fonts/truetype/custom/HuaWenYaoTi.ttf`，然后系统就能识别到这个 `STYaoti_XM` 字体了。

`convert` 是『ImageMagic』的命令，详情参考[ImageMagick 官网](https://imagemagick.org/script/command-line-options.php#font)。

---

## 参考资料：

* [GitHub campo](https://github.com/getcampo/campo)
* [ImageMagick 官网](https://imagemagick.org/script/command-line-options.php#font)
* [利用ImageMagicK给图片加水印](https://www.cnblogs.com/dying/articles/3085580.html)
* [开源中文字体库](http://www.fonts.net.cn/fonts-zh-1.html)