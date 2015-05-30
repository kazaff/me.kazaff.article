最近为前端团队在开发时如何尽可能的不依赖后端服务的开发进度进行了讨论，觉得既然是一大波js工程师，那么理应用nodejs来伪造后端服务，从此走向自给自足的小康之路！

在linux下，这个需求可以使用成熟的扩展来完成：[forever](http://blog.fens.me/nodejs-server-forever/)，[pm2](http://cnodejs.org/topic/51f8c15144e76d216a588fcc)，可能还有其它~~

但是在windows平台下，一开始我确实没找到可行的现有方案，虽然后来同事找到了一个：[supervisor](http://www.cnblogs.com/pigtail/archive/2013/01/08/2851056.html)

其实nodejs提供的库中已经有非常完备的api来解决这个问题，所以简单的拼组了一下，完成了一个简单的模块：[RestMock](https://github.com/kazaff/restMock)~

PS：
昨天工作机升级了，颇有种鸟枪换炮的赶脚，一下子双屏还真有一些不适应，哟呵呵呵呵呵呵~~

