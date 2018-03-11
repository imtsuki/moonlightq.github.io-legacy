---
title: OpenCV 笔记（一）
date: 2018-03-11 18:30:10
categories:
- OpenCV
---

## 更好的矩形边缘检测算法

算法步骤如下：

1. 去噪

   使用 `medianBlur` 中值滤波算法对图像去噪。中值滤波以像素点邻域灰度值的中值来代替该像素点的灰度，能够较好的保留图像的边缘细节，有助于边缘的检测。

   `medianBlur` 算法的函数原型如下：

   ```c++
   CV_EXPORTS_W void medianBlur( InputArray src, OutputArray dst, int ksize );
   ```

2. 边缘检测

   `Canny` 边缘检测算法的函数原型如下：

   ```c++
   CV_EXPORTS_W void Canny( InputArray image, OutputArray edges,
                            double threshold1, double threshold2,
                            int apertureSize = 3, bool L2gradient = false );
   ```

   `Canny` 算法通过求解每一像素点附近的梯度幅度与方向来判断该点是否为边缘点。具体实现中，还拥有两个参数 `threshould1` 与 `threshould2`，作为为 `Canny` 算法的滞后阈值：

   1. 如果某一像素位置的幅值超过高阈值， 该像素被保留为边缘像素；
   2. 如果某一像素位置的幅值小于低阈值，该像素被排除；
   3. 如果某一像素的幅值在两个阈值之间，则只有当该像素与一个高于高阈值的像素相邻时，该像素被保留。

   通过测试，我们调整参数为 `threshould1 = 10, threshould2 = 20`。

3. 膨胀边缘

   `dilate` 算法是 `OpenCV` 的形态学算法之一，其可以对形状进行「膨胀」 。利用 `dilate` 算法，可以去除第二步中因局部噪点产生的孔洞，以防止下一步对轮廓探测的影响。

   `dilate` 算法的函数原型如下：

   ```c++
   CV_EXPORTS_W void dilate( InputArray src, OutputArray dst, InputArray kernel,
                             Point anchor = Point(-1,-1), int iterations = 1,
                             int borderType = BORDER_CONSTANT,
                             const Scalar& borderValue = morphologyDefaultBorderValue() );
   ```

4. 轮廓探测

   `findContour` 算法是 `OpenCV` 所提供的轮廓探测算法，其函数原型如下：

   ```c++
   CV_EXPORTS void findContours( InputOutputArray image, OutputArrayOfArrays contours,
                                 int mode, int method, Point offset = Point());
   ```

   参数 `mode` 定义轮廓的检索模式。有如下几个取值：

   + `CV_RETR_EXTERNAL` 只检测最外围轮廓；
   + `CV_RETR_LIST` 检测所有的轮廓，包括内围、外围轮廓，但是检测到的轮廓不建立等级关系；
   + `CV_RETR_CCOMP` 检测所有的轮廓，但所有轮廓只建立两个等级关系；
   + `CV_RETR_TREE` 检测所有轮廓，所有轮廓建立一个等级树结构。

   参数 `method` 定义轮廓的近似方法。有如下几个取值：

   + `CV_CHAIN_APPROX_NONE` 保存物体边界上所有连续的轮廓点到 `contours` 向量内;
   + `CV_CHAIN_APPROX_SIMPLE` 仅保存轮廓的拐点信息，把所有轮廓拐点处的点保存入 `contours` 向量内。

   根据实际目的，应选择 `CV_RETR_LIST` 与 `CV_CHAIN_APPROX_SIMPLE` 这两个取值。

5. 测试找到的轮廓

   `findContour` 所找到的轮廓包含各种各样的形状。我们需要对上一部中的每个轮廓进行测试，筛选出其中的矩形。

   1. 首先利用 `approxPolyDP` 对轮廓进行多边形拟合。这里利用 `arcLength` 函数计算每个轮廓的周长，来设置多边形拟合的精度；

   2. 对矩形进行判断。显然，一个矩形拥有四个顶点，且为一个凸包。这里利用 `isContourConvex` 来对多边形是否为凸包进行判断。同时，我们还利用 `contourArea` 计算多边形面积过滤掉了过小的多边形；

   3. 这里还对过于狭长的菱形进行了过滤：

      对于四边形 $ABCD$，当且仅当 $max\{\cos \angle A, \cos \angle B, \cos \angle C, \cos \angle D\}<0.3$，该四边形才会被添加到结果列表里。

最终具体实现代码如下：

