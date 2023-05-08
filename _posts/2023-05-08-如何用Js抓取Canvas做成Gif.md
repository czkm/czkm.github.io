---
layout:     post                    # 使用的布局（不需要改）
title:      如何用Js抓取Canvas做成Gif       # 标题 
subtitle:   Canvas J s#副标题
date:       2023-05-08            # 时间
author:     czk                      # 作者
header-img: img/my/img18.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:  
#标签
    - js
    
---

某天在网上冲浪🏄‍♀️时发现了一个页面如下

![图1_mjqtm9_.png](https://cdn.jsdelivr.net/gh/czkm/img-folder@master/gif-render/图1_mjqtm9_.png)


看着gif觉得很有意思便想着保存下来，右键保存发现下载到本地的只有gif的第一帧，更换后缀名也没有生效

![图2_mg06cp_.png](https://cdn.jsdelivr.net/gh/czkm/img-folder@master/gif-render/图2_mg06cp_.png)

检查dom发现其元素是由canvas实现的 所以保存下来的图片只有当前帧

![图3_gx38gl_.png](https://cdn.jsdelivr.net/gh/czkm/img-folder@master/gif-render/图3_gx38gl_.png)

网络上有不少canvas转换的库，图省事的我决定直接在控制台写一段，JS抓取图片然后手动转换为Gif。

思路是逐帧抓取该gif的图片之后下载到本地 之后再手动合成

先写个抓取的js，给canvas 定一个id方便我们抓取
![图4_lkj0sg_.png](https://cdn.jsdelivr.net/gh/czkm/img-folder@master/gif-render/图4_lkj0sg_.png)

```javascript
var saveAsLocalImage = function () { 
    var canvas = document.getElementById("canvas"); 
    //直接改图片的mimeType，强制改成steam流类型的。比如"image/octet-stream"
    var image = canvas.toDataURL("image/png").replace("image/png", "image/octet-stream"); 
    window.location.href = image;
}
```

```javascript
function timer(count) {
  let i = 0;
  const intervalId = setInterval(() => {
    saveAsLocalImage()
    i++;
    if (i === count) {
      clearInterval(intervalId);
    }
  }, 200);
}

timer(15) // 15次
```

再写个定时器去调用，控制台执行后去点击gif

![图6_ajmmbt_.png](https://cdn.jsdelivr.net/gh/czkm/img-folder@master/gif-render/图6_ajmmbt_.png)


自此你就获得了 从gif开始时到结束的关键帧下载完毕之后，会得到20张截图。将多余的图片去掉后重命名为png格式的图片

可以通过其他的gif生成网站的或者直接用我推荐的

[Animated GIF Maker](https://gifmaker.me/)

![图7_pxekpg_.png](https://cdn.jsdelivr.net/gh/czkm/img-folder@master/gif-render/图7_pxekpg_.png)


![test 下午5.11.34_lha6ue_.gif](https://raw.githubusercontent.com/czkm/img-folder/main/gif-render/test%20%E4%B8%8B%E5%8D%885.11.34_lha6ue_.gif)

如此你就可以获得一个由静态图片合成的gif，你可以自由编辑其帧率速度等等，简单分享下其实没有涉及到多高深技术
