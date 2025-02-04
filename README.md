### 自动泊车下的路由决策工作
* 整个路由决策分为两步，第一步是一次规划，第二步是实时导航。
* 一次规划的具体工作：
![f144a7508dbaa2adfa68c813ca1179f](https://github.com/nieting1997/-/assets/90097659/ea5526db-1e05-49ce-832a-de2eb35855f0)
* 实时导航的具体工作：
![ef6fca5d060fbff7640c4ce2ccc404b](https://github.com/nieting1997/-/assets/90097659/28878260-db31-4fea-83b7-d2ca667ac157)

### 城区/高速下的实时导航工作
#### 拓扑场景
![602e232690da7fad15c9bebdb619e3b](https://github.com/nieting1997/-/assets/90097659/388dcdcb-4ab8-45c1-ae81-e108e6cffd66)
* 在行驶的过程中，路口内是没有road的。
  * 对应途中，road 10，road 50，road 60，road 70都是可以检测的
  * Road40，20，30无法检测。（非待转区）
* 实时导航需要根据路口内的路点生成对应的拓扑关系。
  * 拓扑算法采用贝塞尔曲线。
#### 导航场景
* 导航引导场景作用是告知下游应该往哪里换道，以及换道的迫切程度。
* 其中一个重要的概念是cost(代价)。通过对不同的道路评分，让下游知道去其他车道的意图有多强。
  * 两条车道之间的cost越大，说明导航意图就越强
  * 比如，前方要下匝道，则会给一个很大的差值，从而具备很强的向下匝道车道换道的意图。
* 导航信息来源有两个
  * SD导航(标精导航，来源于高德)。其主要组成有两个：
    * 导航路线。从起点到终点的一条线，实际上是一条条SD road连起来的。其中也会有道路等级，车道数，限速，曲率等信息
    * TBT事件信息。比如路口会提醒前方左转右转直行，可以走哪几根车道，车道的转向信息等等。TBT绑定在导航路线上。
  * RC导航(自构图)。基于海量车辆数据在云端构成的矢量图，应对复杂路口。其可以提供lane之间的连接关系，以及lane上的形点。
  * SD导航适用性强，代价小，优先使用SD导航。
* SD导航方法
  * TBT
    * mainAction 前方怎么走
    * laneInfo 每条lane的转向信息 backlane(每条车道的信息) frontlane(可走车道的信息)
    * EHP（电子地平线） 非导航路径的路线，比较关键的是岔口和分叉的角度
  * 具体步骤如下：
    * TBT的排序过滤，去掉超过距离阈值的信息。太远没有参考价值。
    * TBT是和车道严格绑定的，我们常常会遇到路口后三变四，三变二的场景。很远距离后，感知是无法检测道路的联通关系的。
    * 为了应对这种情况，使用了区域的概念。这里其实相当于将离散的通行信息，转化为了区域通行概率分布信息。
    * 道路一般都是3，4，5，6车道类型，取其最大公约数60。将整个道路进行60等分。
    * 将远处的TBT信息映射到现有的区域中。映射方式如图所示：
    ![4712a673603f9690dc838dc8448a7d1](https://github.com/nieting1997/-/assets/90097659/69171a5c-37a2-4f41-831e-3ce9ba671e14)
    * 但是这种0-1脉冲函数不太合理，远方的路口距离远，影响不会特别大。
    * 于是引入高斯模糊，让高斯模糊函数的方差随着距离的增大而增大，这样距离较远的TBT信息就能够减小其概率值，从而改变0-1脉冲函数（缩减成概率值）。
#### 红绿灯场景
![d15f62d629251d371f43a0a90ab7051](https://github.com/nieting1997/-/assets/90097659/91644352-02ec-4725-87c0-53c503cf83d6)
* 当红绿灯状态为红灯时，在路口前刹停，自车减速
* 当红绿灯状态为绿灯时，通过路口，自车不减速
* 当红绿灯状态从绿灯变为黄灯或者绿灯闪烁
  * 如果自车能在路口前以一个减速度刹车停止，则自车减速
  * 反之，通过该路口
为了方便下游决策，router的红绿灯模块需要给出**灯的颜色，类型以及倒计时**。每一帧按0.1s发送，每收到一帧倒计时-0.1s。
在自动驾驶中，参与路口控车的灯颜色有如下几种：
* 绿实灯
* 绿闪灯
* 黄实灯
* 红灯
* unknown灯
前四种灯可以通过地图+感知灯实实在在的看到，后一种灯应对遮挡、漏检等场景无法获知灯的类型。
![aa689f765b826cf9eb6c56fd590dfa0](https://github.com/nieting1997/-/assets/90097659/eb7b39b6-bf70-4468-a368-3092dc18c512)


当收到unknown灯时，需要推导红绿灯。

<img src="/Users/mac/Library/Application Support/typora-user-images/image-20240616125640169.png" alt="image-20240616125640169" style="zoom:75%;" />

