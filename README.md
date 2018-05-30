# Rank
一个基于Bitcoin Cash的内容挖掘协议

## 目的

基于BCH区块链创建一个类似[豆瓣](https://www.douban.com)的内容索引和发掘系统。



## 预备知识

下文对协议进行描述，目前我们可以把应用场景具体为一个电影评价系统。用户可以向系统内添加电影，可以对电影打分，可以对电影进行评论，可以对电影的评论再进行评论，评论可以一直嵌套下去。

### 基本概念

1. 账户：用户的标识，该用户的所有动作都可以被识别为同一用户所为。
2. 内容：电影、电影的评论、评论的评论等用户产生的内容。
3. 评论：对某一内容的文字性评价。
4. 评分：对特定内容的数字量化性评价，被评价的内容可以是电影，但不能是评论。
5. 点赞：对特定内容的定性评价。评论可以被点赞，电影不能被点赞。

### 激励模式

1. 对应用开发者的激励

   用户增加电影时，会给应用开发者掌握私钥的“系统地址”发送一笔小额转账，以标记自己增加了这部电影。大量电影会产生大量转账，应用开发者会因此受益。这也是应用最基本的商业模式。

2. 对内容的激励

   为了激励用户创造内容，一个内容被某个用户添加后，那么这个内容归该用户所有。其他用户对该内容的评价，都需要给该用户一笔BCH转账，评价内容放在转账里，才算是协议认可的合法评价。通过这种方式，激励内容创建者创建更多内容。

3. 对非内容的激励

   因为评分和点赞不是内容，所以不能得到上述的内容激励，但评分和点赞对于内容发掘是非常重要的，所以需要有激励方式对非内容进行激励。

   当用户对某个内容进行评分或点赞时，需要按照特定规则给之前一个进行评分或点赞的用户转一笔BCH转账，才算是协议认可的合法评分或点赞。

## 协议文本

### 基本概念

1. 系统地址：一个BCH地址，用于用户增加电影或进行个人设置时转账使用。
2. 账户：一个BCH地址就代表一个账户。
3. 协议码：每一条协议都对应一个N字节数字，用来标识是什么协议
4. 协议参数：每一条协议都会携带N个参数，用来完整地执行这个协议
5. 转账金额：协议中所有的转账，只要能被打包，任意金额均可。通常使用网络接受的最小转账金额。
6. 协议tx的input： tx的input可以有多个，但必须属于同一个地址，也就是账户地址。
7. 协议tx的找零地址：tx的找零地址必须与input对应的地址相同，也就是账户地址。下面协议中的`目的地址`省略了找零地址。

### 协议

1. 增加电影

   | 源地址   | 目的地址 | OP_RETURN                          |
   | -------- | -------- | ---------------------------------- |
   | 用户地址 | 系统地址 | 协议码+来源(douban或imdb)+电影编号 |

   * 利用外部数据

     豆瓣和IMDB最为全球权威的电影数据库，可以作为数据来源。一方面可以有效标示一部电影，另一方面，也可以借助它们的API来获取影片相关内容，如简介、电影图片等，这些信息可以让应用的表现力更丰富。

   * 重复性处理

     如果有电影被重复添加，那么先打包的视为有效，在同一个块被打包的，先出现在块中的被视为有效。

2. 增加电影评论

   | 原地址     | 目的地址       | OP_RETURN                            |
   | ---------- | -------------- | ------------------------------------ |
   | 评论者地址 | 电影增加者地址 | 协议码+增加该电影的txid+评论文本内容 |

   * `电影增加者地址`与`增加该电影的txid`中的input地址一致时，消息才合法。

3. 增加评论的评论

   | 原地址     | 目的地址     | OP_RETURN                            |
   | ---------- | ------------ | ------------------------------------ |
   | 评论者地址 | 被评论者地址 | 协议码+增加该评论的txid+评论文本内容 |

   * `被评论者地址`与`增加该评论的txid`中的input地址一致时，消息才合法。

4. 评分

   | 原地址     | 目的地址         | OP_RETURN                    |
   | ---------- | ---------------- | ---------------------------- |
   | 评分者地址 | 电影创建者地址   | 协议码+增加该电影的txid+分数 |
   |            | 另一评分者的地址 |                              |

   * `电影创建者地址`与`增加该电影的txid`中的input地址一致时，消息才合法。

   * 另一评价者的选择

     * 如果该评分是该内容的第一个(没有其他评分tx被打包，内存池里存在的不算)评分，那么`目的地址`不需要`另一评价者的地址`
     * 如果不是，则按照如下规则选择`另一评分者`
       * 如果有评分相同的(只计算被打包的tx，内存池的不算)，则从所有评分相同的评分tx中随机选择一个，该tx中的发送方地址作为`另一评价者的地址`
       * 否则，从评分最接近的多个里面(只计算被打包的tx，内存池的不算)随机算则一个评分tx，该tx中的发送方地址作为`另一评价者的地址`

     如果评分不符合上述原则，则为无效评分

   * 修改评分

     一个用户可以对同一内容进行多次评分，但以最后一次评分为准。

5. 点赞

   | 原地址     | 目的地址         | OP_RETURN           |
   | ---------- | ---------------- | ------------------- |
   | 发起人地址 | 内容创建者地址   | 协议码+内容创建txid |
   |            | 另一喜欢者的地址 |                     |

   * `内容创建者地址`与`内容创建的txid`中的input地址一致时，消息才合法。

   * 另一个点赞者的选择

     * 如果该点赞是该内容的第一个（没有其他点赞被打包，内存池里存在的不算），那么`目的地址`不需要`另一点赞者的地址`。
     * 否则，从点赞的tx里面随机选择一个，该tx中的发送方地址作为`另一点赞者的地址`

     如果不符合上述规则，则点赞操作无效

6. 转移电影所有权（待设计）

7. 增加账户昵称（待设计）

8. 增加账户描述（待设计）
