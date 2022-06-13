---
title: 在nestjs中对文章和标签CRUD
tags:
  - nestjs
abbrlink: b7a9926e
date: 2022-05-21 20:07:38
---

研究了一天，该踩的坑一个不少，Typeorm用起来不太顺手。

<!--more-->



## 业务分析

关于文章的增删改查：

![image-20220521185149716](在nestjs中对文章和标签CRUD/image-20220521185149716.png)

这是我画的ER图：

![image-20220521185911497](在nestjs中对文章和标签CRUD/image-20220521185911497.png)

### 实体之间关系：

文章与标签是多对多，一个文章下可以有多个标签，一个标签也可以对应多个文章

文章与用户是一对多，用户可以有多个文章，一篇文章只有一个作者（用户）

### 基本步骤：

创建和更新时：用户（user）提交文章（article），解析出所包含的标签（tag），标签和用户关联到文章上。

查询时：查询所有文章，并将关联标签输出

删除时（软删除）：删除指定文章，并将关联标签删除



### 实体定义文件

**文章**

```typescript
import { User } from '@user/entities/user.entity';
import { Tag } from 'src/tag/entities/tag.entity';
import {
    Entity,
    Column,
    PrimaryGeneratedColumn,
    ManyToOne,
    ManyToMany,
    JoinTable,
    JoinColumn,
    DeleteDateColumn,
} from 'typeorm';
@Entity()
export class Article {
    @PrimaryGeneratedColumn()
    id: string;

    @Column({ length: 255 })
    title: string;

    @Column({ default: '', length: 255 })
    content: string;

    //   关联user和tag;
    @ManyToOne((type) => User, (user) => user.articles)
    user: User;

    @DeleteDateColumn()
    deleteTime: Date;

    @ManyToMany((type) => Tag, (tag) => tag.articles, {
        cascade: true,
        onDelete: 'CASCADE',
        onUpdate: 'CASCADE',
    })
    @JoinTable({
        name: 'article_tag', // 此关系的联结表的表名
        joinColumn: {
            name: 'tag',
            referencedColumnName: 'id',
        },
        inverseJoinColumn: {
            name: 'article',
            referencedColumnName: 'id',
        },
    })
    tags: Tag[];

    @Column()
    createTime: string;

    @Column({ default: '' })
    updateTime: string;
}

```



**标签**

```typescript
import { Article } from 'src/article/entities/article.entity';
import {
    Entity,
    Column,
    PrimaryGeneratedColumn,
    ManyToMany,
    DeleteDateColumn,
    Index,
} from 'typeorm';

@Entity()
export class Tag {
    @PrimaryGeneratedColumn()
    id?: number;

    @Column()
    @Index({ unique: true }) // 唯一索引
    content: string;

    @Column({ default: false })
    is_topics: boolean;

    // 对应多个 article
    @ManyToMany((type) => Article, (article) => article.tags)
    articles: Article[];

    @DeleteDateColumn()
    deleteTime: Date;
}
```

**用户**

```typescript
import { Article } from './../../article/entities/article.entity';
import {
    Entity,
    OneToMany,
    PrimaryGeneratedColumn,
} from 'typeorm';

@Entity()
export class User {
    // ...
    @OneToMany((type) => Article, (article) => article.user)
    articles: Article;
}

```



看起来也挺简单哈，写吧！

## 创建（Create）

在创建时，需要判断前端传入的标签存不存在，如果存在则保持不变，不存在则添加入库。

区分标签的方法就是看内容，因为标签不可能重复，所以我设置了唯一索引（见标签实体定义）。

```typescript
import { UserService } from './../user/user.service';
import { TagService } from './../tag/tag.service';
import { Repository } from 'typeorm';
import { Injectable, UnprocessableEntityException } from '@nestjs/common';
import { CreateArticleDto } from './dto/create-article.dto';
import { UpdateArticleDto } from './dto/update-article.dto';
import { Article } from './entities/article.entity';
import { InjectRepository } from '@nestjs/typeorm';
import * as _ from 'lodash';

@Injectable()
export class ArticleService {
    constructor(
    @InjectRepository(Article)
     private readonly repository: Repository<Article>,
     private readonly tagService: TagService,
     private readonly userService: UserService,
    ) {}
    // 新增文章方法
    async create(userId: string, createArticleDto: CreateArticleDto) {
        const articleDO = {
            title: createArticleDto.title,
            content: createArticleDto.content,
            createTime: new Date().toString(),
            updateTime: new Date().toString(),
            user: await this.userService.findUser(userId),
            tags: [],
        };
        const article = this.repository.create(articleDO);
 
        if (!_.isEmpty(createArticleDto.tags)) {
            // 文章所属tag，存在则保留，不存在即添加
            const existTags = await this.tagService.findExistTags(
                createArticleDto.tags,
            );
            const recivedTags = createArticleDto.tags.map((content) => ({ content }));
            const beInsertTags = _.xorBy(recivedTags, existTags, 'content');
            // 将需要添加到数据库的tag添加到article的tags中
            const beInsertTagEntities = beInsertTags.map((tag) => this.tagService.create(tag));
            
            // 对数据库中的数据和插入后的数据合并后去重
            article.tags = _.uniqBy(
                _.concat(beInsertTagEntities, existTags),
                'content',
            );
        }

        return await this.repository.save(article);
    }
}
```

