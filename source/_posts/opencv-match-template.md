---
title: 使用 gocv 进行图片的模板匹配
date: 2021-06-02 20:07:46
tags: 
- golang
- opencv
categories: golang相关
---
## 前言
之前成功安装了 gocv: {% post_link opencv-gocv %}, 今天就写个 demo 来测试一下。

## demo 逻辑
逻辑如下， 在一张原图中， 从中截一个小片段出来，然后去匹配， 看看匹配度是多少， 然后在原图中圈出要匹配的模板的位置。

比如我从这一张原图中，截取艾斯右手燃烧的部分出来。

![1](1.png)

<!--more-->
## 代码如下:
```golang
package main

import (
	"fmt"
	"image/color"
	"image"
	"gocv.io/x/gocv"
)

func main() {
	// 读取原图和要匹配的模板
	imgScene := gocv.IMRead("images/1.jpg", gocv.IMReadUnchanged)
	if imgScene.Empty() {
		fmt.Println("Invalid read of origin image in MatchTemplate test")
	}
	defer imgScene.Close()

	imgTemplate := gocv.IMRead("images/temp-1.jpg", gocv.IMReadUnchanged)
	if imgTemplate.Empty() {
		fmt.Println("Invalid read of template image in MatchTemplate test")
	}
	defer imgTemplate.Close()
	// 开始匹配
	result := gocv.NewMat()
	defer result.Close()
	m := gocv.NewMat()
	gocv.MatchTemplate(imgScene, imgTemplate, &result, gocv.TmCcoeffNormed, m)
	m.Close()
	// 获取最大匹配度 和 匹配范围
	_, maxConfidence, _, maxLoc := gocv.MinMaxLoc(result)
	fmt.Println(fmt.Sprintf("max confidence %f, %v, %v", maxConfidence, maxLoc.X, maxLoc.Y))

	if maxConfidence < 0.9 {
		fmt.Println(fmt.Sprintf("Max confidence of %f is too low. Not match", maxConfidence))
	}else {
		fmt.Println(fmt.Sprintf("Max confidence of %f is high. Match !!!", maxConfidence))
	}

	// 将匹配的地方圈起来，并重新输出图片
	trows := imgTemplate.Rows()
	tcols  := imgTemplate.Cols()

	r := image.Rectangle{
		Min: maxLoc,
		Max: image.Pt(int(maxLoc.X + tcols) , int(maxLoc.Y) + trows),
	}
	// 用 2px 的红色框框 画出来
	gocv.Rectangle(&imgScene, r, color.RGBA{255, 0, 0, 1},2)

	gocv.IMWrite("images/out.jpg", imgScene)
}
```

接下来看一下运行结果:
```javascript
[root@VM-16-29-centos test]# go run main.go 
max confidence 0.999970, 428, 209
Max confidence of 0.999970 is high. Match !!!
```
可以看到匹配度很高，几乎为 1， 我们再看下新输出的图片有没有正常圈上。

![1](2.png)

可以看到，有正确圈上了。 说明图片模板的匹配是没问题。 当然这个只是最基本的，后面还有进阶版

---
参考资料:
- [opencv关于模板匹配cvMatchTemplate的运用](https://blog.csdn.net/gdut2015go/article/details/45971165)
- [opencv-matchTemplate 之使用场景为大图里面找小图](https://blog.csdn.net/weixin_44517891/article/details/107013842)
- [Template Matching in OpenCV](https://docs.opencv.org/master/d4/dc6/tutorial_py_template_matching.html)






