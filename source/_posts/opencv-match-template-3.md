---
title: 使用 gocv 进行图片的模板匹配优化之 - 缩放后快速匹配
date: 2021-06-07 13:28:30
tags: 
- golang
- opencv
categories: golang相关
---
## 前言
之前有针对图片的模板匹配有做了一个优化的版本:  {% post_link opencv-match-template-2 %},  但是还不够，还需要再优化两个点:
1. 直接从 origin 文件夹里面读取原图，从 template 文件夹里面读取模板图，随后如果有匹配的话，将匹配圈出来的原图输出到 output 文件夹
2. 配置是否启用同步缩放匹配 (好处就是匹配速度很快， 坏处就是如果模板图信息太少的话，会出现匹配度太低的情况)

第一点优化很好理解，如果 原图很多， 模板图也很多的话， 肯定要变成直接读取，然后遍历文件夹的方式， 匹配结束之后，如果有匹配上，然后统一输出到另一个文件夹中

第二点是考虑到如果原图 size 很大，匹配的过程就会很耗费 cpu，而且时间会比较长。 所以可以设置缩放因子，将原图和模板图按照固定的比例进行缩放， 然后再匹配。 这样子做的好处就是因为双方的像素点变少了，所以匹配的速度就会很快， 坏处就是如果模板图的可识别区域比较少，再加上压缩之后，就会出现匹配度太低，导致我们认为无法 match 的情况。

所以最好模板图的尺寸不能低于 30px。 而且缩放的比例也不能高于 0.5， 这样子速度才会快。

<!--more-->

## 实操
采取一批原图放到 `images/origin` 文件夹

![1](1.png)

然后放一批模板图放到 `images/template` 文件夹

![1](2.png)

接下来就是代码

