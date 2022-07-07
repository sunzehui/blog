---
title: 读《SQL反模式》评论表设计
tags:
  - SQL
abbrlink: 4d16e673
date: 2022-07-07 09:29:22
---

SQL是一门比高级语言更高级的语言，今天谈一下关于多级评论的存储结构设计。

<!--more-->

贴子跟评是很常见的场景，通常会有层主的跟评、跟评的跟评...

文中给出四个解决方案：邻接表、枚举路径、嵌套集、闭包表

## 邻接表

很容易想到的存储方法就是类似多级列表，记录该评论的`parent_id`。

```sql
CREATE TABLE Comments (
	comment_id SERIAL PROMARY KEY,
    parent_id BIGHNT UNSIGEND,
    FOREIGN KEY (parent_id) PEFERENCES Commonts(commont_id)
)
```

这种方法叫邻接表，查询嵌套结构使用如下方法

```sql
SELECT c1.*, c2.*, c3.*, c4.*
FROM Comments c1
  LEFT OUTER JOIN Comments c2
  	ON c2.parent_id = c1.comment_id
  LEFT OUTER JOIN Comments c3
  	ON c3.parent_id = c2.comment_id
  LEFT OUTER JOIN Comments c4
  	ON c4.parent_id = c3.comment_id
```

每增加一层嵌套，就要多写一层 `JOIN`，极其笨拙。

但插入数据，修改节点位置很简单。



## 路径枚举

邻接表的缺点之一是查询复杂，路径枚举使用一个字符串，将所有祖先的信息罗列出来，使得查询变得简单。

```sql
CREATE TABLE `comments` (
  `comment_id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `path` varchar(100) DEFAULT NULL,
  PRIMARY KEY (`comment_id`)
);
INSERT INTO `comments` VALUES (1, '1/');
INSERT INTO `comments` VALUES (2, '1/2/');
INSERT INTO `comments` VALUES (3, '1/2/3/');
INSERT INTO `comments` VALUES (4, '1/4/');
INSERT INTO `comments` VALUES (5, '1/4/5/');
INSERT INTO `comments` VALUES (6, '1/4/6/');
INSERT INTO `comments` VALUES (7, '1/4/6/7');
```

![image.png](读《SQL反模式》评论表设计/9e634294c5ee41a4b4361c646f862b36tplv-k3u1fbpfcp-zoom-in-crop-mark3024000.awebp)

这样可以很清楚的得到整棵树的信息，查询#7节点的所有祖先：

```sql
SELECT * from comments AS c where '1/4/6/7/' like CONCAT(c.path,'%');
```

查询#4节点的所有后代

```sql
SELECT * from comments AS c where c.path like CONCAT('1/4/','%');
```

插入一个节点需要做的只是复制一份要插入节点的逻辑上的父亲节点的路径，并将这个新节点的ID追加到路径末尾就行了。

```sql
INSERT INTO comments (author,comment_date,bug_id, comment) VALUES ('Ollie','2021-01-11', 1,'Good job!');

UPDATE comments 
SET path = ( SELECT b.path FROM ( SELECT CONCAT( path, '/8' ) AS path FROM comments WHERE comment_id = 7 ) AS b ) 
WHERE
	comment_id = 8;
```

但是也有弊端，比如路径长度限制在字段长度以内，不能做到无限嵌套。



## 嵌套集

这个方法是自己维护一个树，树节点上有特定的编号，这个编号算法挺复杂的，书上只给了叶子节点的插入方法。

不写了，这个一般用不到。



## 闭包表

闭包表是一个简单而优雅的方案，它记录了树中所有节点之间的关系，而不仅仅是只有直接的父子关系。

新建一个`TreePaths`表：

```sql
CREATE TABLE `treepaths` (
  `ancestor` bigint(20) unsigned NOT NULL,  # 父节点id
  `descendant` bigint(20) unsigned NOT NULL,# 子节点id
  PRIMARY KEY (`ancestor`,`descendant`),
  KEY `descendant` (`descendant`)
);

INSERT INTO `treepaths` VALUES (1, 1);
INSERT INTO `treepaths` VALUES (1, 2);
INSERT INTO `treepaths` VALUES (1, 3);
INSERT INTO `treepaths` VALUES (1, 4);
INSERT INTO `treepaths` VALUES (1, 5);
INSERT INTO `treepaths` VALUES (1, 6);
INSERT INTO `treepaths` VALUES (1, 7);
INSERT INTO `treepaths` VALUES (2, 2);
INSERT INTO `treepaths` VALUES (2, 3);
INSERT INTO `treepaths` VALUES (3, 3);
INSERT INTO `treepaths` VALUES (4, 4);
INSERT INTO `treepaths` VALUES (4, 5);
INSERT INTO `treepaths` VALUES (4, 6);
INSERT INTO `treepaths` VALUES (4, 7);
INSERT INTO `treepaths` VALUES (5, 5);
INSERT INTO `treepaths` VALUES (6, 6);
INSERT INTO `treepaths` VALUES (6, 7);
INSERT INTO `treepaths` VALUES (7, 7);
```

每一条记录代表父节点和子节点之间有一条边，包括自己指向自己的边。

获取评论#4的子节点，只需要获取所有父节点是4的节点。

```sql
SELECT
	c.*,
	t.* 
FROM
	comments AS c
	JOIN TreePaths AS t ON c.comment_id = t.descendant 
WHERE
	t.ancestor = 4;
```

获取评论#4的父节点，类似地查询节点子节点是#4的。

```sql
SELECT
	c.*,
	t.* 
FROM
	comments AS c
	JOIN TreePaths AS t ON c.comment_id = t.ancestor 
WHERE
	t.descendant = 4;
```

插入节点时，插入节点 = 新增节点父节点的所有父节点 + 指向自身节点

向节点1插入节点2时，查找节点1的所有父节点（包括自身），然后`UNION`新增节点指向自身的记录

```sql
INSERT INTO `comment`(id) values(2);
# 1,1
INSERT INTO TreePaths ( ancestor, descendant) SELECT
t.ancestor,
1
FROM
	rel AS t 
WHERE
	t.descendant = 1 UNION ALL
SELECT
	2,
	2;
```

删除一棵子树，应删除所有后代为该节点的记录，和以该节点作为后代的记录。

```sql
DELETE 
FROM
	TreePaths 
WHERE
	descendant IN ( SELECT descendant FROM ( SELECT descendant FROM TreePaths WHERE ancestor = 4 ) AS b );
```



## 总结

作者给出了总结

![image-20220707092706430](读《SQL反模式》评论表设计/image-20220707092706430.png)
