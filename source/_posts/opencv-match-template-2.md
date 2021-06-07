---
title: 使用 gocv 进行图片的模板匹配优化之 - 批量多模板匹配
date: 2021-06-07 11:28:11
tags: 
- golang
- opencv
categories: golang相关
---
## 前言
之前有针对图片的模板匹配有做了一个简单的版本:  {% post_link opencv-match-template %},  接下来针对做一下优化:
1. 之前代码写在一起， 现在当然要抽象出来
2. 允许多个原图进行多个模板来匹配
3. 最后输出的效果图，如果有多个模板匹配的话，都要全部圈出来

## 实操
首先 原图就两个 (`1.jpg` 和 `2.jpg`)， 然后模板图有3个 (分别是从 `1.jpg` 扣下一块， 从 `2.jpg` 扣下两块)。 最后奉上代码:

![1](1.png)

<!--more-->

```go
package main

import (
	"fmt"
	"time"
	"image/color"
	"image"
	"gocv.io/x/gocv"
)

func main() {
	// 读取原图和要匹配的模板
	templateArr := []string{
		"temp-2.jpg",
		"temp-3.jpg",
		"temp-4.jpg",
	}
	originArr := []string {
		"1.jpg",
		"2.jpg",
	}
	// 批量操作
	for _, value := range originArr {
		detectMultipleTemplate(value, templateArr)
	}

}
// 获取图片路径
func getImagePathByName(name string) string {
	return fmt.Sprintf("images/%v", name)
}

// 批量模板匹配
func detectMultipleTemplate(imgSceneName string, templateArr []string){
	startTime := time.Now().UnixNano()
	fmt.Println(fmt.Sprintf("----start match: %v", imgSceneName))
	imgScene := gocv.IMRead(getImagePathByName(imgSceneName), gocv.IMReadUnchanged)
	if imgScene.Empty() {
		fmt.Println("Invalid read of origin image in MatchTemplate test")
	}
	defer imgScene.Close()
	outPutImgScene := &imgScene
	matchCount := 0
	for _, value := range templateArr {
		isMatch := detectSingleTemplate(&imgScene, value, outPutImgScene)
		if isMatch {
			matchCount += 1
		}
	}
	// 最后输出匹配后并且有标记圈出的图片
	if matchCount > 0 {
		gocv.IMWrite(fmt.Sprintf("%v-output.jpg", getImagePathByName(imgSceneName)), *outPutImgScene)
	}
	endTime := time.Now().UnixNano()
	// 得到毫秒
	Milliseconds:= float64((endTime - startTime) / 1e6)
	fmt.Println(fmt.Sprintf("----end match %v, use %v ms, match count: %v", imgSceneName, Milliseconds, matchCount))
}

// 单个匹配模板
func detectSingleTemplate(imgScene *gocv.Mat, templateName string, outPutImgScene *gocv.Mat) bool {
	imgTemplate := gocv.IMRead(getImagePathByName(templateName), gocv.IMReadUnchanged)
	if imgTemplate.Empty() {
		fmt.Println("Invalid read of template image in MatchTemplate test")
	}
	defer imgTemplate.Close()
	// 开始匹配
	result := gocv.NewMat()
	defer result.Close()
	m := gocv.NewMat()
	gocv.MatchTemplate(*imgScene, imgTemplate, &result, gocv.TmCcoeffNormed, m)
	m.Close()
	// 获取最大匹配度 和 匹配范围
	_, maxConfidence, _, maxLoc := gocv.MinMaxLoc(result)
	fmt.Println(fmt.Sprintf("Template: %v -> max confidence %f, %v, %v", templateName, maxConfidence, maxLoc.X, maxLoc.Y))

	if maxConfidence < 0.8 {
		fmt.Println(fmt.Sprintf("Template: %v -> Max confidence of %f is too low. Not match", templateName, maxConfidence))
		return false
	}else {
		fmt.Println(fmt.Sprintf("Template: %v -> Max confidence of %f is high. Match !!!", templateName, maxConfidence))
		// 圈出来
		rectangle(outPutImgScene, &imgTemplate, maxLoc)
		return true
	}
}

// 圈起来
func rectangle(imgScene *gocv.Mat, imgTemplate *gocv.Mat, maxLoc image.Point) {
	// 将匹配的地方圈起来，并重新输出图片
	trows := imgTemplate.Rows()
	tcols  := imgTemplate.Cols()

	r := image.Rectangle{
		Min: maxLoc,
		Max: image.Pt(int(maxLoc.X + tcols) , int(maxLoc.Y) + trows),
	}
	// 用 2px 的红色框框 画出来
	gocv.Rectangle(imgScene, r, color.RGBA{255, 0, 0, 1},2)
}
```

然后执行一下:
```text
[root@VM-16-29-centos test]# go run main.go
----start match: 1.jpg
Template: temp-2.jpg -> max confidence 0.251532, 161, 245
Template: temp-2.jpg -> Max confidence of 0.251532 is too low. Not match
Template: temp-3.jpg -> max confidence 0.500531, 235, 277
Template: temp-3.jpg -> Max confidence of 0.500531 is too low. Not match
Template: temp-4.jpg -> max confidence 0.999982, 295, 249
Template: temp-4.jpg -> Max confidence of 0.999982 is high. Match !!!
----end match 1.jpg, use 188 ms, match count: 1
----start match: 2.jpg
Template: temp-2.jpg -> max confidence 0.999996, 724, 59
Template: temp-2.jpg -> Max confidence of 0.999996 is high. Match !!!
Template: temp-3.jpg -> max confidence 0.999994, 834, 255
Template: temp-3.jpg -> Max confidence of 0.999994 is high. Match !!!
Template: temp-4.jpg -> max confidence 0.452608, 764, 382
Template: temp-4.jpg -> Max confidence of 0.452608 is too low. Not match
----end match 2.jpg, use 707 ms, match count: 2
```

可以看到 第一张原图 match 一个模板， 总耗时 188ms， 第二张原图 match 了两个模板， 总耗时 707 ms

可以看下输出的结果图:

![1](2.png)

可以看到输出结果图是有正确圈出匹配的模板的。