```go
package main

import (
	"fmt"
	"time"
	"image/color"
	"image"
	"io/ioutil"
	"gocv.io/x/gocv"
	"strings"
)

const (
	MinSize = 30  // 模板的最小尺寸不能小于 30px
	MaxScale = 0.5  // 最大缩放比例不能超过 0.5
	UseScale = true // 是否采用缩小因子
)

func main() {
	var s []string
	originArr, _ := GetAllFile("images/origin", s)
	fmt.Printf("origin slice: %v", originArr)

	templateArr, _ := GetAllFile("images/template", s)
	fmt.Printf("templage slice: %v", templateArr)

	// 批量操作
	for _, value := range originArr {
		detectMultipleTemplate(value, templateArr, UseScale)
	}

}
// 遍历文件夹中的文件
func GetAllFile(pathname string, s []string) ([]string, error) {
	rd, err := ioutil.ReadDir(pathname)
	if err != nil {
		fmt.Println("read dir fail:", err)
		return s, err
	}
	for _, fi := range rd {
		if fi.IsDir() {
			fullDir := pathname + "/" + fi.Name()
			s, err = GetAllFile(fullDir, s)
			if err != nil {
				fmt.Println("read dir fail:", err)
				return s, err
			}
		} else {
			fullName := pathname + "/" + fi.Name()
			s = append(s, fullName)
		}
	}
	return s, nil
}


// 批量模板匹配
func detectMultipleTemplate(imgScenePath string, templateArr []string, useScale bool){
	startTime := time.Now().UnixNano()
	fmt.Println(fmt.Sprintf("----start match: %v", imgScenePath))
	// 因为我这一批测试的图片，都是只有 rgb 通道， 所以初始化的时候，就要指定好，不然会报错
	imgScene := gocv.IMRead(imgScenePath, gocv.IMReadColor)
	if imgScene.Empty() {
		fmt.Println("Invalid read of origin image in MatchTemplate test")
	}
	defer imgScene.Close()

	outPutImgScene := &imgScene
	matchCount := 0
	for _, value := range templateArr {
		isMatch, scaleFactor := detectSingleTemplate(&imgScene, value, outPutImgScene, useScale)
		if isMatch {
			matchCount += 1
			fmt.Println(fmt.Sprintf("is match, scale: %v", scaleFactor))
		}
	}
	// 最后输出匹配后并且有标记圈出的图片
	if matchCount > 0 {
		// 这边只取出最后的路径
		pathArr := strings.Split(imgScenePath, "/")
		gocv.IMWrite(fmt.Sprintf("images/output/%v", pathArr[len(pathArr) - 1]), *outPutImgScene)
	}
	endTime := time.Now().UnixNano()
	// 得到毫秒
	Milliseconds:= float64((endTime - startTime) / 1e6)
	fmt.Println(fmt.Sprintf("----end match %v, use %v ms, match count: %v", imgScenePath, Milliseconds, matchCount))
}


// 单个匹配模板
func detectSingleTemplate(imgScene *gocv.Mat, templateUri string, outPutImgScene *gocv.Mat, useScale bool) (bool, float64) {
	imgTemplate := gocv.IMRead(templateUri, gocv.IMReadColor)
	if imgTemplate.Empty() {
		fmt.Println("Invalid read of template image in MatchTemplate test")
	}
	defer imgTemplate.Close()

	// 开始匹配
	result := gocv.NewMat()
	m := gocv.NewMat()
	var scaleFactor float64 = 1
	// 如果有使用缩小因子
	if useScale {
		// 在 match 之前，我们要先图片进行缩放，因为有些原图或者缩略图偏大， 如果没有缩小的话， 匹配的速度不仅慢，而且耗 cpu
		// 初始化原图的缩小图
		fullScaleImage := gocv.NewMat()
		scaleFactor = scaleFullAndTemplateImage(imgScene, &imgTemplate, &fullScaleImage)
		// 开始匹配
		resultWidth := fullScaleImage.Cols() - imgTemplate.Cols() + 1
		resultHeight := fullScaleImage.Rows() - imgTemplate.Rows() + 1
		result = gocv.NewMatWithSize(resultWidth, resultHeight, gocv.MatTypeCV32FC1)
		defer result.Close()
		gocv.MatchTemplate(fullScaleImage, imgTemplate, &result, gocv.TmCcoeffNormed, m)
		m.Close()
	}else{
		defer result.Close()
		gocv.MatchTemplate(*imgScene, imgTemplate, &result, gocv.TmCcoeffNormed, m)
		m.Close()
	}

	// 获取最大匹配度 和 匹配范围
	_, maxConfidence, _, maxLoc := gocv.MinMaxLoc(result)
	fmt.Println(fmt.Sprintf("Template: %v -> max confidence %f, %v, %v", templateUri, maxConfidence, maxLoc.X, maxLoc.Y))

	if maxConfidence <= 0.8 {
		fmt.Println(fmt.Sprintf("Template: %v -> Max confidence of %f is too low. Not match", templateUri, maxConfidence))
		return false, scaleFactor
	}else {
		fmt.Println(fmt.Sprintf("Template: %v -> Max confidence of %f is high. Match !!!", templateUri, maxConfidence))
		// 圈出来
		rectangle(outPutImgScene, &imgTemplate, maxLoc, scaleFactor)
		return true, scaleFactor
	}
}

// 按照缩小因子来缩小图片
func scaleFullAndTemplateImage(imgScene *gocv.Mat, imgTemplate *gocv.Mat, fullScaleImage *gocv.Mat) float64{
	// 在 match 之前，我们要先图片进行缩放，因为有些原图或者缩略图偏大， 如果没有缩小的话， 匹配的速度不仅慢，而且耗 cpu
	// 最小图片尺寸不可以小于 30px
	widthCompare := MinSize / float64(imgTemplate.Cols())
	heightCompare := MinSize  / float64(imgTemplate.Rows())
	scaleFactor := heightCompare
	if widthCompare > heightCompare {
		scaleFactor = widthCompare
	}
	if scaleFactor > MaxScale {
		scaleFactor = MaxScale
	}

	fmt.Println(fmt.Sprintf("scale: %v, %v, %v, %v, %v", scaleFactor, widthCompare, heightCompare,imgTemplate.Cols(), imgTemplate.Rows()))

	// 根据缩放因子同步缩小 full Image 和 template Image
	fullScaleImageSize := image.Point{
		X: int(float64(imgScene.Cols()) * scaleFactor),
		Y: int(float64(imgScene.Rows()) * scaleFactor),
	}
	gocv.Resize(*imgScene, fullScaleImage, fullScaleImageSize, 0, 0, gocv.InterpolationDefault)
	templateScaleImageSize := image.Point{
		X: int(float64(imgTemplate.Cols()) * scaleFactor),
		Y: int(float64(imgTemplate.Rows()) * scaleFactor),
	}
	gocv.Resize(*imgTemplate, imgTemplate, templateScaleImageSize, 0, 0, gocv.InterpolationDefault)

	return scaleFactor
}

// 圈起来
func rectangle(imgScene *gocv.Mat, imgTemplate *gocv.Mat, maxLoc image.Point, scaleFactor float64) {
	// 将匹配的地方圈起来，并重新输出图片
	trows := imgTemplate.Rows()
	tcols  := imgTemplate.Cols()
	// 重新恢复成原来的大小，然后在原图上圈出来
	r := image.Rectangle{
		Min: image.Pt(int(float64(maxLoc.X) / scaleFactor) , int(float64(maxLoc.Y) / scaleFactor)),
		Max: image.Pt(int(float64(maxLoc.X + tcols) / scaleFactor) , int(float64(maxLoc.Y + trows) / scaleFactor)),
	}
	// 用 2px 的红色框框 画出来
	gocv.Rectangle(imgScene, r, color.RGBA{255, 0, 0, 1},2)
}
```

