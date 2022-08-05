---
title: typeorm手动raw转entity
abbrlink: cf526def
date: 2022-08-05 17:26:28
tags:
- nestjs
---

今天用左连接后，查询平均成绩查不出来，判断是 ORM 在转换成实体类的时候，自动把不认识的字段过滤掉了，想到一解决方案，记录一下。

<!--more-->

## 问题描述

我的在线考试系统，后台老师要统计学生的平均成绩。

想着使用`AVG`统计平均分，文档说直接`addSelect`就好了

![060E8A4E08F1821231E7FFBBFE5035AA](typeorm手动raw转entity/060E8A4E08F1821231E7FFBBFE5035AA.jpg)

，但 **最后要使用`getRawMany`**，这样一来，我原本自动`RAW`到`Entity的转换就无了，变成没有结构的一堆数据，像这样

![image-20220805173709408](typeorm手动raw转entity/image-20220805173709408.png)

虽然结构很乱，但好歹我的平均分查出来了。

难道我要一个一个地转换？为什么自带了转换我不用我要自己转？

越想越不对劲，最终我想到能不能在·ORM·帮我转换前把平均分记录下来，然后再转换，然后拼接上平均分上去。



## 实现代码

事实证明是可行的。

根据大佬的代码：

[Mapping raw query to entities · Issue #6803 · typeorm/typeorm (github.com)](https://github.com/typeorm/typeorm/issues/6803#issuecomment-864681382)

下面是我的查询所有考试的平均分

```typescript
// record.service.ts
import { RelationCountLoader } from 'typeorm/query-builder/relation-count/RelationCountLoader';
import { RelationIdLoader } from 'typeorm/query-builder/relation-id/RelationIdLoader';
import { RawSqlResultsToEntityTransformer } from 'typeorm/query-builder/transformer/RawSqlResultsToEntityTransformer';

async findAll(userId: number) {
  // 创建 queryBuilder
  const qb = this.repo
  .createQueryBuilder('e_record')
  .addSelect('avg(e_record.score)', 'averageScore')
  .where('userId = :userId', { userId })
  .orWhere('e_record.relTeacherId = :userId', { userId })
  .leftJoinAndSelect(
    'e_record.exam_room',
    'e_room',
    'e_room.id = e_record.examRoomId',
  )
  .leftJoinAndSelect(
    'e_room.for_classes',
    'classes',
    'classes.id = e_room.for_classes',
  )
  .leftJoinAndSelect(
    'e_record.exam_paper',
    'e_paper',
    'e_paper.id = e_record.examPaperId',
  )
  .groupBy('e_record.relTeacherId');
  // 直接拿raw，后面转换
  const rawResults = await qb.getRawMany();
  // 临时保存聚合函数计算的平均分
  const avgScoreList = rawResults.map((entity) => ({
    id: entity.e_record_id,
    score: entity.averageScore,
  }));
  // 下面这些都是拿到用到的实体信息
  const connection = this.repo.manager.connection;
  const queryRunner = connection.createQueryRunner();
  const relationIdLoader = new RelationIdLoader(
    connection,
    queryRunner,
    qb.expressionMap.relationIdAttributes,
  );
  const relationCountLoader = new RelationCountLoader(
    connection,
    queryRunner,
    qb.expressionMap.relationCountAttributes,
  );
  const rawRelationIdResults = await relationIdLoader.load(rawResults);
  const rawRelationCountResults = await relationCountLoader.load(rawResults);
  const transformer = new RawSqlResultsToEntityTransformer(
    qb.expressionMap,
    connection.driver,
    rawRelationIdResults,
    rawRelationCountResults,
    queryRunner,
  );
  // 转换成实体
  const transformed = transformer.transform(
    rawResults,
    qb.expressionMap.mainAlias,
  );
  return transformed.map((entity) => {
    // 将平均分拼接到转换后的实体上
    const avgScore =
          avgScoreList.find((score) => score.id === entity.id)?.score || 0;
    return {
      ...entity,
      avgScore,
    };
  });
}
```

查询结果符合我的预期，带结构并且平均分也查出来了

![image-20220805175515153](typeorm手动raw转entity/image-20220805175515153.png)
