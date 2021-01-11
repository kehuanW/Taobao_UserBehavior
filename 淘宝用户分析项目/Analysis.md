# 淘宝用户行为分析

## 一、简介

### 1. 数据集

#### (1) 数据来源

https://tianchi.aliyun.com/dataset/dataDetail?dataId=649

#### (2) 概况

**UserBehavior.csv**

本数据集包含了2017年11月25日至2017年12月3日之间，约一百万随机用户的所有行为（行为包括点击、购买、加购、喜欢）。关于数据集中每一列的详细描述如下：

| 列名          | 解释                                                         |
| ------------- | ------------------------------------------------------------ |
| User_ID       | Integer                                                      |
| Item_ID       | Integer                                                      |
| Category_ID   | Integer                                                      |
| Behavior_Type | String ('pv'-点击, 'fav'-收藏, 'cart'-加入购物车, 'buy'-购买) |
| Timestamp     | Integer                                                      |

### 2. 分析思路

#### (1) 总体流量指标分析

**1) 总量统计 (总UV, 总PV, 总成交量)**

**2) 日统计 (日期, 日UV, 日PV, 日PV/日UV, 日成交量)**

**3) 复购率和跳失率**

复购率=购买次数>1的用户数/购买次数=1的用户数

跳失率=只有一次浏览行为的用户数/总用户数

#### (2) 留存率与用户转化分析

**1) 活跃用户第n日留存率**

**2) PV转化**

**3) 用户转化不同路径**

#### (3) 用户行为习惯分析

**1) 日分布 **

**2) 时刻分布**

#### (4) 用户商品偏好分析

**商品类目**

#### (5) 用户商业价值分析

**RFM模型**

https://www.zhihu.com/question/49439948?sort=created



## 二、过程详解

## 1. 数据处理

### (1) 数据导入

由于源数据集过大，本次仅导入了前15000条数据进行分析。

数据导入工具：Navicat

### (2) 重复值查询

```mysql
SELECT * 
FROM   userbehavior 
GROUP BY User_ID,Item_ID,Category_ID,Behavior_Type,Timestamp 
HAVING COUNT(*) > 1;
```

### (3) 缺失值查询

```mysql
SELECT count(User_ID),count(Item_ID),count(Category_ID),count(Behavior_type),count(Timestamp) 
FROM   userbehavior;
```

### (4) 分离出Date与Time

```mysql
ALTER TABLE userbehavior ADD COLUMN Date_time TIMESTAMP(0) NULL;
ALTER TABLE userbehavior ADD COLUMN Date char(10) NULL; # 后改为Date格式
ALTER TABLE userbehavior ADD COLUMN Time char(10) null; 

UPDATE userbehavior
SET Date_time = FROM_UNIXTIME(Timestamp);

UPDATE userbehavior
SET Date = FROM_UNIXTIME(Timestamp,'%y-%m-%d');

update userbehavior
SET Time = SUBSTRING(Date_time FROM 12 FOR 2);
```

###(5) 查看数据的时间分布

```mysql
SELECT date, COUNT(*)
FROM   userbehavior
GROUP BY date;
```

### (6)创建WeekData视图

```mysql
CREATE VIEW WeekData AS 
SELECT User_ID, Item_ID, Category_ID, Behavior_Type, Date, Time
FROM   userbehavior 
WHERE  Date BETWEEN '2017-11-27' AND '2017-12-03';
```

## 2. 数据分析

### 2.1 总体流量指标分析

#### (1) 总量统计

```mysql
SELECT COUNT(DISTINCT User_ID) AS '总用户数量(UV)', 
	   SUM(CASE WHEN Behavior_Type = 'pv' THEN 1 END) AS '总访问量(PV)',
	   SUM(CASE WHEN Behavior_Type = 'buy' THEN 1 END) AS '总成交量'
FROM   weekdata;
```

#### (2) 日统计

```mysql
SELECT Date AS '时间', COUNT(DISTINCT User_ID) AS '日访客数',
	   SUM(CASE WHEN Behavior_type='pv' THEN 1 END) AS '日浏览量',
       ROUND(SUM(CASE WHEN Behavior_type='pv' THEN 1 END)/COUNT(DISTINCT User_ID),2) AS '日人均浏览次数',
       SUM(CASE WHEN Behavior_type ='buy' THEN 1 END) AS '日成交量'
FROM   weekdata
GROUP BY Date
ORDER BY Date;
```

####(3) 复购率和跳失率