注意我们是通过 `UseScale` 来判断是否要采用缩放因子的方式来测试。

而且需要注意一点的是，这一批原图 和 模板图都是只有 GRB 三个通道的，而不是正常的那种 RGBA 四个通道的， 所以读取图片的时候，要指定读取的是 RGB 三个通道的, 也就是第二个参数要指定好， 不然会有问题
```text
imgTemplate := gocv.IMRead(templateUri, gocv.IMReadColor)
```

## 测试1 -> 不缩放
也就是 `UseScale = false`

因为日志比较长，我就只取一个看看:
```text
----start match: images/origin/SSS-1041_ScreenShot_20210521_2.png
Template: images/template/1.jpg -> max confidence 0.225314, 241, 211
Template: images/template/1.jpg -> Max confidence of 0.225314 is too low. Not match
Template: images/template/2.jpg -> max confidence 0.343062, 213, 683
Template: images/template/2.jpg -> Max confidence of 0.343062 is too low. Not match
Template: images/template/3.jpg -> max confidence 0.351453, 220, 167
Template: images/template/3.jpg -> Max confidence of 0.351453 is too low. Not match
Template: images/template/4.end.jpg -> max confidence 0.336641, 428, 77
Template: images/template/4.end.jpg -> Max confidence of 0.336641 is too low. Not match
Template: images/template/5.end.jpg -> max confidence 0.452431, 326, 342
Template: images/template/5.end.jpg -> Max confidence of 0.452431 is too low. Not match
Template: images/template/6.end.jpg -> max confidence 0.941337, 416, 84
Template: images/template/6.end.jpg -> Max confidence of 0.941337 is high. Match !!!
is match, scale: 1
Template: images/template/7.end.jpg -> max confidence 0.410051, 217, 666
Template: images/template/7.end.jpg -> Max confidence of 0.410051 is too low. Not match
Template: images/template/8.end.jpg -> max confidence 0.329708, 195, 162
Template: images/template/8.end.jpg -> Max confidence of 0.329708 is too low. Not match
----end match images/origin/SSS-1041_ScreenShot_20210521_2.png, use 2006 ms, match count: 1
```
平均下来，在不缩放的情况下， 一张原图要匹配 8 张模板图的情况， 要花  2s 左右。 最后能匹配到的图有 15 张

![1](3.png)

## 测试2 -> 有缩放
也就是 `UseScale = true`

