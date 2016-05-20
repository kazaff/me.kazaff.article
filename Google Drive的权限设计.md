关于google drive的权限设计，一开始看官方文档，搞的云里雾里的！

可能是对google drive不了解的关系，再加上没有参与设计过类似的平台化系统，所以对于它的权限设计内容，总觉得很绕~

停下来花了点时间仔细捋了一遍，发现这个设计还是非常优雅的，至于通用性，我就持保留态度了！

首先，我们来列出整套权限系统设计到的元素有哪几个：

* account（帐号）
  * user
  * user group
  * domain
  * anyone
* resource（资源）
  * file
  * folder
* permission（权限设置）
  * role
  * type

咱们简单的就这么先罗列出来。

首先我们先看一下三个主要的元素的关系：

```
resource  ------>  permission  ------> account [ ------>  resource]
          1     n              1     1           n     n
```        
解释：

* 一个资源对应多个权限设置
* 一个资源的metadata中还会包含多个特定帐号（owners, sharingUser, lastModifyingUser）
* 一个权限设置绑定到多个帐号（根据type来确定和帐号的绑定类型，根据domain和emailAddress来确定具体哪个或哪些帐号）
* 一个权限设置会预先设定好几个拥有不同操作范围的角色（role），具体关系可以看[官方文档](https://developers.google.com/drive/v3/web/manage-sharing#roles)
* 最终，帐号就会和资源关联上，就是这么绕！

现在应该已经解释的很清楚了吧？上面的关系图中最后的account与resource的关系其实是根据前面的关系推出来的，而不是直接可以获取到的（除了在resource的metadata中的那几个特定帐号字段，个人猜测之所有会有这几个metadata，是为了性能）。

不能确定这么设计是好还是坏，但整体来看，其实并不算复杂。
