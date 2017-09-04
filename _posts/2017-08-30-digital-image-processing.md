---
layout: post
title: 图像预处理函数
excerpt: "本文介绍了常用的图像处理基本方法，着重于代码实现。主要基于OpenCV库，主要用C++和Matlab实现，函数功能分类参照《数字图像处理》。"
categories: [Digital Image Processing]
comments: true
---

## 1 灰度变换和空间滤波 
### 1.1 彩色图像转灰度图
* 代码 

{% highlight cpp %}
Mat img_gray;
img_gray = imread("1.jpg", 0);
{% endhighlight %}

{% highlight cpp %}
cv::Mat img_bgr, img_gray;
cv::cvtColor(img_bgr, img_gray, cv::COLOR_BGR2GRAY);
{% endhighlight %}

* 效果图

![img-binary](/img/201708-30-img-gray.jpg "img-binary" =320)


### 1.2 二值化
* 代码 

{% highlight cpp %}
cv::Mat img_gray, img_binary;
img_gray = imread(fileFullName, 0);		// 灰度图读入
cv::Scalar thd = cv::mean(img_gray);	// 阈值设置
cv::threshold(img_gray, img_binary, thd[0], 255, THRESH_BINARY);
{% endhighlight %}

* 效果图

![img-binary](/img/201708-30-img-binary.jpg =409x320)

## 2 形态学图像处理
### 2.1 腐蚀与膨胀
* 代码 

{% highlight cpp %}
cv::Mat img_binary, img_erode, img_dilate;
cv::Mat element = cv::getStructuringElement(cv::MORPH_CROSS, cv::Size(3, 3));	// MORPH_CROSS，滤波器设置，此处为+形状
cv::erode(img_binary, img_erode, element);	// 腐蚀
cv::erode(img_binary, img_dilate, element);	// 膨胀
{% endhighlight %}

* 效果图 

![img-erode](/img/201708-30-img-erode.jpg "img-erode" =409x320)


### 2.2 孔洞填充 
* 代码 

{% highlight cpp %}
// 漫水算法
cv::Mat img_binary;
cv::Mat ones = cv::Mat::zeros(img_binary.size(), img_binary.type());
for (int i = 0; i < img_binary.rows; i += img_binary.rows / 10) {	// 沿着左边线寻找背景区域（黑色），也可以绕一周
  floodFill(im_floodfill, cv::Point(0, i), Scalar(255));
  ones = (im_floodfill | ones);
}

// Invert floodfilled image
Mat im_floodfill_inv;
bitwise_not(ones, im_floodfill_inv);

// Combine the two images to get the foreground.
Mat im_out = (img_binary | im_floodfill_inv);	// 保留不同处
{% endhighlight %}

* 效果图 

![img-hole-filling](/img/201708-30-img-hole-filling.jpg "img-hole-filling" =409x320)


### 2.3 边界提取
* 代码 

{% highlight cpp %}
//// 最大连通域边界提取
cv::Mat img_binary;
vector<vector<cv::Point>> contours;

// 查找轮廓，对应连通域
cv::findContours(img_binary, contours, CV_RETR_EXTERNAL, CV_CHAIN_APPROX_NONE);

// 寻找最大连通域
if (contours.size() == 0) {
	// todo
}
double maxArea = 0;
size_t max_contour_ind = -1;
for (size_t i = 0; i < contours.size(); i++)
{
	double area = cv::contourArea(contours[i]);
	if (area > maxArea)
	{
	  maxArea = area;
	  max_contour_ind = i;
	}
}

// 将轮廓转为矩形框
cv::Rect maxRect = cv::boundingRect(contours[max_contour_ind]);
 
{% endhighlight %}

* 效果图 

![img-boundary](/img/201708-30-img-boundary.jpg "img-boundary" =409x320)


### 参考引用 
* [matlab连通域处理函数们](http://blog.csdn.net/abcjennifer/article/details/6672468)  

## 3 图像分割


