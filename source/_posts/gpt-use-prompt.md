---
title: chatgpt 使用过程中常用的 prompt
date: 2023-05-31 17:00:52
tags: 
- prompt
categories: ai 相关
---
## 前言
随着 AI 浪潮的兴起，人工智能将席卷一切，基本上所有的行业都要发生变革。 而在使用 AI 提高工作效率的过程中，prompt 变得越来越重要。

毫不夸张的说，一个好的 prompt 和一个普通的 prompt 对 chatgpt 的内容的准确性，可用性的输出将会产生巨大的差别。如果把 AI 比作 拉力赛的赛车手， 那么 prompt 就相当于坐在副驾驶坐的引导员。 没有 prompt 去指导方向，再牛逼的车，再牛逼的驾驶员都有可能迷失方向。

所以我针对我日常在 chatgpt 的使用中，一些常用的 prompt 有记录下来，以作备忘

### 1. 角色扮演
这个是 prompt 最常用的方式，就是让 chatgpt 来扮演一个角色，然后来问答

比如可以这样子写:
```text
你是一个呼吸内科的医生，你需要通过向我提出一些问题，并获得我的回答来判断我大概得的是什么类型的感冒，
如果我回答的信息不够，请继续向我询问，直到你得到充分的信息。你在最好还需要对我的病情给出专业的建议
```
<!--more-->
效果就是:

![](1.png)

### 2. 中文转英文
这个 prompt 也是很常见，因为有时候用 Google 翻译来翻译英文的话，还是略显生硬，不够地道，而 chatgpt 对英文的适用性是最强的，因此充当翻译其实是非常合适的。

比如可以这样子写
```text
帮我把后面的中文用地道的美式英语重新表达以下，要注意中文和英文的不同表达方式，不要直译 "{{Chinese}}"
```

上面的这个 <b>&#123;&#123;Chinese&#125;&#125;</b> 其实就是这个 prompt 要输入的变量，也就是我们要翻译的文案。效果就是

![](2.png)

### 3. 根据提供的资料来回答问题，或者总结
chatgpt 其实也非常适合做总结，这种也可以弄一个比较标准的带有变量的 prompt 的模式，比如以下这个:
```text
Context information is below.
----------------
{{请输入问题的参考资料}}
----------------
Using the provided context information, write a comprehensive reply to the given query.
Make sure to cite results using [number] notation after the reference.
If the provided context information refer to multiple subjects with the same name, write separate answers for each subject.
Guide customers to ask the right questions when the given context didn't provide enough information.
Answer the questions: {{你要询问的问题是什么?}}
Reply in chinese.
```

上面的 prompt 其实有两个变量，一个是 <b>&#123;&#123;请输入问题的参考资料&#125;&#125;</b> 表示问题的参考资料，一个是 <b>&#123;&#123;你要询问的问题是什么?&#125;&#125;</b> 表示要问的问题

效果如下

![](3.png)

### 4. 自问自答的上帝模式
有时候我们想知道一个知识点，但是又不想自己问，或者是认为自己问的会比较片面。 这时候就可以用这种 prompt 的方式，来让 chatgpt 自问自答， 开启上帝视角模式。

比如我下面这个:

```text
你的任务是在助手(A)和用户(U)之间切换。我会通过写A或U来指定您应该以助手成用户的身份回答或继续回答。作为用户，您尝试解决一个问题，作为助手，你尝试帮助用户。
您作为用户的任务是："{{你想了解的知识点?}}”。
现在并始以用户的身份提问
U:
```

上面的 prompt 有一个变量 <b>&#123;&#123;你想了解的知识点?&#125;&#125;</b>，就是换成你想问的问题即可。

这时候随着你的操纵，你就会发现真的是两个人在一起对话。而且真的可以聊的很嗨

效果如下:

![](4.png)

## 参考
关于更多 prompt 的使用可以看:
- [微軟官方親自出教學指南，介紹Prompt 工程中的一些進階玩法](https://www.techbang.com/posts/106279-microsoft-advanced-promp)