```mysql
-- 创建用户行为透视表
CREATE VIEW userpivot AS
SELECT User_ID, COUNT(Behavior_Type) AS 用户行为量,
			 SUM(CASE WHEN Behavior_type='pv' THEN 1 ELSE 0 END) AS 点击,
			 SUM(CASE WHEN Behavior_type='fav' THEN 1 ELSE 0 END) AS 收藏,
			 SUM(CASE WHEN Behavior_type='cart' THEN 1 ELSE 0 END) AS 加购物车,
			 SUM(CASE WHEN Behavior_type='buy' THEN 1 ELSE 0 END) AS 购买
FROM   weekdata
GROUP BY User_ID
ORDER BY 用户行为量 DESC;

-- 计算复购率与跳失率
SELECT CONCAT(ROUND(SUM(CASE WHEN 购买 > 1 THEN 1 ELSE 0 END) / SUM(CASE WHEN 购买 > 0 THEN 1 ELSE 0 END)*100,2),'%') AS 复购率,
	   CONCAT(ROUND((SELECT COUNT(*) FROM userpivot WHERE 点击 = 1 AND 收藏 = 0 AND 加购物车 = 0 AND 购买 = 0)/COUNT(*), 2)*100,'%') AS 跳失率
FROM   userpivot; 
```

### 2.2 用户留存与转化分析

#### (1) 留存率

```mysql
-- 活跃用户留存率计算
-- 方法一（重点）
-- (1)活跃用户计算
CREATE VIEW ActiveUser AS
SELECT A.Date, 
       COUNT(DISTINCT IF(DATEDIFF(B.Date, A.Date)=0, A.User_ID, NULL)) AS 'DAU',
       COUNT(DISTINCT IF(DATEDIFF(B.Date, A.Date)=1, A.User_ID, NULL)) AS 'DAU_1',
       COUNT(DISTINCT IF(DATEDIFF(B.Date, A.Date)=2, A.User_ID, NULL)) AS 'DAU_2',
       COUNT(DISTINCT IF(DATEDIFF(B.Date, A.Date)=3, A.User_ID, NULL)) AS 'DAU_3',
       COUNT(DISTINCT IF(DATEDIFF(B.Date, A.Date)=4, A.User_ID, NULL)) AS 'DAU_4',
       COUNT(DISTINCT IF(DATEDIFF(B.Date, A.Date)=5, A.User_ID, NULL)) AS 'DAU_5',
       COUNT(DISTINCT IF(DATEDIFF(B.Date, A.Date)=6, A.User_ID, NULL)) AS 'DAU_6'
FROM   ((SELECT User_ID, Date FROM weekData) AS A
       JOIN
       (SELECT User_ID, Date FROM weekData) AS B
       ON A.User_ID = B.User_ID AND A.Date <= B.Date)
GROUP BY A.Date;

-- (2)留存率计算
SELECT Date AS '日期', DAU,
       CONCAT(ROUND(DAU_1/DAU*100, 2),'%') AS '次日留存率',
       CONCAT(ROUND(DAU_2/DAU*100, 2),'%') AS '第二日留存率',
       CONCAT(ROUND(DAU_3/DAU*100, 2),'%') AS '第三日留存率',
       CONCAT(ROUND(DAU_4/DAU*100, 2),'%') AS '第四日留存率',
       CONCAT(ROUND(DAU_5/DAU*100, 2),'%') AS '第五日留存率',
       CONCAT(ROUND(DAU_6/DAU*100, 2),'%') AS '第六日留存率'
FROM   ActiveUser;

-- 方法二（仅仅列出日活跃用户数计算）
SELECT  A.Date AS DATE, COUNT(DISTINCT A.User_ID) AS '活跃用户数',
        COUNT(DISTINCT B.User_ID) AS '次日留存用户数',
				COUNT(DISTINCT D.User_ID) AS '三日留存用户数',
				COUNT(DISTINCT F.User_ID) AS '五日留存用户数'
FROM   (SELECT User_ID, Date FROM weekdata) AS A
			 LEFT JOIN (SELECT User_ID, Date FROM weekdata) AS B 
			 ON A.User_ID = B.User_ID AND DATEDIFF(B.Date, A.Date) = 1
			 LEFT JOIN (SELECT User_ID, Date FROM weekdata) AS D 
			 ON A.User_ID = D.User_ID AND DATEDIFF(D.Date, A.Date) = 3
			 LEFT JOIN (SELECT User_ID, Date FROM weekdata) AS F 
			 ON A.User_ID = F.User_ID AND DATEDIFF(F.Date, A.Date) = 5
GROUP BY A.Date;
```

