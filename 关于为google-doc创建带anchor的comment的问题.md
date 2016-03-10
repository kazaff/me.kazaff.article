这个问题，如果你跟我一样只从google提供的[官方文档](https://developers.google.com/drive/v3/web/manage-comments#anchoring_comments)上理解的话，那么你肯定是无法明白这篇文章的问题指的是啥？

使用官方提供的接口调用工具，你无论如何都无法给你google drive中指定的google document创建带有anchor的comment的！

为嘛？

官方不是说的很清楚么，是不是你拼接的json字符串有问题？或者，你的region定义的有问题？或者是不是你用错了region classifier？

我只想说：呵呵~

再查看了不少资料后，发现其实很多人都吐槽这个细节，官方对anchor的注意事项确实解释的很清楚，但却没有讲清楚一个问题：

**到底google doc是不是plain-text类型的文件？？**

你如果像我一样把它当成是普通的plain-text，你就也会想我一样浪费太久的时间在这个问题上！

目前能查到的资料上都解释说，到目前为止你还是无法通过api实现对google doc添加绑定anchor的comment，所以，死心吧！
