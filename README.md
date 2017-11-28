# OCR-
开源ocr框架进行framework编译后自己训练模型进行文字match


#Tesseract OCR使用介绍
[TOC]
##下载地址及介绍

* 官网介绍：<http://code.google.com/p/tesseract-ocr/wiki/TrainingTesseract3>
* Github源码连接： <https://github.com/tesseract-ocr>


##安装 Tesseract

* 语言包查看 <https://www.macports.org/ports.php?by=name&substr=tesseract->
* 支持Windows、linux、macOS

```
1、安装 tesseract和语言包
sudo port install tesseract	
sudo port install tesseract-<langcode>

2、homebrew 安装
brew install tesseract
brew install --with-training-tools tesseract

3、重新安装
brew uninstall tesseract
brew install --with-training-tools tesseract
```

* Homebrew 是一个包管理器，如果没装的话，在终端执行

```
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

##使用 Tesseract

* 使用命令行进行图像识别
* imagename 就是要识别的图片文件的名称，outputbase 就是识别结果输出文件的名称。
* lang 就是要识别的语言代码，例如英语为 eng、简体中文为 chi_sim 等等。可以同时识别多种语言，使用 “+” 相连，例如 eng+chi_sim。缺省时识别英语。

1、格式信息如下
```
tesseract imagename outputbase [-l lang] [-psm pagesegmode] [configfile...]
```
2、示例: 识别image图片并将结果保存在out.txt文件中
```
tesseract image.png out -l chi_sim
tesseract image.png out -l chi_sim -psm 10
```
3、pagesegmode 为识别的具体模式，具体包含以下模式：
```
•	0 = Orientation and script detection (OSD) only.
•	1 = Automatic page segmentation with OSD.
•	2 = Automatic page segmentation, but no OSD, or OCR
•	3 = Fully automatic page segmentation, but no OSD. (Default)
•	4 = Assume a single column of text of variable sizes.
•	5 = Assume a single uniform block of vertically aligned text.
•	6 = Assume a single uniform block of text.
•	7 = Treat the image as a single text line.
•	8 = Treat the image as a single word.
•	9 = Treat the image as a single word in a circle.
•	10 = Treat the image as a single character.
```

##训练样本

* 训练工具 <https://github.com/tesseract-ocr/tesseract/wiki/AddOns>
* 以jTessBoxEditor为例

```
> 1、收集文本信息的图片
> 2、制作的图片转为tiff格式
> 3、jTessBoxEditor进行tiff格式图片合成 <Tool->Merge TIFF>

合成后的图片取名规范 [lang].[fontname].exp[num].tif
[lang]是语言，[fontname]是字体，[num]是标号
```




###1、Make Box Files

* 使用 Tesseract 识别，生成 box 文件：
* 确保 tif 和 box 文件同名且位于同一目录下，用 jTessBoxEditor 打开 tif 文件），或者直接用文本编辑器编辑。

```
tesseract hz.font.exp0.tif hz.font.exp0 -l chi_sim -psm 10 batch.nochop makebox
```

###2、Run Tesseract for Training

* 使用修改正确后的 box 文件，对 Tesseract 进行训练，生成 .tr 文件：

```
tesseract hz.font.exp0.tif hz.font.exp0 -psm 10 nobatch box.train
```

###3、Compute the Character Set

* 生成字符集的文本

```
unicharset_extractor hz.font.exp0.box
```

###4、font_properties (new in 3.01)

* 定义字体特征文件，Tesseract-OCR 3.01 以上的版本在训练之前需要创建一个名称为 font_properties 的字体特征文件。font_properties 不含有 BOM 头，文件内容格式如下：

```
<fontname> <italic> <bold> <fixed> <serif> <fraktur>
```

* 其中 fontname 为字体名称，必须与 [lang].[fontname].exp[num].box 中的名称保持一致。<italic> 、<bold> 、<fixed> 、<serif>、<fraktur> 的取值为 1 或 0，表示字体是否具有这些属性。
* 这里就是普通字体，不倾斜不加粗，所以新建一个名为 font_properties 的文件，内容为： font 0 0 0 0 0

###5、Clustering

* 修改 Clustering 过程生成的 4 个文件（inttemp、pffmtable、normproto、shapetable）

```
shapeclustering -F font_properties -U unicharset hz.font.exp0.tr

mftraining -F font_properties -U unicharset -O hz.unicharset hz.font.exp0.tr

cntraining hz.font.exp0.tr
```

* 生成后的文件需要添加前缀， 如这里改为 hz.inttemp、hz.pffmtable、hz.normproto、hz.shapetable。

###6、Putting it all together

* 生成最后的训练文件

```
combine_tessdata hz.
```

###7、use example

* 使用训练的文件进行识别

```
tesseract test.png out -l hz
```

##脚本运行

```
#!/bin/sh
read -p "输入你语言:" lang
echo ${lang}
read -p "输入你的字体:" font
echo ${font}
echo "完整文件名为："
echo ${lang}.${font}.exp0.tif
echo "开始。。。"
echo ${font} 0 0 0 0 0 >font_properties
#tesseract  ${lang}.${font}.exp0.tif $(lang).$(font).exp0 -l chi_sim -psm 10 batch.nochop makebox
#read -p "继续生产tr文件？"
tesseract  ${lang}.${font}.exp0.tif ${lang}.${font}.exp0 -psm 10 nobatch box.train
unicharset_extractor ${lang}.${font}.exp0.box
shapeclustering -F font_properties -U unicharset ${lang}.${font}.exp0.tr
mftraining -F font_properties -U unicharset -O unicharset ${lang}.${font}.exp0.tr
cntraining ${lang}.${font}.exp0.tr
echo "开始重命名文件"
mv inttemp ${font}.inttemp
mv normproto ${font}.normproto
mv pffmtable ${font}.pffmtable
mv shapetable ${font}.shapetable
mv unicharset ${font}.unicharset
echo "生成最终文件"
combine_tessdata ${font}.
echo "完成"
```


