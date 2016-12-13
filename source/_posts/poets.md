title: python爬虫隔一段时间一乐之海子的诗
date: 2016-12-08 23:33:46
tags: python, 生活
categories: 生活

---

![](http://7teb9r.com1.z0.glb.clouddn.com/haizi.png)

每隔一段时间(一周到一个月)拿出1到2天来做一个好玩的东西，不求回报，只为快感。
前两天刚买了一本电子书《海子的诗》，晚上读了快一半，好多诗里面都提及了麦子和村庄。想到可以对海子的所有的诗来个词频分析，顺便做一个词云图片。


用到了python的图片处理PIL，绘图模块matplotlib，科学计算numpy，还有中文分词jieba，词云模块wordcloud。很多代码都是从网上或者wordcloud示例程序中摘抄过来的。在做这个的过程中发现了一篇相关内容非常不错的博客，强烈推荐：http://minimaxir.com/2016/05/wordclouds/

直接贴代码吧，前提是需要把海子的诗保存到txt中

``` python
# -*- coding: utf-8 -*-

import jieba
import numpy as np
import matplotlib.pyplot as plt

from os import path
from collections import Counter
from PIL import Image

from wordcloud import WordCloud, STOPWORDS, ImageColorGenerator

d = path.dirname(__file__)

with open('haizi.txt', 'r') as poet:
    s = poet.read()

seg_list = [x for x in jieba.cut(s) if len(x) > 1 and x not in [u'一个', u'一只', u'一样', u'一直', u'一种']]

alice_coloring = np.array(Image.open(path.join(d, "haizi.jpg")))

font = "/Library/Fonts/Lantinghei.ttc"
wc = WordCloud(background_color="white", font_path=font, mask=alice_coloring, max_font_size=80, random_state=42, scale=1.5)

# generate word cloud
wc.fit_words(Counter(seg_list).items())

# create coloring from image
image_colors = ImageColorGenerator(alice_coloring)


# show
plt.imshow(wc)
plt.axis("off")
plt.figure()

# recolor wordcloud and show
# we could also give color_func=image_colors directly in the constructor
plt.imshow(wc.recolor(color_func=image_colors))
plt.axis("off")
plt.figure()
plt.imshow(alice_coloring, cmap=plt.cm.gray)
plt.axis("off")
plt.show()

# store to file
wc.to_file(path.join(d, 'haizi.png'))
```

下面这个代码是爬虫的代码，最主要的还是中文乱码处理，从 http://www.eywedu.com/haizi/ 上面爬下来了海子的大部分诗，没有全部爬下来，代码里只对下一页进行了爬取，后来发现有的长诗里面还有目录，本来以为麦子应该占很大的比重，生成了图片才发现没有麦子这个词的踪迹。

中间花费了很大部分的时间来处理中文乱码问题，历史遗留的ASP网站果然不行，http返回头里都不带content-type字段。

``` python
# -*- coding: utf-8 -*-

from bs4 import BeautifulSoup
import requests
import re


def parse_poet(html, encoding):
    soup = BeautifulSoup(html, from_encoding=encoding)
    title = soup.find("table", attrs={"width": "95%", "border": "0", "align": "center"}).text
    text = re.sub('\n[ \t]+', '\n', soup.find("blockquote").text)
    hrefs = soup.find("p", attrs={"align": "right"}).find_all('a')
    next_page = None
    if len(hrefs) == 3:
        next_page = hrefs[-1].get('href')
    return title, text, next_page


url = "http://www.eywedu.com/haizi/01/001.htm"

with open('haizi.txt', 'a') as mfile:
    while url is not None:
        r = requests.get(url)
        if r.encoding == 'ISO-8859-1':
            encodings = requests.utils.get_encodings_from_content(r.content)
            if encodings:
                r.encoding = encodings[0]
            else:
                r.encoding = r.apparent_encoding

        poet = parse_poet(r.text, r.encoding)
        mfile.write(poet[0].encode('utf8'))
        mfile.write(poet[1].encode('utf8'))
        url = poet[2] and '/'.join(url.split('/')[0:-1]) + '/' + poet[2] or None

```

现在可以去想想下一次要搞个什么好玩的东西了，不出意外还会是基于爬虫。
