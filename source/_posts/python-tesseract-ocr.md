---
title: python对图片初步处理并用tesseract-ocr识别简单数字内容
date: 2019-1-10
toc: true
mathjax: false
tags:
  - python
  - 图片处理
  - tesseract
categories: python
typora-root-url: ./
---

​    最近在做一个自动化登录系统的脚本，网站有简单的纯数字验证码，使用tesseract-ocr识别发现效果正确率不佳，在不进行额外的ocr训练的情况下，提高图片识别正确率。



#### 一、tesseract-ocr使用

如下图的图片，内容为简单的4位数字，直接使用tesseract-ocr对图片进行识别，默认参数无法识别出任何内容。

![captcha](/python-tesseract-ocr/captcha.jpg)

![1547098242517](/python-tesseract-ocr/1547098242517.png)

上述情况可能是程序默认的分页模式不匹配，如下图，该OCR可以指定多种不同分页模式。

<!-- more -->

![1547098606685](/python-tesseract-ocr/1547098606685.png)

使用模式13再次尝试：

![1547099248561](/python-tesseract-ocr/1547099248561.png)

已经能够识别出数字，但是内容还是不正确，4个数字被识别位5个数字。

> 小结：部分图片程序无法识别，可尝试更改默认分页模式。

#### 二、python利用pytesseract库调用tesseract-ocr

通过pip安装pytesseract库

> pip install pytesseract

```python
import pytesseract
from PIL import Image   # pytesseract调用需要入参Image.open()的对象
# 设置tesseract运行路径，如果已经添加到env:path中可省略
pytesseract.pytesseract.tesseract_cmd = r'C:\Program Files (x86)\Tesseract-OCR\tesseract.exe'
print(pytesseract.image_to_string(Image.open('captcha.jpg'), config='--psm 13'))

>>>
6921
```

> 注：直接通过cmd运行tesseract.exe和通过python调用识别得到结果不同，原因未知，可能是Pillow在open和save过程中内容有改变。

#### 三、对图像进行预处理，提高识别成功率

##### 3.1 对图片进行叠加处理

由于源图片像素明暗较明显，可以通过对图片进行自我叠加操作，使亮处更亮，暗处更暗。

像素点叠加算法是：

> 当基色<128时：结果色=混合色\*基色/128；当基色>128时：混合色=255-（255-混合色）\*（255-基色）/128

将图片转换为灰度图片后进行自我叠加，即每次基色和混合色相同。

```python
from PIL import Image
image = Image.open(filename)	# pillow载入图片
imgry = image.convert('L')		# 将图片转换为灰度图片

# 定义叠加处理函数
def pic_add(image):
    rows, cols = image.size		# 获取图片的像素行列数
    # 遍历所有像素点，逐像素点自我叠加
    for i in range(rows):
        for j in range(cols):
            pixel = image.getpixel((i,j))  # 获取每个像素点的数字
            if pixel <= 128:
                value = int(pixel * pixel)/128
            else:
                value = int(255-(255-pixel)*(255-pixel)/128)
            
            image.putpixel((i,j), value)
    return image

imgry = pic_add(pic_add(imgry))		# 叠加两次，多次叠加可修改叠加函数或多次调用
imgry.show()		# 查看叠加后的图片
```

![1547126736889](/python-tesseract-ocr/1547126736889.png)叠加后图片颜色黑白对比更加明显，但是中间白色噪点也更加突出。

##### 3.2 对叠加后图片进行一次均值滤波降噪

这里图片处理采取方式，将四周边缘视为无效值，直接置为白色；中间其他点取周围紧邻的8个像素点与自身像素指的平均值作为改点的像素值。

```python
def avg_nosie(image):
    rows, cols = image.size
    pos = []
    value = []
    for i in range(1, rows-1):
        for j in range(1, cols-1):
            pos.append((i,j))		# 记录坐标，存入队列
            val = 0
            for m in range(i-1,i+2):
                for n in range(j-1, j+2):
                    val += image.getpixel((m,n))
            value.append(int(val/9))	# 计算像素平均值，存入队列
    
    for i, j in zip(pos, value):
        image.putpixel((i,j))	# 根据记录的坐标队列和对应的像素值队列，重新写入图像像素数据
    # 将上下边缘像素置为白色
    for i in range(cols):
        image.putpixel((0,i), 255)
        image.putpixel((rows-1,i), 255)
    # 将左右边缘像素置为白色
    for i in range(rows):
        image.putpixel((i,0), 255)
        image.putpixel((i, cols-1), 255)
    return image
```
对比可见，均值处理后图像变得比较模糊，但是明显的白色噪点也得到缓解。
<table><tr>
    <td style="text-align:center">
        <img src=/python-tesseract-ocr/1547126736889.png title="滤波前">
        <span>滤波前</span>
    </td>
    <td style="text-align:center">
        <img src=/python-tesseract-ocr/1547127690279.png title="滤波后">
        <span>滤波后</span>
    </td>
</tr></table>

> 均值滤波采用线性的方法，平均整个窗口范围内的像素。本身存在固有缺陷，不能很好的保护图像细节，降噪的同时也破坏了图像细节，使图像变得模糊

##### 3.3 将图像二值化

简单的二值化算法，获取目标灰度图中，灰度值平均值为阈值，以一定的误差比例，将阈值周边的颜色置为白色，其他灰度置为黑色。

```python
from collections import defaultdict
# 获取灰度平均值
def get_threshold(image):
    # 生成一个dict对象，当调用dict的key不存在时，自动将该key的value设为0
    # 用来统计各像素值出现的次数
    pixel_dict = defaultdict(int)
    rows, cols = image.size
    for i in range(rows):
        for j in range(cols):
            pixel = image.getpixel((i, j))
            pixel_dict[pixel] += 1
    threshold = sum([k * v for k, v in pixel_dict.items()])/(rows * cols)
    return threshold

# 生成二值化模板
def get_bin_table(threshold,rate=0.3):
    table = []
    for i in range(256):
        # 像素值小于阈值置为黑色，大于等于的置为白色
        # 灰度平均值阈值比较粗略，允许一定的误差来处理图片结果，或者更换精度较好的阈值算法
        if i < threshold * (1 - rate):
            table.append(0)
        else:
            table.append(255)
    return table

# 将图片按照二值化模板转换
out = imaga.point(table,'1')
```

##### 3.4 消除图像中的噪点

在图片内遍历像素点（忽略边框），像素点周围白色点少于5个的，认为是噪点，置为黑色  

```python
def cut_noise(image):
    rows, cols = image.size
    change_pos = []
    for i in range(1, rows - 1):
        for j in range(1, cols -1):
        	pixel_set = []
        	for m in range(i - 1, i + 2):
            	for n in range(j - 1, j + 2):
                	if image.getpixel((m, n)) != 0:
                    	pixel_set.append(image.getpixel((m, n)))
            if len(pixel_set) <= 5:
                change_pos.append((i, j))
    for pos in change_pos:
        image.putpixel(pos, 0)
    return image

image = cut_noise(image)
text = pytesseract.image_to_string(image, config='--psm 13')
# 删除一些异常符号
exclude_char_list = ' .:\\|\'\"?![],()~@#$%^&*_+-={};<>/¥'
text1 = ''.join([x for x in text if x not in exclude_char_list])
print(text1)
```

使用处理后的图片调用tesseract，能正确识别。