有缩放之后，明显执行就快多了，截取同一个
```text
----start match: images/origin/SSS-1041_ScreenShot_20210521_2.png
scale: 0.07142857142857142, 0.06818181818181818, 0.07142857142857142, 440, 420
Template: images/template/1.jpg -> max confidence 0.236107, 17, 15
Template: images/template/1.jpg -> Max confidence of 0.236107 is too low. Not match
scale: 0.5, 0.13513513513513514, 0.6818181818181818, 222, 44
Template: images/template/2.jpg -> max confidence 0.363811, 107, 342
Template: images/template/2.jpg -> Max confidence of 0.363811 is too low. Not match
scale: 0.5, 0.17045454545454544, 0.5882352941176471, 176, 51
Template: images/template/3.jpg -> max confidence 0.366116, 110, 84
Template: images/template/3.jpg -> Max confidence of 0.366116 is too low. Not match
scale: 0.5, 0.1271186440677966, 0.5555555555555556, 236, 54
Template: images/template/4.end.jpg -> max confidence 0.362354, 214, 38
Template: images/template/4.end.jpg -> Max confidence of 0.362354 is too low. Not match
scale: 0.12552301255230125, 0.12552301255230125, 0.10238907849829351, 239, 293
Template: images/template/5.end.jpg -> max confidence 0.457758, 40, 43
Template: images/template/5.end.jpg -> Max confidence of 0.457758 is too low. Not match
scale: 0.5, 0.17857142857142858, 0.8108108108108109, 168, 37
Template: images/template/6.end.jpg -> max confidence 0.914187, 208, 42
Template: images/template/6.end.jpg -> Max confidence of 0.914187 is high. Match !!!
is match, scale: 0.5
scale: 0.2542372881355932, 0.15873015873015872, 0.2542372881355932, 189, 118
Template: images/template/7.end.jpg -> max confidence 0.425736, 55, 169
Template: images/template/7.end.jpg -> Max confidence of 0.425736 is too low. Not match
scale: 0.5, 0.0949367088607595, 0.5, 316, 60
Template: images/template/8.end.jpg -> max confidence 0.353062, 98, 81
Template: images/template/8.end.jpg -> Max confidence of 0.353062 is too low. Not match
----end match images/origin/SSS-1041_ScreenShot_20210521_2.png, use 361 ms, match count: 1
```

可以看到缩放之后，速度就很快了， 360 ms， 快了 6 倍。 不过可以看到原本不缩放的 match 点的匹配度是 94， 缩放后虽然也匹配上了，但是匹配度只有 91。 所以是有可能是会出现原图可以匹配上，但是缩放之后，就匹配不上了。上图的缩放比例是 0.5， 如果缩放比例更小的话， 这种概率更大。

事实上也是如此， 有缩放之后的匹配图， 其实只有 11 张

![1](4.png)

有四张本来原图可以匹配上的， 但是缩放之后，匹配度达不到 80 了，所以就算没有匹配了。 我们可以找一张看下数据:
```text
----start match: images/origin/1622309716082.jpg.jpg
...
Template: images/template/2.jpg -> max confidence 0.975171, 503, 67
Template: images/template/2.jpg -> Max confidence of 0.975171 is high. Match !!!
is match, scale: 1
...
----end match images/origin/1622309716082.jpg.jpg, use 2082 ms, match count: 1
```

```text
----start match: images/origin/1622309716082.jpg.jpg
...
scale: 0.5, 0.13513513513513514, 0.6818181818181818, 222, 44
Template: images/template/2.jpg -> max confidence 0.784327, 252, 33
Template: images/template/2.jpg -> Max confidence of 0.784327 is too low. Not match
...
----end match images/origin/1622309716082.jpg.jpg, use 359 ms, match count: 0
```
可以看到原图是 97 的匹配度，缩放之后，只剩下 78 的匹配度了。 也就不算匹配上了。 (程序要大于等于 80 才算匹配上)


## 结论
缩放因子虽然运行速度会比较快，但是如果模板可识别因子太少的话，就会出现匹配度太低的情况，所以尽可能让模板图片有更多明显的标志物， 而且也不能尺寸太小。




