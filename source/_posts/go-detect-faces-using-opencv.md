# 使用 Golang 和 OpenCV 侦测人脸

![](https://img.zvz.im/imgs/2020/04/d055ce9a0a5c233c.png)

OpenCV 是一个用于计算机视觉处理的代码库，面世已有 20 多年了。大学时期，我曾在个人的 C++ 和 Python项目中使用过它，因为这些编程语言对它有很好的支持。不过随着我开始学习并使用 Go 语言，我开始好奇 Go 语言能否使用 OpenCV。网上有一些关于如何使用 Go 语言调用 OpenCV 的例子和教程，但我发现它们都太过黑科技和复杂了。还好我发现了一个名为 [*hybridgroup*](https://github.com/hybridgroup) 小组的伙计们写的封装库，它很容易使用，而且文档也很全。这里我要向你们展示如何使用 gocv ，并且创建一个简单的 Haar Cascades 面部探测器。

**准备工作**

* Go
* OpenCV （下文附有安装链接）
* 一个网络摄像头

**安装地址**

linux: [https://gocv.io/getting-started/linux/](https://gocv.io/getting-started/linux/)

macOS: [https://gocv.io/getting-started/macos/](https://gocv.io/getting-started/macos/)

windows: [https://gocv.io/getting-started/windows/](https://gocv.io/getting-started/windows/)

## 实例一

在第一个例子中，让我们尝试打开一个窗口，并显示从你的摄像头获取到的视频流。

首先引入我们需要的库。

```go
import (
   “log”
   “gocv.io/x/gocv”
)
```

然后使用 [VideoCaptureDevice](https://github.com/hybridgroup/gocv/blob/master/videoio.go#L181) 方法创建一个 [VideoCapture](https://github.com/hybridgroup/gocv/blob/master/videoio.go#L160) 对象。[VideoCaptureDevice](https://github.com/hybridgroup/gocv/blob/master/videoio.go#L181)  方法能让你从摄像头中获取一个视频流。该方法需要一个表示设备 ID 的整型参数。

```go
webcam, err := gocv.VideoCaptureDevice(0)
if err != nil {
    log.Fatalf(“error opening web cam: %v”, err)
}
defer webcam.Close()
```

我们需要创建一个窗口来展示视频流。可以使用 [NewWindow](https://github.com/hybridgroup/gocv/blob/master/highgui.go#L33) 方法完成这个任务。

```go
window := gocv.NewWindow(“webcamwindow”)
defer window.Close()
```

现在到了有趣的时候。

由于视频是一个持续不断的图像流，我们将不得不使用一个无限循环持续不断地从摄像头读取数据。为此我们将使用 VideoCapture 类型的 [Read](https://github.com/hybridgroup/gocv/blob/master/videoio.go#L217) 方法。它需要一个 [Mat 类型](https://github.com/hybridgroup/gocv/blob/master/core.go#L105) （我们在上文创建的矩阵）入参，同时返回一个布尔值表示 VideoCapture 是否成功读取到了帧数据。

```go
for {
     if ok := webcam.Read(&img); !ok || img.Empty() {
        log.Println(“Unable to read from the webcam”)
        continue
     }
.
.
.
}
```

最后我们把图像帧显示在创建的窗口中，等待 50ms 后再处理下一帧。

```go
window.IMShow(img)
window.WaitKey(50)
```

当运行程序时，我们可以看到一个窗口会弹出，里面显示着你的摄像头中的视频流。

![](https://img.zvz.im/imgs/2020/04/e5e8627147512749.png)

```go
package main

import (
	"log"

	"gocv.io/x/gocv"
)

func main() {
	webcam, err := gocv.VideoCaptureDevice(0)
	if err != nil {
    log.Fatalf("error opening device: %v", err)
	}
	defer webcam.Close()

	img := gocv.NewMat()
	defer img.Close()

	window := gocv.NewWindow("webcamwindow")
	defer window.Close()

	for {
		if ok := webcam.Read(&img); !ok || img.Empty() {
			log.Println("Unable to read from the webcam")
			continue
		}

		window.IMShow(img)
		window.WaitKey(50)
	}
}
```

## 实例二

此例中，我们将在上一个例子的基础上使用 Haar Cascades 进行人脸侦测。

不过首先。。什么是 Haar Cascades ？

简单来讲 Haar cascades 是基于哈尔小波（Haar Wavelet）技术训练得到的层叠分类器。它通过分析图片中的像素来侦测其中的特征。想要了解更多关于 Haar-Cascades 的知识你可以访问以下链接。

[Viola-Jones object detection framework](https://www.cs.ubc.ca/~lowe/425/slides/13-ViolaJones.pdf)

[Cascading classifiers](https://en.wikipedia.org/wiki/Cascading_classifiers)

[Haar-like feature](https://en.wikipedia.org/wiki/Haar-like_feature)

你可以从 [opencv 的代码库](https://github.com/opencv/opencv/tree/master/data/haarcascades)中下载预先训练好的 Haar-Cascades。此例中我们将使用 Haar-Cascade 帮助我们识别人的面部。

首先我们创建一个[分类器](https://github.com/hybridgroup/gocv/blob/master/objdetect.go#L23)并且将预先训练好的 Haar-Cascade 文件给到它。这个例子中我已经下载了 opencv_haarcascade_frontalface_default.xml 文件放到了我们的程序所在的目录。

```go
harrcascade := “opencv_haarcascade_frontalface_default.xml”
classifier := gocv.NewCascadeClassifier()
classifier.Load(harrcascade)
defer classifier.Close()
```

然后调用 [DetectMultiScale](https://github.com/hybridgroup/gocv/blob/master/objdetect.go#L51) 方法来识别图片中的脸。这个函数会使用刚从摄像头读取到的帧（[Mat Type](https://github.com/hybridgroup/gocv/blob/master/core.go#L105)）作为参数并且返回一组 [Rectangle](https://golang.org/src/image/geom.go#L85) 类型数据。数组的大小代表了在图片中识别到的脸的个数。为了确认我们可以考到侦测的结果，让我们遍历这一组矩形并且将它们输出到终端在矩形周围显示边框。你可以通过调用 Rectangle 方法来实现。这个方法的入参有摄像头读取的 Mat，一个由 DetectMultiScale 方法返回的 Rectangle 对象，一个颜色和一个边框粗细。

```go
for _, r := range rects {
fmt.Println(“detected”, r)
gocv.Rectangle(&img, r, color, 2)
}
```

最后，让我们运行这个程序。

![](https://img.zvz.im/imgs/2021/05/71175eb78a8f1ec4.png)

![](https://img.zvz.im/imgs/2021/05/8c791ccc15c0ea43.png)

```go
package main

import (
	"fmt"
	"image/color"
	"log"

	"gocv.io/x/gocv"
)

func main() {
	webcam, err := gocv.VideoCaptureDevice(0)
	if err != nil {
		log.Fatalf("error opening web cam: %v", err)
	}
	defer webcam.Close()

	img := gocv.NewMat()
	defer img.Close()

	window := gocv.NewWindow("webcamwindow")
	defer window.Close()

	harrcascade := "opencv_haarcascade_frontalface_default.xml"
	classifier := gocv.NewCascadeClassifier()
	classifier.Load(harrcascade)
	defer classifier.Close()

	color := color.RGBA{0, 255, 0, 0}
	for {
		if ok := webcam.Read(&img); !ok || img.Empty() {
			log.Println("Unable to read from the device")
			continue
		}

		rects := classifier.DetectMultiScale(img)
		for _, r := range rects {
			fmt.Println("detected", r)
			gocv.Rectangle(&img, r, color, 3)
		}

		window.IMShow(img)
		window.WaitKey(50)
	}
}
```

大功告成！我们有了一个 Go 编写的简单的面部探测器。我计划继续扩展这个系列并且使用 Go 和 OpenCV 构建更多的酷应用。请继续关注后续的文章。



翻译自：[Detect faces using Golang and OpenCV](https://medium.com/@fonseka.live/detect-faces-using-golang-and-opencv-fbe7a48db055)