#### (2) PV转化

```mysql
SELECT CONCAT(ROUND(SUM(CASE WHEN Behavior_Type = 'pv' THEN 1 ELSE 0 END)/COUNT(*)*100,2),'%') AS 'pv占比',
	   CONCAT(ROUND(SUM(CASE WHEN Behavior_Type = 'fav' THEN 1 ELSE 0 END)/COUNT(*)*100,2),'%') AS 'fav占比',
	   CONCAT(ROUND(SUM(CASE WHEN Behavior_Type = 'cart' THEN 1 ELSE 0 END)/COUNT(*)*100,2),'%') AS 'cart占比',
	   CONCAT(ROUND(SUM(CASE WHEN Behavior_Type = 'buy' THEN 1 ELSE 0 END)/COUNT(*)*100,2),'%') AS 'buy占比'
FROM   weekdata; 
-- PV约占总行为量的90%
```

```mysql
-- 行为转化
-- PV转化为其他行为的概率
-- 率
SELECT CONCAT(ROUND(SUM(CASE WHEN Behavior_Type = 'pv' THEN 1 ELSE 0 END)/SUM(CASE WHEN Behavior_Type = 'pv' THEN 1 ELSE 0 END)*100,2),'%') AS 'pv',
	   CONCAT(ROUND(SUM(CASE WHEN Behavior_Type = 'fav' THEN 1 ELSE 0 END)/SUM(CASE WHEN Behavior_Type = 'pv' THEN 1 ELSE 0 END)*100,2),'%') AS 'fav',
	   CONCAT(ROUND(SUM(CASE WHEN Behavior_Type = 'cart' THEN 1 ELSE 0 END)/SUM(CASE WHEN Behavior_Type = 'pv' THEN 1 ELSE 0 END)*100,2),'%') AS 'cart',
	   CONCAT(ROUND(SUM(CASE WHEN Behavior_Type = 'buy' THEN 1 ELSE 0 END)/SUM(CASE WHEN Behavior_Type = 'pv' THEN 1 ELSE 0 END)*100,2),'%') AS 'buy'
FROM   weekdata;

-- 量
SELECT SUM(CASE WHEN Behavior_Type = 'pv' THEN 1 ELSE 0 END) AS 'pv',
	   SUM(CASE WHEN Behavior_Type = 'fav' THEN 1 ELSE 0 END) AS 'fav',
	   SUM(CASE WHEN Behavior_Type = 'cart' THEN 1 ELSE 0 END) AS 'cart',
	   SUM(CASE WHEN Behavior_Type = 'buy' THEN 1 ELSE 0 END) AS 'buy'
FROM   weekdata;

-- 用户转化
-- 有PV的用户转化为有其他行为的用户
-- 率
SELECT '100.00%' AS 点击用户转化率,
       CONCAT(ROUND(SUM(CASE WHEN 收藏>0 THEN 1 ELSE 0 END)/SUM(CASE WHEN 点击>0 THEN 1 ELSE 0 END)*100,2),'%') AS 收藏用户转化率,
	   CONCAT(ROUND(SUM(CASE WHEN 加购物车>0 THEN 1 ELSE 0 END)/SUM(CASE WHEN 点击>0 THEN 1 ELSE 0 END)*100,2),'%') AS 加购用户转化率,
       CONCAT(ROUND(SUM(CASE WHEN 购买>0 THEN 1 ELSE 0 END)/SUM(CASE WHEN 点击>0 THEN 1 ELSE 0 END)*100,2),'%')  AS 购买用户转化率
FROM   userpivot;

-- 量
SELECT SUM(CASE WHEN 点击>0 THEN 1 ELSE 0 END) AS 点击用户数,
       SUM(CASE WHEN 收藏>0 THEN 1 ELSE 0 END) AS 收藏用户数,
	   SUM(CASE WHEN 加购物车>0 THEN 1 ELSE 0 END) AS 加购用户数,
       SUM(CASE WHEN 购买>0 THEN 1 ELSE 0 END) AS 购买用户数
FROM   userpivot;
```

#### (3) 用户转化--不同路径

