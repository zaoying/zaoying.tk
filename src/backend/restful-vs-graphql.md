前段时间组内搞代码检视，经常能看到一些 “挂着 `RESTful` 羊头，卖的却是 `GraphQL` 狗肉”的 `API` 设计。

举个例子，假如后台有两种资源用户 `User` 和 群组 `Group` ，按照RESTful的规范，他们设计以下API端点：

```sh
# 获取用户列表
GET /users
# 获取指定用户
GET /user/{id}
# 创建用户
POST /users
# 修改用户
PUT /user/{id}
# 删除用户
DELETE /user/{id}

# 获取群组列表
GET /groups
# 获取指定群组
GET /group/{id}
# 创建群组
POST /groups
# 修改群组
PUT /group/{id}
# 解散群组
DELETE /group/{id}
```

咋一看没啥问题，可是到了后面，要实现 “用户加入群组” 和 “用户退出群组” 两个特性时，他们给出的API设计：

```sh
# 用户加入群组
PATCH /group/{id}
{
  "addMembers": ["userID"]
}

# 用户退出群组
PATCH /group/{id}
{
  "deleteMembers": ["userID"]
}
```

我当场就蚌埠住了，这哪里是 `RESTful` 风格，这不就是他们老大深痛恶绝的 `GraphQL` ？
眼看我就要当场发飙，有个老手赶紧出来圆场，他给出以下API设计：

```sh
# 用户加入群组
PATCH /user/{id}/group/{groupID}

# 用户退出群组
DELETE /user/{id}/group/{groupID}
```

这个设计看起来是解决刚才的问题，但实际上只是掩耳盗铃。
我接着追问： “管理员邀请用户加入群组” 和 “管理员将用户踢出群组” 要怎么设计呢？
那个老手依瓢画葫芦给出以下设计：

```sh
# 管理员邀请用户加入群组
PATCH /group/{id}/user/{userID}

# 管理员将用户踢出群组
DELETE /group/{id}/user/{userID}
```
眼看他们还不醒悟，我就接着追问：“用户加入群组需要管理员同意” 和 “管理员邀请用户加入群组需要用户同意” 又怎么实现呢？
这下子老手也没辙，只能说这块的代码需要重新设计。

我让他们把原来的设计文档重新拿出来检视，然后发现很多地方都不符合规范，连基本的实体关系图ER都没有！
我忍不住吐槽：怪不得当年我一个人拿着不到一半的工资，连前后端一起搞，效率却比这帮985/211的高材生/海归还要高，而他们连一个最基本的CURD都要不断地返工！
原来招人的HR完全不考虑专业是否对口，只看毕业院校和学历，结果这帮非科班出身的人连ER图都不会，更别说更复杂一点的类图、活动图以及泳道图，要知道对科班出身的人来说画UML是基本功！

其实他们的设计里面已经有了 `User` 和 `Group` 两张实体表，再增加一张 `Relation` 关系表：

```go
struct Relation {
    id string
    user_id string
    group_id string
    status string # 根据cookie的session获取当前用户id
    # 再判断当前用户是不是当前group的管理员
    # 如果是则管理员邀请用户，把status设置为"wait_for_user_accept"
    # 否则就是用户申请加入群组，把status设置为"wait_for_admin_approve"
}
```

然后这样设计API：

```sh
# 用户加入群组
POST /relations
{
  user: "abc",
  group: "ikun"
}

# 用户退出群组
DELETE /relations/{id}

# 管理员邀请用户加入群组
POST /relations
{
  user: "abc",
  group: "ikun"
}

# 管理员将用户踢出群组
DELETE /relations/{id}
```

回头总结，不难发现，大多数人在实践 `RESTful` 规范时，最容易犯的错误就是处理嵌套资源的时候，容易设计出 `/parent/xxx/sub/xxx` 这样的API。这种设计咋一看起来确实非常容易理解，但其实后期非常难以维护，尤其是遇到 `n:m` 多对多的关系时，这种嵌套的API设计就是一场灾难。


即便是 `1:n` 一对多的情况下，嵌套的API设计看起来没有问题，但后期当子资源 `n` 越来越大的时候，单独增加/删除某个子资源需要把所有资源都获取一遍，就非常容易形成性能瓶颈，并且没办法通过分库分表的方式进行横向扩容。


尽管我们可以换成 `GraphQL` 的风格，通过 `addSub` 和 `delSub` 的方式单独新增或删除某个子资源，来避免每次都需要获取所有的子资源。但我们也必须要考虑并发更新资源频繁引发冲突的风险；尤其越来越多的Java开发转向基于Go的云原生开发，为了方便维护不再额外引入SQL数据库，直接使用etcd，但etcd本身也有mvcc机制来保证数据的一致性，这样就会导致并发更新资源的冲突！


经过这次事情，我也彻底明白了，真正掌握RESTful规范，离不开对`关系模型`的深厚功底；尽管我们从SQL数据库切换到etcd这类NoSQL数据库，按道理API设计规范应该相应地换成GraphQL与`对象模型`紧密相连的规范。


但考虑团队水平参差不齐，后台API接口设计规范还是要坚持 `RESTful` 为主，先把基于关系数据库的基本功练好！
前端团队是可以基于 `GraphQL` 规范实现`Backend for Frontend`，搞一个接口聚合模块，作为性能优化手段，减少首页白屏时间。