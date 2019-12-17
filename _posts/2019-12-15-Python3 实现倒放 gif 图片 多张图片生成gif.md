---
layout:     post
title:      Python3 实现 gif 倒放,多张图片生成 gif
subtitle:   Python3 实现 gif 倒放,多张图片生成 gif
date:       2019-12-15
author:     he xiaodong
header-img: img/default-post-bg.jpg
catalog: true
tags:
    - python3
    - PTL
    - imageio
    - gif倒放
    - reverse gif
    - generate gif
    - python 合并图片成 gif
---

> 一个娱乐代码，将表情包 gif 倒放很搞笑，当成自己的一个小玩具，操作就是将图片读取成帧，倒排合并一下，成为新的图片，完成。

效果：翻转前 ![nba.gif](https://alpha2016.github.io/img/2019-12-15-nba.gif) 翻转后 ![r-nba.gif](https://alpha2016.github.io/img/2019-12-15-r-nba.gif)

代码如下：
```python
# encoding: utf-8

from PIL import Image, ImageSequence 

# 读取GIF

im = Image.open("nba.gif")

# GIF图片流的迭代器

iter = ImageSequence.Iterator(im)

index = 1
# 遍历图片流的每一帧

for frame in iter:
    print("image %d: mode %s, size %s" % (index, frame.mode, frame.size))
    frame.save("./images/frame%d.png" % index)
    index += 1

# frame0 = frames[0]

# frame0.show()

# 把GIF拆分为图片流

imgs = [frame.copy() for frame in ImageSequence.Iterator(im)]

# 把图片流重新成成GIF动图

# imgs[0].save('out.gif', save_all=True, append_images=imgs[1:])

# 图片流反序

imgs.reverse()

# 将反序后的所有帧图像保存下来

imgs[0].save('./reverse_out.gif', save_all=True, append_images=imgs[1:])
```

**另外极简版更简单**
```python
# encoding: utf-8

from PIL import Image, ImageSequence 

im = Image.open(r'./nba.gif')
sequence = []

for f in ImageSequence.Iterator(im):
    sequence.append(f.copy())    

sequence.reverse()
sequence[0].save(r'./r-nba.gif',save_all = True, append_images=sequence[1:])
```

**将多张静态图片生成 gif 的代码，这个年底可以将每年的图片合并成一个大的 gif 以留作回忆，倒是很有用的**
```python
# coding=utf8

import imageio
import os 

path = '../images'
filenames = []
for files in os.listdir(path):
    if files.endswith('jpg') or files.endswith('jpeg') or files.endswith('png'):
        file = os.path.join(path,files)
        filenames.append(file)

images = []
for filename in filenames:
    images.append(imageio.imread(filename))
    
imageio.mimsave('movie.gif', images, duration = 0.3)
```


原作者不建议使用 image2gif 包，[详见一楼回答](https://stackoverflow.com/questions/753190/programmatically-generate-video-or-animated-gif-in-python)

当然如果你有需求将一组**尺寸一样的照片生成 gif**，可以邮箱 hexiaodong1992@outlook.com

参考链接：
1. [知乎 CVPy 专栏](https://zhuanlan.zhihu.com/p/32874659)
2. [imageio 官方文档](https://imageio.readthedocs.io/en/stable/examples.html)

