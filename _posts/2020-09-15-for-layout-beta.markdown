---
layout: post
title:  "WHan's first blog for demo"
date:   2020-09-15 21:25:56 +0800
categories: word
---

* 流程
  * 校准
  * 射频干扰消除
  * 频率参考系转换
  * Gridding（计算最密集） 不规则->规则
  * 信号提取

* HyGrid
  * 限制卷积的数据查找空间和计算量
  * 第一部分利用多核CPU对采样数据进行重排，并基于有序数据设计了两级查找表的构建方法，从而快速定位需要卷积的数据并且减少计算量;第二部分在GPU上加速卷积计算本身，采用了线程组织配置、寄存器和纹理内存、线程粗化等并
    行优化策略来提升存储器访问的效率以及设备的利用率
  * 数据类型（三维）
    * 赤经
    * 赤纬
    * 频率或谱线速度
    * 最终需要在赤经、赤纬两个空间方向上具备均匀的间隔（Gridding）
  * 射电数据处理流程通常就需要进行Gridding将原始数据从不规则的采样空间转换到间隔均匀的规则网格空间上

* Gridding
  * 基于卷积（通用）
  * ![image-20200730001250590](/Users/ansakai/Library/Application Support/typora-user-images/image-20200730001250590.png)
  * 实现方法
    * 散发法
      * ![image-20200730112806314](/Users/ansakai/Library/Application Support/typora-user-images/image-20200730112806314.png)
    * 聚合法
      * ![image-20200730112758515](/Users/ansakai/Library/Application Support/typora-user-images/image-20200730112758515.png)
      * 优点
        * 不会遭受写竞争的毒害，因为一个输出网格点的所有更新操作永远都只由单个线程执行
      * 缺点（所以需要预排序）
        * 输入采样点固有的不规则性，使得输出网格点无法预测贡献点在输入数据的具体位置，导致大量的对卷积核外采样点的无用访问

* HyGrid

  * Gridding
    * 预排序
    * GPU加速
      * ![image-20200730164235570](/Users/ansakai/Library/Application Support/typora-user-images/image-20200730164235570.png)
    * 实现HyGrid算法

* Gridding

  * 用途：通过Gridding操作将原始数据从不规则的采样空间转换到规则的网格空间

  * 原理：利用望远镜的波束形态，根据所需的目标均匀各点的位置，对每个空间点、每个频率上的观测数据同与望远镜波束形态有关的函数作卷积

  * 卷积

    * ![image-20200730173631953](/Users/ansakai/Library/Application Support/typora-user-images/image-20200730173631953.png)

    * ![image-20200730173840393](/Users/ansakai/Library/Application Support/typora-user-images/image-20200730173840393.png)

  * 并行方法

    * 散发法（写竞争）
    * 给予查找表的聚合法（1)对任意gi, j，
      为其单独分配一个线程;(2)查找gi, j的所有贡献点;(3)计算每个贡献点
      的局部重采样值V · w和局部权值w，并分别累加到V[gi, j]和Wi, j）
      * 对采样空间进行分块，使划分的区块能够全覆盖采样空间，并且区块与区块之间不存在重复覆盖的区域
      * 对于任意采样点sn，根据它的坐标信息计算出它所属于的区块
      * 建立区块-采样点的一对多映射，并存储在内存上
        * 固定法
        * 动态法
        * 预排序法（看起来最佳）
          * ![image-20200730181706260](/Users/ansakai/Library/Application Support/typora-user-images/image-20200730181706260.png)

  * 现状

    * 优化方法：
      * 基于GPU的散发
        * 为每个采样点分配独有的输出网格存储空间 ，太占用空间了
        * 令一个线程沿着采样轨迹遍历采样点并执行Gridding计算。在线程
          执行计算的过程中，将卷积核内网格点的加权和尽可能的存储在寄存器上，并且只有在网格点脱离卷积核范围的情况下，才将临时加权和写入设备内存
      * 基于预排序的聚合法
        * 减少对非贡献点的内存访问
      * CPU和GPU平台优化

* 异构环境下Gridding方法设计与算法描述

  * 利用CPU多线程对原生数据进行预排序，使贡献点的搜索范围大幅度降低;并且在GPU上实现聚合法，同时利用线程组织配置、数据布局策略以及线程粗化策略加快聚合法的计算速度。

  * 方法设计

    * 预排序（key-value排序、S的重排列和查找表的构建）
      * 基于区块号的快速查找表
        * 采样点->区块号->纬度环号
      * 基于环号的二级查找表
        * 二级查找表为环号-起始区块号的一一映射;其中，某环的起始区块是指S′的所有区块中编号最小并且属于该环的区块。
        * ![image-20200731104059382](/Users/ansakai/Library/Application Support/typora-user-images/image-20200731104059382.png)

    * 聚合法（贡献点的查找、贡献点的卷积计算）

      * 贡献点的查找方法

        * 区块号->纬度环号
        * 寻找卷积核内贡献点的问题就能
          够转换为计算卷积核内纬度环，并遍历纬度环寻找起始贡献区块的问题。
          这里，起始贡献区块是指S ′pix中既位于rix环又位于卷积核内，并且编号最小
          的区块。‘
        * ![image-20200731104916612](/Users/ansakai/Library/Application Support/typora-user-images/image-20200731104916612.png)

      * 基于GPU的并行优化策略

        * 通过线程组织的配置策略实现任务的负载均衡
        * 通过不同的设备内存类型实现数据布局的优化（另一种因地制宜么）
        * 在高分辨率时使用线程粗化加速聚合法（不大理解）

        * ![image-20200731105637570](/Users/ansakai/Library/Application Support/typora-user-images/image-20200731105637570.png)

* HyGrid

  * 步骤
    * read
    * sort（预）
    * 建查找表（预）
    * 导入数据（聚）
    * 执行kernel（聚）
    * 导出数据（聚）
    * 写数据
  * 算法

* HyGrid实现

  * 架构
    * ![image-20200731113540346](/Users/ansakai/Library/Application Support/typora-user-images/image-20200731113540346.png)
  * 测试
    * 测试HpxPreOrdering ：
      * ![image-20200731115326300](/Users/ansakai/Library/Application Support/typora-user-images/image-20200731115326300.png)
    * 测试GPUGridGather ：
    * 测试HyGrid:根据以上分析，在HpxPreOrdering算法调用最佳排序算 法，GPUGridGather使用最优线程组织配置、数据布局策略的前提下， 针对HyGrid进行了以下测试: 
      * ![image-20200731115657219](/Users/ansakai/Library/Application Support/typora-user-images/image-20200731115657219.png)