```mysql
-- 点击——购买的留存分析
SELECT *
FROM
(SELECT COUNT(DISTINCT User_ID) AS 浏览用户数 FROM weekdata WHERE Behavior_Type = 'pv') AS A,
(SELECT SUM(CASE WHEN 购买>0 THEN 1 ELSE 0 END) AS 购买用户数
FROM   userpivot
WHERE 收藏=0 AND 加购物车=0) AS B

-- 点击——加购——购买的留存分析
SELECT *
FROM
(SELECT COUNT(DISTINCT User_ID) AS 浏览用户数 FROM weekdata WHERE Behavior_Type = 'pv') AS A,
(SELECT SUM(CASE WHEN 加购物车>0 THEN 1 ELSE 0 END) AS 加购物车用户数,
        SUM(CASE WHEN 购买>0 THEN 1 ELSE 0 END) AS 购买用户数
FROM   userpivot
WHERE 收藏=0) AS B

-- 点击——收藏——购买的留存分析
SELECT *
FROM
(SELECT COUNT(DISTINCT User_ID) AS 浏览用户数 FROM weekdata WHERE Behavior_Type = 'pv') AS A,
(SELECT SUM(CASE WHEN 收藏>0 THEN 1 ELSE 0 END) AS 收藏用户数,
        SUM(CASE WHEN 购买>0 THEN 1 ELSE 0 END) AS 购买用户数
FROM   userpivot
WHERE 加购物车=0) AS B

-- 点击——收藏/购——购买的留存分析
SELECT *
FROM
(SELECT COUNT(DISTINCT User_ID) AS 浏览用户数 FROM weekdata WHERE Behavior_Type = 'pv') AS A,
(SELECT COUNT(*) AS '收藏/购物车用户数',
 	    SUM(CASE WHEN 购买>0 THEN 1 ELSE 0 END) AS 购买用户数
 FROM   userpivot
 WHERE  加购<>0 and 收藏<>0;) AS B
```

###2.3 用户行为习惯分析

#### (1) 日分布

```mysql
SELECT Date AS '日期', COUNT(DISTINCT User_ID) AS '用户量', COUNT(*) AS '行为量', 
	   SUM(CASE WHEN Behavior_Type = 'pv' THEN 1 ELSE 0 END) AS '浏览量',
	   SUM(CASE WHEN Behavior_Type = 'fav' THEN 1 ELSE 0 END) AS '收藏量',
	   SUM(CASE WHEN Behavior_Type = 'cart' THEN 1 ELSE 0 END) AS '加入购物车量',
	   SUM(CASE WHEN Behavior_Type = 'buy' THEN 1 ELSE 0 END) AS '购买量'
FROM   weekdata
GROUP BY Date
ORDER BY Date;
```

#### (2) 时刻分布

```mysql
SELECT Time AS '时刻', COUNT(DISTINCT User_ID) AS '用户量', COUNT(*) AS '行为量',
	   SUM(CASE WHEN Behavior_Type = 'pv' THEN 1 ELSE 0 END) AS '浏览量',
	   SUM(CASE WHEN Behavior_Type = 'fav' THEN 1 ELSE 0 END) AS '收藏量',
	   SUM(CASE WHEN Behavior_Type = 'cart' THEN 1 ELSE 0 END) AS '加入购物车量',
	   SUM(CASE WHEN Behavior_Type = 'buy' THEN 1 ELSE 0 END) AS '购买量'
FROM   weekdata
GROUP BY Time
ORDER BY Time;
```

### 2.4 用户商品偏好分析

#### (1) 商品类目分析

```mysql
CREATE VIEW CategoryTable AS
SELECT *
FROM (SELECT Category_ID, pv, fav, cart, buy, RANK() OVER (ORDER BY buy DESC) rnk_by_buy 
      FROM   (SELECT Category_ID, 
			        SUM(CASE WHEN Behavior_Type = 'pv' THEN 1 ELSE 0 END) AS 'pv',
                     SUM(CASE WHEN Behavior_Type = 'fav' THEN 1 ELSE 0 END) AS 'fav',
                     SUM(CASE WHEN Behavior_Type = 'cart' THEN 1 ELSE 0 END) AS 'cart',
                     SUM(CASE WHEN Behavior_Type = 'buy' THEN 1 ELSE 0 END) AS 'buy'
               FROM   weekdata
               GROUP BY Category_ID
			 ) AS A
	 ) AS B
WHERE rnk_by_buy < 21

SELECT Category_ID, pv, buy, CONCAT(ROUND(buy/pv*100, 2),'%') AS 'buy/pv',
       RANK() OVER (ORDER BY buy/pv DESC) AS 'rnk_by_buy/pv', rnk_by_buy
FROM   CategoryTable
```

### 2.5 用户价值分析