如果前端有传入标签（tags）的话，我要判断他是否已经在数据库了，用`_.xorBy`，这个是取对称差集(symmetric_difference)，而不是差集（difference）！

关于介绍差集和对称差集的文章：[python 并集union, 交集intersection, 差集difference, 对称差集symmetric_difference_Python学习者的技术博客_51CTO博客](https://blog.51cto.com/u_14246112/3141583)

另外某lodash中文文档将`_.xor`翻译成差集，害我怀疑学的假数学！





## 读取（Retrieve）和 删除（Delete）

读取和删除都挺简单的，放在一起了

在 `article.service.ts`

```typescript
// import { Transactional, OrmContext } from '@malagu/typeorm/lib/node';
import { UserService } from './../user/user.service';
import { TagService } from './../tag/tag.service';
import { Repository } from 'typeorm';
import { Injectable, UnprocessableEntityException } from '@nestjs/common';
import { CreateArticleDto } from './dto/create-article.dto';
import { UpdateArticleDto } from './dto/update-article.dto';
import { Article } from './entities/article.entity';
import { InjectRepository } from '@nestjs/typeorm';
import * as _ from 'lodash';

@Injectable()
export class ArticleService {
    constructor(
    @InjectRepository(Article)
     private readonly repository: Repository<Article>,
     private readonly tagService: TagService,
     private readonly userService: UserService,
    ) {}
    // 查询该用户下所有文章
    findAll(user: string) {
        return this.repository.find({
            where: { user: { id: user } },
            relations: ['user', 'tags'],
        });
    }
	// 删除文章
    async remove(id: string) {
        const article = await this.repository.findOne({
            where: { id },
            relations: ['tags'],
        });
        if (_.isEmpty(article)) {
            throw new UnprocessableEntityException('文章不存在！');
        }
        return this.repository.softRemove(article);
    }
}

```



## 更新（Update）

更新这里，我想了一中午

关于存在即保留，不存在即添加，将所有情况列了出来（不止下面这些）

![image-20220521184415342](在nestjs中对文章和标签CRUD/image-20220521184415342-16531298574961.png)

觉得麻烦，遂谷歌，查出一些我知道的方法，例如`upsert`、`replace`，这些方法都存在一些问题，虽然可以插入，由于我太菜了，批量插入之后只会返回最后一个插入成功的id，所以我不知道怎么去将标签关联到文章。

以上两种方法我选择放弃，然后偶尔看到某仓库写法很巧妙，我照猫画虎的写了一个，忘记是哪个仓库了，记得是咖啡店后台管理系统来着。。。

在`article.service.ts`：

```typescript
async update(id: string, updateArticleDto: UpdateArticleDto) {
	// 这里调用 tagService 的插入方法了
    const tags = await this.tagService.insert(updateArticleDto.tags);
    const articleDO: Partial<Article> = {
        id: id,
        title: updateArticleDto.title,
        content: updateArticleDto.content,
        updateTime: new Date().toString(),
        tags,
    };
    return await this.repository.save(articleDO);
}
```

继续看 `tag.service.ts`：

```typescript
import { ArticleService } from './../article/article.service';
import { Inject, Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { In, Repository } from 'typeorm';
import { CreateTagDto } from './pojo/create-tag.dto';
import { UpdateTagDto } from './pojo/update-tag.dto';
import { Tag } from './entities/tag.entity';
import * as _ from 'lodash';
@Injectable()
export class TagService {
    constructor(
    @InjectRepository(Tag)
     private repository: Repository<Tag>,
    ) {}
    create(createTagDto: CreateTagDto) {
        return this.repository.create(createTagDto);
    }
    
    // 存在即返回，不存在即创建
    async findOrCreate(tag: string) {
        let tagEntity = await this.repository.findOne({
            where: { content: tag },
        });
        if (!tagEntity) {
            tagEntity = await this.repository.save({ content: tag });
        }
        return tagEntity;
    }

    // 插入 tag
    async insert(tags: string[]) {
        const tagEntities = [];
        for (const tag of tags) {
            // 统一 save 处理
            const tagEntity = await this.findOrCreate(tag);
            await this.repository.save(tagEntity);
            tagEntities.push(tagEntity);
        }
        return tagEntities;
    }
}
```

这里 `findOrCreate` 方法一定会返回一个数据库实体，而 `repository.save` 一定会返回创建好后的对象（无论存在冲突或正常插入），这样我就可以平稳的拿到所有插入后的实体，然后附加到文章关系上，也不用担心对新旧标签插入还是删除。



## 总结

今天一个增删改查写了一中午，其实这个存在即更新，不存在即插入，这个问题之前工作遇到过，当时是手动diff的，也不知道有`replace、upsert`这些。

感觉自己死脑筋，跳不出思维定势出来，有些问题也不止有一种解法，换个切入点或许效率更高。