```c++
void ImgOperate::detectSquares() {
    ::std::vector<::std::vector<::cv::Point>> _squares;
    ::std::vector<::std::vector<::cv::Point>> contours;
    ::cv::Mat blurred(main_image.size(), CV_8UC3);
    // 使用 ::cv::medianBlur 去噪
    ::cv::medianBlur(main_image, blurred, 9);
    ::cv::Mat gray0(blurred.size(), CV_8U), gray;
    // 对图像的每个通道进行一次识别
    for (int c = 0; c < 3; c++) {
        int ch[] = { c, 0 };
        ::cv::mixChannels(&blurred, 1, &gray0, 1, ch, 1);
        const int threshold_level = 2;
        for (int l = 0; l < threshold_level; l++) {
            if (l == 0) {
                // 边缘检测
                ::cv::Canny(gray0, gray, 10, 20, 3);
                // 膨胀边缘
                ::cv::dilate(gray, gray, ::cv::Mat(), ::cv::Point(-1, -1));
            } else {
                gray = gray0 >= (l + 1) * 255 / threshold_level;
            }
            // 轮廓探测
            ::cv::findContours(gray, contours, CV_RETR_LIST, CV_CHAIN_APPROX_SIMPLE);
            // 测试找到的轮廓
            ::std::vector<::cv::Point> approx;
            for (size_t i = 0; i < contours.size(); i++) {
                // 多边形拟合
                ::cv::approxPolyDP(::cv::Mat(contours[i]), approx, ::cv::arcLength(::cv::Mat(contours[i]), true)*0.02, true);
                // 矩形判断基本条件
                if (approx.size() == 4 &&
                    fabs(::cv::contourArea(::cv::Mat(approx))) > 1000 &&
                    ::cv::isContourConvex(::cv::Mat(approx))) {
                    // 过滤狭长菱形
                    double maxCosine = 0;
                    for (int j = 2; j < 5; j++) {
                        double cosine = fabs(angle(approx[j % 4], approx[j - 2], approx[j - 1]));
                        maxCosine = MAX(maxCosine, cosine);
                    }

                    if (maxCosine < 0.3)
                        _squares.push_back(approx);
                }
            }
        }
    }
    /*Some Other Code*/
}
```
## 可拖拽框线的改进

为了提高应用程序的可操作性，我们对可拖拽框线进行了改进。

首先是对边框的拖拽操作。

对于一条边，我们可写出两点式方程：$(y_1 - y_2)x + (x_2 - x_1)y = x_1y_2 - x_2y_1$；

令 $A = y_1 - y_2,B=x_2-x_1,C=x_1y_2 - x_2y_1$，当鼠标在坐标 $(x, y)$ 处按下时，可计算出点到边线的距离 $d = |Ax + By + C| / \sqrt{A^2 + B^2}$；

经过测试，当 $d < 6$ 时，我们可认为用户将要进行拖拽边框的操作。

注意到，当 `mouseMoveEvent` 事件发生时，我们并不知道上一刻鼠标的原坐标；但幸运的是，由于我们可以认为拖拽时鼠标一直在边框上，我们便可以通过还未移动（此时）的边框顶点坐标，计算出上一时刻鼠标的原坐标，从而与事件内包含的此时坐标相减，计算出坐标差，对边框顶点的位置进行更新。

于是，我们要在 `mousePressEvent` 事件发生时，计算出鼠标单击位置对于这条框线的比例 $\lambda$。从数学上来说，这个比例 $\lambda = (y_1 - y)/(y_1-y_2)=(x_1 - x)/(x_1 - x_2)$，两个式子均可计算；然而，由于浮点数的精确性所限，我们必须对直线的斜率进行判断，动态地选择两个式子；否则，当直线平行或接近平行于 $x$ 轴时，利用第一个式子计算会产生巨大的误差（反之亦然）。

在当鼠标既没有落在顶点，也没有落在边框上时，我们便可对整个框线进行平移操作。

最后是边框线的自动隐藏。在 `DragableLabel` 类中添加一个 `QTimer` 类型的成员变量，当没有 `mouseMoveEvent` 事件发生时开始计时，达到阈值时自动隐藏边框线，以提高用户对处理结果的观赏体验。

这里需要注意，Qt 默认只在鼠标按下时发出 `mouseMove` 信号。在 `DragableLabel` 类的构造函数中添加如下代码：

```c++
setMouseTracking(true);
```

这样在没有按下鼠标时，也可正常产生 `mouseMoveEvent` 事件。