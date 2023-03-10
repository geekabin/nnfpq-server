# 基于Gatewayworker的牛牛发牌器，服务端支持


## 如何运行
移步 https://www.workerman.net/doc/gateway-worker/

## 后台逻辑设计

### 登录逻辑

    要求每个人进入房间后写入一个名字，session存储

### 开房逻辑

    type 0 固定庄 1 轮庄


### 房间状态
    1 准备中
    2 游戏|结算中    

### 发牌逻辑

    1 庄家建房，所有人准备
    2 全部准备玩之后直接发牌
    3 庄家结算完（线下）准备进入下一轮，房间全部人返回准备状态

## 数据库设计

    貌似不需要

## 前端设计

    1 采用vant作为基础ui库
    2 页面（主界面，游戏界面）
    3 组件（登录组件，介绍组件，房间创建组件）

## 统计

    百度统计

## 关键逻辑处理

- ### 登录逻辑
    ```
    1 前端 type : login data中申明username
    2 反正来了就把 username放session中，反馈client_id和登录成功
    ```
- ### 创建房间的逻辑
    ```
    1 把房间信息存进 redis list中，包括房间名称，密码，做庄类型，房间的创建者名称，
       clientId，庄家名称 ，创建时候的时间戳，和散列值，反馈给用户成功后，用户跳转页面
       页面把hash值带进去，并且把hash值缓存到localstorage
    2 这个地方可能还存在用户多次创建房间的问题，用户退出后client_id会触发on_close
      所以当房间的所有client_id离线后，房间会自动释放，不会出现资源越积累越多的问题
    3 这个地方需要解决房间的唯一性问题，对于用户来讲，他只在乎自己创建的房间的名称
      但是我们不能让房间重复，需要增加其他的元素进去，比如时间戳，随机数，然后通过这
      些求一个hash值，把这个散列值反馈给用户，存储到房间信息中，作为房间的id，用户
      每次的动作都要带上这个散列值，因为一个用户可以在多个房间中
  
    ```
- ### 加入房间逻辑
    ```
    1 直接把client_id 加入对应的hash的房间，不用关注用户的姓名，因为其他的各种操作都
      通过client_id区分
    ```
- ### 准备逻辑
   ```
   1 不管是用户还是庄家，都需要做准备动作，当判断到房间内所有人都准备后，发送客户端，房间
    状态变成 1 ，游戏的round+1(实际上是推送依次房间信息)，也就是游戏进行状态，这里就不细分发牌，结算，因为结算在线下，无法控制
    然后进入发牌逻辑
   2 准备的边界不是房间，而是定位到某一局游戏，这个状态怎么存储？直接房间hash+round,存一个
     set,全部准备根据count来,到时候用户准备的时候，通过广播到客户端已经准备。
   3 看是一次性推过去所有数据，还是只推送单个人的准备数据，因为准备是比较频繁的操作，全部
     数据量要比单个的数据大房间人数倍，比如进入页面的时候拉全部，广播的时候只给单条
   ```
- ### 发牌逻辑
    ```
    1 找出房间的所有clientId，洗牌后直接每人取5张牌，一次性广播到房间所有人，判断房间的
     庄家规则，如果是轮庄，则需要判断庄家是否要变更，变更则重写房间redis，广播房间信息到
      房间的所有人
    ```

- ### 中途加入的数据拉取逻辑
    ```
    1 中途加入也是join的逻辑，这里不确定某个人退出后会不会及时走 onclose，不然可能一
    进一出就会有两个同名的人，但是client_id不同，当然庄家也是好处理的，直接两个都T掉就行
    2 进入的时候，都是走login和加入房间的逻辑，所以初始的数据都会给，按照事件依次推过去
    ```

## 其他棘手的问题

- ### 页面刷新和跳转的问题
  ```
    1 创建房间后，拿到了房间的hash,跳转页面后用户的client_id变化，导致，无法匹配到房间创始人
      和庄家数据
      解决方案：创建房间的页面只是收集数据，真正的创建跳转到游戏页面去创建，创建完成之后更新路由
    2 如果用户刷新怎么操作？
      解决方案：刷新就按照正常先退出在加入的方案做，如果退出的是房主或者庄家，则随机从房间内抽取
      一个玩家做房主和庄家
    3 我们是否不需要管房间是谁创建的，主要管庄家
      是
  ```
- ### 如何解决因为退出而造成的庄家选举问题
  ```
    1 我可以直接把房间解散，重新选举逻辑稍微复杂点，像QQ飞车这样的游戏就是重新选举的
  
    2 因为只是一个实验性项目，以简单为主，所以直接解散房间
  ```

## 鸣谢
  ```
  gatewaywork
  php
  ```