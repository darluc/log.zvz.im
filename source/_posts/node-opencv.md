title: OpenCV 入门：Node.js 计算机视觉处理
date: 2017-3-20 19:21:42
tags:
- Nodejs
- opencv
---

在这篇 OpenCV 入门文章中，我将会向大家展示如何使用 Node.js 进行计算机视觉处理。并且结合实例讲解使用 OpenCV 这个开源库进行图像处理的基础方法。

目前，我正在完成我的硕士论文，其中使用到了 React Native，神经网络，和 OpenCV 计算机视觉库。请允许我向你们展示一些我在使用 OpenCV 过程中学习到的一些东西。

> 计算机视觉是计算机科学中的一个领域，主要专注于使用不同的算法从图像和视频中获取数据。

计算机视觉在许多领域得到了广泛地应用，例如安全摄像头的运动跟踪，控制车辆进行自动驾驶，从图片或视频中识别或搜索对象。

要实现计算机视觉算法是一件非常繁复的工作，不过幸好有 [OpenCV](http://opencv.org/) 这个非常好的开源库，此库起源于 1999 年，并一直发展到现在。

OpenCV 官方支持 C，C ++，Python 和 Java。幸运的是，由 Peter Braden 领导的一群 Javascript 程序员开发了一个 Javascript 的 OpenCV 接口库，名为 [node-opencv](https://github.com/peterbraden/node-opencv)。

利用该接口库，我们可以实现用于图像分析的 Node.js 应用。此库目前还没有实现所有的 OpenCV 特性 - 特别是 OpenCV 3 的一些特性 - 不过已经基本够用了。
<!--more-->
## 安装

要在 Node.js 中使用 OpenCV 库，你得先进行全局安装。在 MacOS 上，你可以通过 [Homebrew](http://brew.sh/) 来安装。在这篇文章中我安装并使用的是 OpenCV 的 2.4 版本。

> 译者注：由于译者实际使用的是 OpenCV 3.2.0 版本，故在边缘侦测部分的代码相对于原文有所修改。

```shell
brew tap homebrew/science
brew install opencv
```

如果你使用的是其它的操作系统，这里有 [Linux](http://docs.opencv.org/2.4/doc/tutorials/introduction/linux_install/linux_install.html#linux-installation) 和 [Windows](http://docs.opencv.org/2.4/doc/tutorials/introduction/windows_install/windows_install.html#windows-installation) 版本的教程。在成功安装之后，我们就可以在 Node.js 项目中安装 node-opencv 了。

```shell
npm install --save opencv
```

有时安装会失败（它是开源项目，还没有到达最终完成的阶段），不过你应该可以在[该项目的 GitHub 页面](https://github.com/peterbraden/node-opencv)上找到对应的解决办法。

## OpenCV 基础

### 加载及保存图片 + 矩阵

OpenCV 的最基本功能是加载和保存图像。你可以通过下面的方法调用这些功能：**cv#readImage()** 和 **Maritx#save()**；

```javascript
const cv = require('opencv');

cv.readImage('./img/myImage.jpg', function(err, img) {
  if (err) {
    throw err;
  }
  
  const width = im.width();
  const height = im.height();
  
  if (width < 1 || height < 1) {
    throw new Error('Image has no size');
  }
  
  // do some cool stuff with img
  
  // save img
  img.save('./img/myNewImage.jpg');
});
```

![](https://img.zvz.im/imgs/2019/06/c1afacfcbce536d3.jpg)

承载加载图片数据的对象，是 OpenCV 使用的一个基本数据结构 - 矩阵。所有载入和生成的图像都是用矩阵来描述的，矩阵中的每一个元素都对应图像的一个像素。矩阵的大小由载入图像的大小决定。在 Node.js 中你可以使用特定参数调用 new Matrix() 构造方法来生成一个矩阵。

```javascript
new cv.Matrix(rows, cols);
new cv.Matrix(rows, cols, type, fillValue);
```

### 修改图像

变换图像颜色是一个基础方法。例如，我们可以调用 **Matrix#convertGrayscale()** 得到一个灰度图片。

```javascript
img.convertGrayScale();
img.save('./img/myGrayscaleImg.jpg');
```

![](https://img.zvz.im/imgs/2019/06/470560edbd5675be.jpg)

在进行边缘探测时经常会用到这个方法。

我们可以使用 **Matrix#convertHSVscale()** 方法将图像转换为 HSV 圆柱坐标表示（HSV cylindrical-coordinate representation ）。

```javascript
img.convertHSVscale();
img.save('./img/myHSVscaleImg.jpg');
```

![](https://img.zvz.im/imgs/2019/06/2d1840e7761522b8.jpg)

我们可以使用 **Matrix#crop(x, y, width, height)** 方法来裁剪图片，并指定其中的参数。

这个方法并不会改变当前的图像，而是返回一个全新的图像。

```javascript
let croppedImg = img.crop(1000, 1000, 1000, 1000);
croppedImg.save('./img/croppedImg');
```

如果我们想要将图像对象赋值给另一个变量，可以使用 **Matrix#copy()** 方法返回一个新的图片对象。

```javascript
let newImg = img.copy();
```

这样，我们可以用一些基础的 Matrix 方法进行工作了。我们还能找到各种模糊和滤镜方法来进行图片编辑。你可以在 GitHub 项目里的 [Matrix.cc](https://github.com/peterbraden/node-opencv/blob/976788c411cdff098247c40a17082c0ddaeaaadd/src/Matrix.cc) 文件中找到 Matrix 对象实现的所有方法。

### 腐蚀和膨胀

腐蚀和膨胀是数学形态学（ mathematical morphology ）的基本方法。我将结合下面的图像修改操作来解释它们是如何工作的。（译者注：具体数学定义可[参考此文](http://media.cs.tsinghua.edu.cn/~ahz/digitalimageprocess/chapter08/chapt08_ahz.htm)）

![](https://img.zvz.im/imgs/2019/06/c2316008cb739a25.jpg)

二进制图像 A 通过结构元素 B 的膨胀定义如下

![](https://img.zvz.im/imgs/2019/06/97dd7ea051bb7cbb.jpg)

OpenCV 有一个 **Matrix#dilate(iterations, structEl)** 方法，其中的 **iterations** 参数指定膨胀的量，**structEl** 参数是用于膨胀的结构元素（默认为 3X3）。

我们可以用此参数调用膨胀方法。

```javascript
img.dilate(3);
```

OpenCV 调用膨胀方法时如下。

```c++
cv::dilate(self->mat, self->mat, sturctEl, cv::Point(-1, -1), 3);
```

调用过此方法后，我们可以得到如下修改过的图像。

![](https://img.zvz.im/imgs/2019/06/e4af008c72801d71.jpg)

二进制图像 A 通过结构元素 B 的腐蚀定义如下

![](https://img.zvz.im/imgs/2019/06/6c51dfe0d4de36c5.jpg)

在 OpenCV 中，我们可以调用 **Martix#erode(iterations, structEl)** 方法，和前面的膨胀方法相似。

我们可以这样调用它：

```javascript
img.erode(3);
```

同样我们可以得到一个腐蚀过的图像。

![](https://img.zvz.im/imgs/2019/06/c200eeb23703b2c0.jpg)

### 边缘侦测

关于边缘侦测，我们可以使用「坎尼边缘探测算法」。该算法起源于 1986 年，并且一个非常流行，被称作「最佳探测器」。算法规定了边缘侦测中三个重要的标准，列举如下：

1. 边缘侦测要保证低错误率；
2. 良好的边缘定位 - 探测到的边缘和实际边缘像素差必须最小；
3. 图像的边缘只能被标记一次；

在使用「坎尼边缘探测算法」前，我们可以先将图像转为灰度格式，通常这样做可以让我们获得更好的结果。接下来，我们可以通过高斯模糊滤镜消除图像上的噪点，它需要一个向量作为高斯核大小的参数（which receives a parameter as a field - Gaussian kernel size）。再调用这两个方法处理过后，我们可以在坎尼边缘侦测时获得更好更准确的结果。

```javascript
im.convertGrayscale();
im.gaussianBlur([3, 3]);
```

![](https://img.zvz.im/imgs/2019/06/470560edbd5675be.jpg)

图像现已准备就绪，可以开始使用坎尼边缘侦测算法了。此算法接收两个参数：**lowThreshold** 和 **highThreshold** 。

这两个阈值帮你将所有像素划分为三组。

* 如果梯度像素（ gradient pixel ）的值比 **highThreshold** 高，则该像素被标记为强边缘像素（ strong edge pixels ）。
* 如果梯度像素的值介于高低阈值间，则该像素被标记为弱边缘像素（ weak edge pixels ）。
* 如果梯度像素的值比低阈值还低，那这些像素就完全被抑制。

并不存在一个针对所有图像都有效的通用阈值设定。**你需要针对每个图像设定合适的阈值**。不过存在一些方法可以预测合适的阈值，但是我不会在此文中具体说明。

在调用 Canny Edge 方法后，我们还会调用一次膨胀方法。

```javascript
const lowThresh = 0;
const highThresh = 150;
const iterations = 2;

img.canny(lowThresh, highThresh);
img.dilate(iterations);
```

经过以上步骤，我们就有了一张被分析过的图像。从这张图像上，我们可以选出所有的轮廓线，只需调用 **Matrix#findContours()** 方法，再将其写入到一个新的图像中。

> 译者注：此处代码根据 https://github.com/peterbraden/node-opencv/blob/master/examples/contours.js 进行了部分调整。

```javascript
const WHITE = [255, 255, 255];
let contours = img.findContours();
var all = new cv.Matrix(height, width);
all.drawAllContours(contours, WHITE);

all.save('./img/allContoursImg.jpg');
```

![](https://img.zvz.im/imgs/2019/06/07508d8ffe2c236e.jpg)

膨胀过后的图像

![](https://img.zvz.im/imgs/2019/06/246848b1793b0f98.jpg)

未进行膨胀的图像

在图片中我们可以看到所有通过坎尼边缘侦测到的轮廓。

如果我们只要选出最大的轮廓，可以使用下面的代码：循环每个轮廓，找出最大的那个。可以使用 **Matrix#drawContour** 方法将其画出来。

```javascript
const GREEN = [0, 255, 0];
let contours = img.contours();
let largestContourImg;
let largestArea = 0;
let largestAreaIndex;

largestContourImg = new cv.Matrix(height, width);

for (let i = 0; i < contours.size(); i ++) {
  if (contours.area(i) > largestArea) {
    largestArea = contours.area(i);
    largestAreaIndex = i;
  }
}

const lineType = 8;
const maxLevel = 0;
const thickness = 1;
largestContourImg.drawContour(contours, largestAreaIndex, GREEN, thickness, lineType, maxLevel, [0, 0]);
```

![](https://img.zvz.im/imgs/2019/06/df9ace1263c0476b.jpg)

如果想要多一些轮廓线，比如大于特定值的所有轮廓，我们只需要将 **Matrix#drawContour()** 方法挪到 for 循环中同时更改一下 **if** 条件即可。

```javascript
const GREEN = [0, 255, 0];
const lineType = 8;
const maxLevel = 0;
const thickness = 1;

let contours = img.contours();
let largestContourImg;
let largestArea = 500;

for ( let i = 0; i < contours.size(); i ++) {
  if (contours.area(i) > largestArea) {
    largestContourImg.drawContour(contours, i, GREEN, thickness, lineType, maxLevel, [0, 0]);
  }
}
```

![](https://img.zvz.im/imgs/2019/06/88e343ab79695efc.jpg)

### 多边形近似

多边形近似（ Polygon approximation ）可以用来做许多事情。最简单的就是使用 **Contours#boundingRect(index)** 方法在对象周围画上一个矩形框。此方法是属于 Contours 对象的。对图像进行坎尼边缘侦测后，调用 **Martix#findContours()** 方法就可以得到 Contours 对象（上一个例子中我们讲过的）。

```javascript
let bound = contours.boundingRect(largestAreaIndex);
largestContourImg.rectangle([bound.x, bound.y], [bound.width, bound.height], WHITE, 2);
```

![](https://img.zvz.im/imgs/2019/06/4f73351216c0607f.jpg)

第二种使用多边形近似的方法是通过调用 **Contours#approxPolyDP()** 方法获得指定精确的多边形。通过调用 **Contours#cornerCount(index)** 方法，可以获取到多边形顶角的数量。下面附上了两种使用不同级别精度获取到的图像。

```javascript
let poly;
let RED = [0, 0, 255];
let arcLength = contours.arcLength(largestAreaIndex, true);
contours.approxPolyDP(largestAreaIndex, arcLength * 0.05, true);
poly.drawContour(contours, largestAreaIndex, RED);

// number of corners
console.log(contours.cornerCount(largestAreaIndex));
```

![](https://img.zvz.im/imgs/2019/06/cb6d4e52603a38f1.jpg)

![](https://img.zvz.im/imgs/2019/06/36f55bf3133a2346.jpg)

使用 **Contours#minAreaRect()** 方法获取一个旋转过的面积最小的近似矩形也很有意思。

在项目中我使用它来判断一个特定的物件旋转到正确位置后的角度。下面一个例子，我们会对 **largestContourImg** 增加一个旋转过的矩形，并且打印出该矩形的旋转角度。

```javascript
let rect = contours.minAreaRect(largestAreaIndex);
for (let i = 0; i < 4; i ++) {
  largestContourImg.line([rect.points[i].x, rect.points[i].y], [rect.points[(i+1)%4].x, rect.points[(i+1)%4].y], RED, 3);
}

// angle of polygon
console.log(rect.angle)
```

![](https://img.zvz.im/imgs/2019/06/a65f66ec8a92d4c0.jpg)

### 无裁剪图片旋转

有一件事情是我需要解决，而 OpenCV 没有实现的。这件事就是旋转图像的同时，保证图像不会被裁剪掉。旋转图像是很简单的。

> 译者注：译者使用 opencv 3.2.0 版本进行测试，发现 opencv 的旋转功能不会发现本文描述的图片被裁剪的状况。

```javascript
img.rotate(90);
```

只是我们会得到如下的结果：

![](https://img.zvz.im/imgs/2019/06/69ffacfce4d67078.jpg)

**怎样才能做到既旋转了图片又不会被裁剪呢？**在旋转之前，我们先生成一个 8 位 3 通道的矩阵 **bgImg** ，其边长等于原图对角线的长度。

然后，我们计算出图像在新的 **bgImg** 矩阵中摆放的位置，使其能够被完整放入 **bgImg** 中。在 **bgImg** 中，我们使用计算出来的值调用 **Matrix#rotate(angle)** 方法。

```javascript
let rect = contours.minAreaRect(largestAreaIndex);
let diagonal = Math.round(Math.sqrt(Math.pow(im.size()[1], 2) + Math.pow(im.size()[0], 2)));
let bgImg = new cv.Matrix(diagonal, diagonal, cv.Constants.CV_8UC3, [255, 255, 255]);
let offsetX = (diagonal - im.size()[1]) / 2;
let offsetY = (diagonal - im.size()[0]) / 2;

IMG_ORIGINAL.copyTo(bgImg, offsetX, offsetY);
bgImg.rotate(rect.angle + 90);

bgImg.save('./img/rotatedImg.jpg');
```

![](https://img.zvz.im/imgs/2019/06/4010207470c71e31.png)

然后，我们可以对新的旋转图片使用坎尼边缘侦测方法。

```javascript
const GREEN = [0, 255, 0];;
let rotatedContour = new cv.Matrix(diagonal, diagonal);
bgImg.canny(lowThresh, highThresh);
bgImg.dilate(nIters);
let contours = bgImg.findContours();

for (let i = 0; i < contours.size(); i++) {
  if (contours.area(i) > largestArea) {
    largestArea = contours.area(i);
    largestAreaIndex = i;
  }
}

rotatedContour.drawContour(contours, largestAreaIndex, GREEN, thickness, lineType);
rotatedContour.save('./img/rotatedImgContour.jpg');
```

![](https://img.zvz.im/imgs/2019/06/0f82295326f2c5fd.png)

还有许多操作图片的方法。比如，很实用的去除背景功能 - 不过本文就暂不做介绍了。

### 物体侦测

实际上我工作时，我的程序会处理植物图像，并不会使用人脸，汽车或其它物件的识别器。

> 即便如此，我还是决定在本文中提一下面部识别，因为它更能够提现 OpenCV 的强大技术。

对于载入的图像，我们可以使用 **Matrix#detectObject()** 方法，该方法接受一个「级联分类器」（ cascade classifier ）路径作为参数。**OpenCV 自带一些预先训练好的分类器，可以用来侦测数字，面部，眼睛，耳朵，猫咪等等。**

```javascript
cv.readImage('./img/face.jpg', function(err, im){  
  if (err) throw err;
  if (im.width() < 1 || im.height() < 1) throw new Error('Image has no size');

  im.detectObject('./data/haarcascade_frontalface_alt2.xml', {}, function(err, faces){
    if (err) throw err;

    for (var i = 0; i < faces.length; i++){
      var face = faces[i];
      im.ellipse(face.x + face.width / 2, face.y + face.height / 2, face.width / 2, face.height / 2, [255, 255, 0], 3);
    }

    im.save('./img/face-detection.jpg');
    console.log('Image saved.');
  });
});
```

![](https://img.zvz.im/imgs/2019/06/2b7a99d9ea52c2d6.png)

> 译者注：译者找了个照片试了一下，发现会有误认的情况。。。不知道是否有方法提高准确度。

**本文中，我讲了一些 OpenCV 库在 Node.js 中的有趣特性。**虽然 OpenCV 没有官方的 Node.js 接口挺令人失望的，还好有这个 node-opencv 库，尽管有些功能没有实现，API 也不是很稳定，但是瑕不掩瑜。

如果你想要在工作中使用此类库，你应当研究一下 [node-opencv](https://github.com/peterbraden/node-opencv) 代码库中的 .cc 文件，因为此类库目前还没有一个完整的文档。

直接去读代码当然是最好的，我也很喜欢这样做；但是类库中有些返回值与 OpenCV 官方不一致和区别的地方，还是让我不那么爽。**希望这个类库能够快快发展，我也会尽我所能贡献一些代码。**

> 译者注：修改了边缘侦测部分的代码后，译者发现类库 API 与官方不一致的问题有所改善。

> 翻译自：[OpenCV tutorial: Computer vision with Node.js](https://community.risingstack.com/opencv-tutorial-computer-vision-with-node-js/)