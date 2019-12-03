# 背景介绍
本项目是CCF竞赛[[1]](#ref)的参赛项目，拟解决身份证复印件识别的问题，在复赛测试集上达到95%的识别准确率。
竞赛提供的数据集为生成数据，与真实样本存在差异。我们的方案针对竞赛数据设计实现，故依赖许多强假设。
应用场景发生变化时，需要对方案进行相应调整。

# 数据特点
图像背景干净无噪声，每幅图中包含一张身份证的正面和反面。
身份证的大小、形状始终不变，为长约445像素、宽约280像素的矩形，左上角标有“仅供BDCI比赛使用”字样。
身份证上叠加的半透明水印会遮挡文字或边缘，水印方向与身份证的方向始终一致。图像存在不同程度的模糊。
![avatar](https://github.com/hzli-ucas/CCF-OCR/blob/master/images/00df9505b7e647d8b936fd4bf939afdd.jpg)

# 方案简介
我们将要素提取过程划分为身份证定位、文本行定位、文本识别三个步骤，在main.py中统一调用。

身份证定位对应location_card.py文件，其中的getCardVertexes函数以测试图像为输入，返回身份证的顶点坐标。
文本行定位对应location_text.py文件，其中的getTextFromCard函数以原图和身份证顶点为输入，返回待识别的文本行图像。
文本识别对应recognition.py文件，其中的readTextImages函数以文本行图像为输入，返回识别结果。
各部分的实现思路如下。

## 身份证定位
（1）由于背景干净而身份证底纹丰富，采用边缘提取获取身份证底纹；闭操作将底纹连接成完整区域。
（2）根据面积滤除无效轮廓，获得身份证外边缘；霍夫变换检测直线，根据夹角及间距，获取候选矩形。
（3）将候选矩形与身份证外边缘匹配，将与边缘线重合度最高的两个矩形取为身份证外框，返回顶点。

## 文本行定位
（1）根据身份证顶点将原图投影变换，获取固定大小的身份证图像（445x280）。
（2）将“仅供BDCI比赛使用”字样与图像的左上角和右下角进行匹配，判断图像是否倒置并旋转为正向。
（3）采用模板匹配定位水印位置，对水印区域像素的灰度值线性变换以去除水印。
（4）两幅图像与身份证正反面模板交叉匹配，根据匹配结果区分正反面并矫正位置偏差，随后按照对应位置切割得到文本行图像。

这里所采用的模板匹配，是将目标图像与模板图像进行卷积，最高响应值即为两者的匹配程度，最高响应值的坐标即为目标图像相对于模板的位置偏差。
我们采用拉普拉斯图像进行匹配，因其与原图灰度无关且保留了丰富的形状信息。模板图像是从训练集取相应图像，计算多幅（500-1000）图像的均值得到。

## 文本识别
（1）采用CNN+CTC进行文本行识别，模型是在生成数据集上预训练、真实文本图像上微调得到，数字和汉字识别采用不同的模型。
（2）对于特定项（性别、民族、日期），采用基于词典的Beam Search方法解码网络输出。
（3）部分存在对应关系的项（比如出生日期和身份证号），取高置信度的识别结果对低信度结果进行修正。

其中Beam Search的部分代码参照了[[2]](#ref)。

<div id='ref' />

# 参考资料
[1] 2019年CCF大数据与计算智能大赛（CCF Big Data & Computing Intelligence Contest，简称CCF BDCI），[“基于OCR的身份证要素提取”赛题](https://www.datafountain.cn/competitions/346)
  
[2] [githubharald/CTCWordBeamSearch](https://github.com/githubharald/CTCWordBeamSearch)
