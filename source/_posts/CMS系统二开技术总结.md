---
title: CMS系统二开技术总结
tags:
  - php
  - coding
abbrlink: 5c8b710b
date: 2022-05-23 07:11:36
---

最近接活，客户需求是给现有的CMS系统记录文章阅读量，要求按天统计做出柱状图表，三下五除二就做完了，客户很满意。

<!--more-->

这个CMS系统是[易优CMS|企业建站系统_免费_安全_易用-Eyoucms](https://www.eyoucms.com/)

内部仅仅是做了文章访问量一个字段，并没有记录访问日期等等。

![image-20220523053703898](CMS系统二开技术总结/image-20220523053703898.png)

## 实现效果 （评估需求）

![image-20220523054017168](CMS系统二开技术总结/image-20220523054017168.png)

内容管理页可以看到今日浏览数量，点进数字之后可以看见汇总图表

![image-20220523055727995](CMS系统二开技术总结/image-20220523055727995.png)

栏目管理处也可以看到栏目下的所有文章阅读量汇总

![image-20220523060322926](CMS系统二开技术总结/image-20220523060322926.png)

也是可以点进去有个图表展示，这里就不演示了。

所以总结下来，客户需求如下：

- 按日统计访问量，并做图表展示

- 包括内容管理、栏目管理
- 内容管理是单独文章展示浏览量，粒度是文章
- 栏目管理是栏目下所有文章的浏览量，粒度是当前栏目。如果是一级栏目，那么便是所有二级栏目统计的数量汇总。



## 设计分析

下面分为数据库结构设计和可能用到的查询语句列表，整理出基本框架，便可以动手编程了。

### 结构设计

因为需要数据库记录，所以新开一张表：`read_count`，存储栏目编号（typeid）、文章编号（aid）、日期（click_time）、当日阅读量（click_count）日期存放精确到天

### 数据库操作

#### 关于文章查询

查询今日阅读量：当前时间转成`Y-m-d`格式，作为条件并且带上**文章ID**限制去数据库查，拎出click_count字段即可

查询时间区间阅读量：传入`startTime`和`endTime`，使用`whereBetween`



#### 关于栏目查询

二级栏目查询今日阅读量：同文章时间格式，带上**栏目ID**限制去数据库查，对时间打组（`groupBy`），将click_count字段汇总求和（`sum`）即可

查询时间区间阅读量：同文章时间格式，同以上查询条件



#### 文章阅读记录

我这里魔改了官方统计标签`arcclick`，在其要对文章浏览量自增的时候，为`read_count`插入一条记录

前端保证传入正确的`typid`和`aid`，带上当前时间即可！



## 编码开发

### 阅读记录

首先要先有数据，先把文章阅读记录做出来

```php
/**
* 内容页浏览量的自增接口
*/
public function arcclick()
{
    \think\Session::pause(); // 暂停session，防止session阻塞机制
    if (IS_AJAX) {
        $click = 0;
        $aid = input('aid/d', 0);
        $type = input('type/s', '');
        if ($aid > 0) {
            $archives_db = Db::name('archives');
            
            if ('view' == $type) {
                $archives_db->where(array('aid' => $aid))->setInc('click');
                $archive = Db::name('archives')->where(array('aid' => $aid))->find();
                // 查出当前文章
+               $read_count = Db::name("read_count")->where([
+                   'typeid' => $archive['typeid'],
+                   'aid' => $aid, 'click_time' => date('Y-m-d', time())
+               ])->find();
                // 如果存在一条记录是当前文章，则对其自增，否则插入当前文章
+               if (!$read_count) {
+                   Db::name("read_count")->insertGetId([
+                       'typeid' => $archive['typeid'],
+                       'aid' => $aid, 'click_time' => date('Y-m-d', time()),
+                       'click_count' => 1,
+                   ]);
+               } else {
+                   Db::name("read_count")->where([
+                       'typeid' => $archive['typeid'],
+                       'aid' => $aid, 'click_time' => date('Y-m-d', time())
+                   ])->setInc('click_count', 1);
+               }
            }

            $click = $archives_db->where(array('aid' => $aid))->getField('click');
        }
        echo ($click);
        exit;
    } else {
        abort(404);
    }
}
```

当访问的时候，我们的数据库就会有

![image-20220523063513457](CMS系统二开技术总结/image-20220523063513457.png)

### 文章统计查询

`SQL`什么的如我上面所说，没有什么问题，关于时间上：

```php
$today = date('Y-m-d', time());
// 计算全部访问量,直接查
$data_count['total'] = Db::name('read_count')->where(['typeid' => $typeid, 'aid' => $aid])->sum('click_count');
$data_count['total'] = $data_count['total'] ? $data_count['total'] : 0;
// 上个月也就是当前时间减一个月
$lastMonth = date('Y-m-d', strtotime('-1 month', strtotime(date('Y-m-d'))));
// 将上个月所有click_count汇总
$data_count['month'] = Db::name('read_count')->where(['typeid' => $typeid, 'aid' => $aid])->whereBetween('click_time', [$lastMonth, $today])->sum('click_count');
$data_count['month'] = $data_count['month'] ? $data_count['month'] : 0;
```

### 栏目统计文章阅读量查询

```php
// 计算上个月访问量
$lastMonth = date('Y-m-d', strtotime('-1 month', strtotime(date('Y-m-d'))));

$data_count['month'] = Db::name('read_count')->whereIn('typeid', $typeid)->whereBetween('click_time', [$lastMonth, $today])->sum('click_count');
$data_count['month'] = $data_count['month'] ? $data_count['month'] : 0;
```

也很简单，不过这里要注意，我用的是`whereIn`，后面传入个id数组，而这里的id数组也就是所有栏目id，因为要考虑顶级栏目，看代码吧

```php
// controller: ReadCount
$typeid = input('typeid/d', 0);
$level = input('grade/d', 2);
if ($level == 0 || $level == 1) {
    // 处理二级栏目
    // 查出所有子项id
    $ids = Db::name('arctype')->where('parent_id', $typeid)->column('id');
    // 去查子项
    $typeid = !empty($ids) ? $ids : [];
	if($level == 0){
        // 处理一级栏目
        // 查出所有子项id
        $ids = Db::name('arctype')->where('parent_id', $typeid)->column('id');
        // 再去查子项
        $typeid = !empty($ids) ? $ids : [];
    }
} else {
    $typeid = [$typeid];
}
```

栏目表字段结构：

```sql
-- -----------------------------
-- Table structure for `ey_arctype`
-- -----------------------------
DROP TABLE IF EXISTS `ey_arctype`;
CREATE TABLE `ey_arctype` (
  `id` int(10) NOT NULL AUTO_INCREMENT COMMENT '栏目ID',
  `parent_id` int(10) DEFAULT '0' COMMENT '栏目上级ID',
  `typename` varchar(200) DEFAULT '' COMMENT '栏目名称',
  `grade` tinyint(1) DEFAULT '0' COMMENT '栏目等级',
  -- 省略一些无关紧要的字段...
  PRIMARY KEY (`id`),
  UNIQUE KEY `dirname` (`dirname`,`lang`) USING BTREE,
  KEY `parent_id` (`channeltype`,`parent_id`) USING BTREE
) ENGINE=MyISAM AUTO_INCREMENT=68 DEFAULT CHARSET=utf8 COMMENT='文档栏目表';
```

三级栏目parent_id是二级栏目，二级栏目parent_id是一级栏目，一级栏目parent_id是0

这里默认是三级栏目，如果是二级栏目，查出所有二级栏目id，汇总

如果是一级栏目，先把所有二级栏目阅读数算出来，然后再把一级栏目下的所有二级栏目阅读数汇总



### Echarts 图表

直接看示例，复制一份过来

[Examples - Apache ECharts](https://echarts.apache.org/examples/zh/editor.html?c=bar-background)

也很简单，查出所有记录汇总成`Echarts`需要的格式就好了。

说一下我遇到的问题

#### x轴柱子溢出：![如何巧妙地解决echarts柱状图x轴溢出问题](CMS系统二开技术总结/16cd73c6df345e6dtplv-t2oaga2asx-zoom-crop-mark130413041304734.awebp)

参考：[如何巧妙地解决echarts柱状图x轴溢出问题 - 掘金 (juejin.cn)](https://juejin.cn/post/6844903857995120653)

#### x轴文字太长以至于消失

![img](CMS系统二开技术总结/Center.png)

参考：[(87条消息) echarts x轴文字显示不全(xAxis文字倾斜比较全面的3种做法值得推荐)_dotnet全栈开发的博客-CSDN博客_echarts x轴文字显示不全](https://blog.csdn.net/kebi007/article/details/68488694)

博主给的几种方案都不错，我最后是将年份和月日分割成两行这样，问题解决。

## 系统上线

客户发给我他们的宝塔面板，我直接将改过的代码贴进去了，其实一切都很顺利。

进入我写的页面报错，貌似是说“? 语法错误”，原来是我写的新语法

```php
$a = $b ?? 0 
```

在他的php5上是不认的，改成php7就好了。

第一次找不到表是因为没创建，创建就好了。



## 总结

其实我遇到的问题不止上面这些，这个CMS系统把`thinkphp`内核都改了，一些数据库查询方法还带有保护字段，害我不得不去看`thinkorm`源码，不过也没什么，读本地的配置文件而已，源码就在手里，没有什么搞不了的。

用php做debug好麻烦啊，我纯靠脑子运行代码，一些逻辑都是我猜出来的，还有一些关于数组操作的，一见到数组我就想念`lodash.js`了。。。