```mysql
-- R
CREATE VIEW R AS
SELECT User_ID, (CASE WHEN T_Interval = 6 THEN 1
					WHEN T_Interval BETWEEN 4 AND 5 THEN 2
					WHEN T_Interval BETWEEN 2 AND 3 THEN 3
					WHEN T_Interval BETWEEN 0 AND 1 THEN 4
				ELSE NULL END) AS R
FROM   (SELECT User_ID, DATEDIFF('2017-12-03',MAX(Date)) AS 'T_Interval'
	    FROM weekdata
		WHERE Behavior_Type = 'buy'
		GROUP BY User_ID) AS A;


-- F
CREATE VIEW F AS
SELECT User_ID, (CASE WHEN fre BETWEEN 1 AND 3 THEN 1
                      WHEN fre BETWEEN 4 AND 7 THEN 2
					WHEN fre BETWEEN 7 AND 12 THEN 3
					WHEN fre > 12 THEN 4 
				ELSE NULL END) AS F
FROM  (SELECT User_ID, COUNT(Behavior_type) AS fre
       FROM   weekdata
       WHERE  Behavior_type='buy'
       GROUP BY User_ID) AS A;
  

-- 客户分群
SELECT '挽留客户' AS '客户群', COUNT(User_ID) AS '数量' 
FROM R JOIN F USING (User_ID) 
WHERE R<(SELECT AVG(R) FROM R JOIN F USING (User_ID))
      AND F<(SELECT AVG(F) FROM R JOIN F USING (User_ID))
UNION ALL
SELECT '保持客户', COUNT(User_ID)
FROM R JOIN F USING (User_ID) 
WHERE R<(SELECT AVG(R) FROM R JOIN F USING (User_ID))
      AND F>(SELECT AVG(F) FROM R JOIN F USING (User_ID))
UNION ALL
SELECT '发展客户', COUNT(User_ID)
FROM R JOIN F USING (User_ID) 
WHERE R>(SELECT AVG(R) FROM R JOIN F USING (User_ID))
      AND F<(SELECT AVG(F) FROM R JOIN F USING (User_ID))
UNION ALL
SELECT '价值客户量', COUNT(User_ID)
FROM R JOIN F USING (User_ID) 
WHERE R>(SELECT AVG(R) FROM R JOIN F USING (User_ID))
      AND F>(SELECT AVG(F) FROM R JOIN F USING (User_ID))
UNION ALL
SELECT '总计', COUNT(DISTINCT User_ID)
FROM   R JOIN F USING (User_ID);
```



## 三、结果及建议

1.通过对用户行为的转化分析，可以看出用户从点击到购买的转化率还是比较高的，目前来看可以通过引导用户收藏并加购来提高用户从收藏/加购到购买的转化率。而流量行为从点击到购买的转化率仅有2.3%，故从点击到购买的行为转化是一个提高的重点。这说明了用户大部分时间在浏览和寻找合适的商品上，并且有可能最终是因为没找到自己想购买的商品而流失的。针对这一环节的建议有：

- 优化电商平台的搜索匹配度和推荐策略，提高筛选精确度，并对搜索和筛选的结果排序的优先级进行优化；
- 可以给客户提供同类产品比较的功能，让用户不需要多次返回搜索结果进行点击查看，方便用户确定心仪产品，增加点击到后续行为的转化；
- 优化收藏到购买的操作过程，增加用户收藏并加购的频率，以提高购买转化率。

2.通过对用户行为的分析

- 可以看出用户的活跃时间高峰期主要在20-22点，此时使用人数最多，活动最容易触达用户，所以可以将营销活动安排在这个时间段内，来进行引流并转化。
- 周末点击量和用户量有明显增加，但是购物量并没有明显增加。周末推出营销活动来挖掘用户周末购物的欲望。

3.通过对用户的商品偏好分析，可以看出能吸引用户注意力的商品购买转化率并不高，是一个提高销量的突破口。针对用户关注度高但销量不高的这部分产品，可以从以下几个方面着手：

- 商家在详情页上的展示突出用户重点关注的信息，优化信息的呈现方式，减少用户的时间成本；
- 增加这些产品的质量管控力度，加强对用户反馈建议如评论的管理，认真采纳并根据自身的优劣势进行商品优化。

4.通过RFM模型对客户群进行划分，可以对不同的用户群体采用不同的管理策略，达到对不同的客户群进行精准营销的目的：

- 对于重要价值用户，需要重点关注并保持， 应该提高满意度，增加留存；
- 对于重要发展客户和重要保持用户，可以适当给点折扣或捆绑销售来增加用户的购买频率；
- 对于重要挽留客户，需要关注他们的购物习性做精准化营销，以唤醒他们的购买意愿。

