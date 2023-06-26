---
title: 使用chatgpt编写小说
date: 2023-06-26 06:42:05
tags:
- chatgpt
---
自己喂提示词给chatgpt，总是感觉效果不理想，没有逻辑，今天看到一个项目，可以使用chatgpt编写小说，效果还不错，可以参考一下。
<!--more-->
## 项目地址
https://github.com/mshumer/gpt-author
## 使用方法
由于作者提供的代码是直接使用gpt4.0的，没有适配gpt3.5，所以在issue里有人提供了适配gpt3.5的代码，另外添加了多语言翻译。
issue: https://github.com/mshumer/gpt-author/issues/1#issuecomment-1602105017
进入colab后，点击标题栏`复制到云端硬盘`，将代码保存一份避免丢失。

在第一段代码中填写openai.api_key和stability_api_key，openai用于生成小说内容，stability用于生成封面。
```python
!pip install openai --quiet
!pip install EbookLib --quiet

import openai
import os
from ebooklib import epub
import base64
import os
import requests


openai.api_key = "ENTER OPENAI KEY HERE"       # get it at https://platform.openai.com/
stability_api_key = "ENTER STABILITY KEY HERE" # get it at https://beta.dreamstudio.ai/
```

代码下方有一个`setting`面板，修改model（默认gpt-3.5-turbo-16k），num_chapters（想生成多少章的小说）...
其中destLanguage是翻译语言，可以设置为`zh-cn`，这样生成的小说就是中文的了。
更多语言请执行第3段代码，查看支持的语言。
```python
# To find the language code to use in the cell form above.
for key, value in LANGUAGES.items():
    print(f"Code: {key}\t Language: {value}")
```
我这里测试gpt3.5，生成18章小说就会报错：`InvalidRequestError: This model's maximum context length is 8192 tokens`
这是我的设置：


myModel:
gpt-3.5-turbo-16k
num_chapters:
18
novelStyle:
写实主义和现实主义的结合
author:
GPT-Author
prompt:
一个普通的上班族，因为一次意外事件而失去了工作，他决定改变自己的生活，追求自己真正的梦想，并在困境中找到了新的机会和意义。
writing_style:
简洁明快，语言流畅自然。运用恰如其分的描写和对话，使得故事情节紧凑而引人入胜。叙述方式富有情感，能够深入人心，让读者对故事中的人物和事件产生共鸣，在小说中巧妙地融入一些哲学思考和社会观察。通过人物的内心独白和对社会现象的揭示，探讨了个人命运与社会环境的关系，引发读者对生活意义和社会问题的思考
destLanguage:
zh-cn

## 问题解决
执行中可能会出现以下问题：
### 1. 报错SyntaxError: invalid character '：' (U+FF1A)
```
step cost: 0.004595
step cost: 0.00587
step cost: 0.006504
step cost: 0.0024980000000000002
Generating storyline with chapters and high-level details...
step cost: 0.005292
step cost: 0.009691
Traceback (most recent call last):

  File "/usr/local/lib/python3.10/dist-packages/IPython/core/interactiveshell.py", line 3553, in run_code
    exec(code_obj, self.user_global_ns, self.user_ns)

  File "<ipython-input-19-ce40d1d73907>", line 1, in <cell line: 1>
    novel, title, chapters, chapter_titles = write_fantasy_novel(prompt, num_chapters, writing_style)

  File "<ipython-input-18-0c9dea3b0c8c>", line 168, in write_fantasy_novel
    chapter_titles = ast.literal_eval(storyline)

  File "/usr/lib/python3.10/ast.py", line 64, in literal_eval
    node_or_string = parse(node_or_string.lstrip(" \t"), mode='eval')

  File "/usr/lib/python3.10/ast.py", line 50, in parse
    return compile(source, filename, mode, flags,

  File "<unknown>", line 1
    这是我重写后的故事情节：
               ^
SyntaxError: invalid character '：' (U+FF1A)
在generate_storyline的prompt内添加`Just provide the code without replying`
```
### 2. This model's maximum context length is 16385 tokens. However, your messages resulted in 16407 tokens. Please reduce the length of the messages
调小num_chapters，或者换个model

