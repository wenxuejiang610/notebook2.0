title: 地理位置附近查询的GEOHASH解决方案
tag: miscellany
---
地理位置附近查询的GEOHASH解决方案
<!-- more -->

## 1.需求场景

现今互联网确实从方方面面影响我们的生活。现在我们可以足不出户就能买到我们心仪的衣服，找到附近的美食。当我们点开一个外卖的app就能看到自己附近的餐厅，那我们有没有想过这是怎么实现的呢？


![image](http://xiaozhao.oursnail.cn/GEOHASH%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%881.png)

## 2.尝试解决

- 首先我们能想到的就是把所有餐厅的经纬度存下来
- 然后当用户选择附近餐厅时
- 我们先获取用户的经纬度，然后到数据库中查出所有的经纬度，依次计算它们和用户间的距离。
- 最后根据用户输入的距离范围过滤出合适的餐厅，并根据距离做一个升序排列。

这样貌似能查出附近的餐厅，但是餐厅的数量这么多，直接全查出来内存也要爆掉，即使分批处理计算量也十分大。这样用户等待的时间就会特别长。那有什么办法能减少我们的计算量呢？


![image](http://xiaozhao.oursnail.cn/GEOHASH%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%883.png)

其实很简单，我们应该只计算用户关心的那一片数据，而不是计算所有的。例如用户在北京，那完全没必要计算海南，黑龙江，新疆，浙江等其它地区的数据。如果我们能快速定位到北京甚至某个区，那么我们的计算量将大大减少。我们发现这其实就是索引的功能，但是`MySQL`对这种二维的地理位置的索引支持并不友好（`mongodb`有直接的地理位置索引），它对一维的像字符串这样的支持很好。那如果我们的数据在MySQL中，有没有什么方法能将我们的二维坐标转换为一种可比较的字符串呢？这就是我们今天要介绍的`geohash`算法。

## 3.基本思想

**`geohash`简单来说就是将一个地理坐标转换为一个可比较的字符串的算法。不过生成的字符串表示的是一个矩形的范围，并不是一个点。**

比如西二旗地铁附近这一片矩形区域就可以用`wx4eyu82`这个字符串表示，并且越靠前的编码表示额范围越大，比如中国绝大部分地区可以用w这个字母表示的矩形区域内。像`wx4eyu82`表示的区域一定在`wx4e`表示的区域范围内。利用这些特性我们就可以实现附近餐厅的功能了，比如我们希望查看西二旗地铁附近的餐厅就可以这样查询：`select * from table where geohash like 'wx4eyu82%';` 这样就可以利用索引，快速查询出相关餐厅的信息了。并且我们还可以用`wx4eyu82`为`key`，餐厅信息为`value`做缓存。

![image](http://xiaozhao.oursnail.cn/GEOHASH%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%882.png)

**通过上面的介绍我们知道了`GeoHash`就是一种将经纬度转换成字符串的方法，并且使得在大部分情况下，字符串前缀匹配越多的距离越近.**

## 4.GeoHash算法的步骤

- 首先我们将经度和纬度都单独转换为一个二进制编码
- 得到经度和纬度的二进制编码后，我们按照奇数位放纬度，偶数为放经度的规则（我们这里奇数偶数下标是从0开始）将它们合成一个二进制编码
- 最后我们需要将这个二进制编码转换为base32编码

**举例**

- 地球纬度区间是[-90,90]， 北海公园的纬度是39.928167，可以通过下面算法对纬度39.928167进行逼近编码:
	- 区间[-90,90]进行二分为[-90,0),[0,90]，称为左右区间，可以确定39.928167属于右区间[0,90]，给标记为1；
	- 接着将区间[0,90]进行二分为 [0,45),[45,90]，可以确定39.928167属于左区间 [0,45)，给标记为0；
	- 递归上述过程39.928167总是属于某个区间[a,b]。随着每次迭代区间[a,b]总在缩小，并越来越逼近39.928167；
	- 如果给定的纬度x（39.928167）属于左区间，则记录0，如果属于右区间则记录1，这样随着算法的进行会产生一个序列1011100，序列的长度跟给定的区间划分次数有关。
- 通过上述计算，纬度产生的编码为10111 00011，经度产生的编码为11010 01011。偶数位放经度，奇数位放纬度，把2串编码组合生成新串：11100 11101 00100 01111。
- 最后使用用0-9、b-z（去掉a, i, l, o）这32个字母进行base32编码，首先将11100 11101 00100 01111转成十进制，对应着28、29、4、15，十进制对应的编码就是wx4g。

## 5.缺陷-geohash的边界问题

比如红色的点是我们的位置，绿色的两个点分别是附近的两个餐馆，但是在查询的时候会发现距离较远餐馆的`GeoHash`编码与我们一样（因为在同一个GeoHash区域块上），而较近餐馆的`GeoHash`编码与我们不一致。

![image](http://xiaozhao.oursnail.cn/GEOHASH%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%884.png)

目前比较通行的做法就是我们不仅获取当前我们所在的矩形区域，还获取周围8个矩形块中的点。那么怎样定位周围8个点呢？关键就是需要获取周围8个点的经纬度，那我们已经知道自己的经纬度，只需要用自己的经纬度减去最小划分单位的经纬度就行。因为我们知道经纬度的范围,又知道需要划分的次数，所以很容易就能计算出最小划分单位的经纬度。

## 6.几种实现geohash方案的对比

#### 6.1支持二维索引的存储数据库：mongodb

`mongoDB`支持二维空间索引,使用空间索引,`mongoDB`支持一种特殊查询,如某地图网站上可以查找离你最近的咖啡厅,银行等信息。这个使用`mongoDB`的空间索引结合特殊的查询方法很容易实现。

- API直接支持，很方便
- 支持按照距离排序，并支持分页。支持多条件筛选。
- 可满足实时性需求。
- 资源占用大，数据量达到百万级请流量在10w左右查询速度明显下降。

#### 6.2升级Mysql至5.7，支持Geohash

`MySQL 5.7.5` 增加了对`GeoHash`的支持，提供了一系列`geohash`的函数，但是其实`Mysql`并没有提供类似`mogodb`类型`near`这样的函数，仅仅提供了一些经纬度转`hash`、`hash`取经纬度的一些函数。

- 优点:函数直接调用，生成目标`hash`、根据`hash`获取经纬度。
- 缺点：不支持范围查询函数，需要自行处理周边8点的问题，需要补充`geo`的算法

#### 6.3Redis Commands: Geography Edition

GEO 特性在 Redis 3.2 版发布， 这个功能可以将用户给定的地理位置信息储存起来， 并对这些信息进行操作，GEO通过如下命令来完成GEO需求.



命令 | 描述
---|---
geoadd | 添加一个或多个经纬度地理位置
georadius | 获取指定范围内的对象，也可以增加参数withdistance直接算出距离，也可以增加参数descending/ascending 进行距离排序
georadiusbymember | 通过指定的对象，获取其周边对象
geoencode | 转换为geohash，52-bit，同时返回该区域最小角的geohash,最大角的geohash，及中心点
geodecode | 同上逆操作

- 优点:效率高，API丰富
- 缺点：3.2版本是否稳定？

面试的时候，问到geohash算法以及技术选型大概也能说一说了...

本文章借鉴很多优秀文章，七拼八凑而出。