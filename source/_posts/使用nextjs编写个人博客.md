---
title: 使用nextjs编写个人博客
tags:
  - coding
abbrlink: f2f67e66
date: 2023-07-24 07:06:32
---
MDX是一种将Markdown和JSX结合的语法，可以在Markdown中使用JSX，一直想独立编写一个博客平台，这次就用nextjs+MDX来写一个博客平台吧。
<!--more-->

## 使用Next.js开发博客平台的主要步骤

1. **使用Next.js创建项目**

    使用`pnpm create next-app`命令可以快速创建一个Next.js项目。

2. **设计目录结构**

    使用Next.js提供的app目录结构。

3. **实现Markdown解析**

    可以使用`next-mdx-remote`等MDX解析库,将MDX文件解析为ReactNode。

4. **静态页面生成**

    使用`generateStaticParams`实现文章静态页生成。

5. **添加样式**

    可以使用SASS或tailwindcss给博客及mdx元素添加样式。 

6. **部署上线**

    可以部署到Vercel、Netlify等平台上。

Next.js的静态生成和路由功能可以方便实现一个简单的博客平台。

## 项目初始化
```bash
pnpm create next-app
```
默认创建的项目是nextjs 13.4版本(2023-7-20)，是最新使用`app router`的版本，相较于之前`page router`有些改动。

## 目录结构
由于我的文章文件名都是中文的，不利于网页的SEO，所以我为每一个文章通过文件名生成唯一id，使用id作为文章的路由。
目录结构大致是：
```
app:
├─(site)
│  │  layout.tsx
│  │  page.tsx
│  │
    # 文章列表页
│  └─post
│      │  header.tsx
│      │  layout.tsx
│      │  page.tsx
│      │
        # 文章详情页
│      └─[slug]
│              not-found.tsx
│              page.tsx
│
└─data
        # 解析mdx文件
        post.ts
```
src:
```
├─assets
│  │  globals.scss
│  │  md-el.scss
│  │
│  └─icons
│
├─components

├─lib
│      mdx.ts
│
├─posts
│      《函数式编程指北》练习题速通指南.mdx
│      使用chatgpt编写小说.mdx
│
└─utils
```

## 实现Markdown解析
实现了两个函数，获取所有meta数据和获取文章内容。
```ts
const rootDirectory = path.join(process.cwd(), 'src', 'posts')

// 通过文件名获取文章
export const getPostBySlug = async (slug: string) => {
  const realSlug = slug.replace(/\.mdx$/, '')
  const filePath = path.join(rootDirectory, `${realSlug}.mdx`)

  const fileContent = fs.readFileSync(filePath, { encoding: 'utf8' })
  const { data, content } = matter(fileContent)

  const id = crc32(realSlug).toString(16)

  return { meta: { ...data, slug: realSlug, id, readingTime: readingTime(content) }, content: fileContent } as PostDetail
}

// 获取所有文章meta数据
export const getAllPostsMeta = async () => {
  const files = fs.readdirSync(rootDirectory)

  let metas = await Promise.all(files.map(async filename => {
    const { meta } = await getPostBySlug(filename)
    return meta
  }))
  return metas
}

// 通过id获取文章详情
export const getPost = async (id: string) => {
  // id to slug
  const slug = await getPostName(id);
  if (!slug) throw new Error('not found')
  const post = await getPostBySlug(slug)
  return post
}
```
关于生成文章唯一id，我使用了crc32算法，这个算法的特点是速度快，生成的id短，但是会有冲突，不过我觉得这个冲突概率很小，而且我也不会写那么多文章，所以就这样吧。


- 对于文章列表，直接使用`getAllPostsMeta`获取所有文章的meta数据，然后渲染列表即可。
```tsx
interface PostProps {
  id: number | string
  title: string
  desc: string
  tags: string[]
  date: string
}
const Post = ({ ...props }: PostProps) => {
  const { title, desc: content, tags, date: time, id } = props

  return (
    <div className="post-card">
      <Link href={`/post/${id}`}>
        <div className="post-title">
          {title}
        </div>
      </Link>
      <div className="post-content">
        {content}
      </div>
      <div className="post-footer">
        <TimeBar time={time} />
        <div className="post-footer__tag">
          {
            tags.length <= 0 && <Tag val={'未归档'} />
          }
          {
            tags.map((tag, idx) => {
              if (idx > 3) return null;
              return <Tag key={tag} val={tag} />
            })
          }
          {
            tags.length > 3 && <Tag isMore />
          }
        </div>
      </div>
    </div>
  )
}


export default async function PagePostList() {
  const posts = await getPostList()

  return (
    <div className="post-list">
      {
        posts.map(post => {
          return <Post key={post.id} {...post} />
        })
      }
    </div>
  )
}
```
- 对于文章详情页，通过id获取文章内容，将内容传入`MDXRemote`组件中，然后渲染即可。
```tsx
import { MDXRemote } from 'next-mdx-remote/rsc'

export default async function Post({
  params,
}: {
  params: { slug: string };
}) {
  try {
    const postPayload = await getPost(params.slug)

    return <>
      <div className="post-detail">
        {/* ... */}
        {/* 文章内容 */}
        <div className="markdown-body w-full break-words whitespace-normal ">
          <Suspense fallback={<>Loading...</>}>
            {/* @ts-ignore */}
            <MDXRemote source={postPayload.content} options={{ parseFrontmatter: true, }} components={components} />
          </Suspense>
        </div>
       {/* ... */} 
      </div>
    </>
  } catch (e) {
    return notFound()
  }
}
```

## 静态页面生成
在`post/[slug]/page.tsx`中，通过导出`generateStaticParams`函数，可以实现静态页面生成(app router)。
在app router中，生成静态页面路径的函数已经由原来的`getStaticPaths`改为`generateStaticParams`
```tsx
export async function generateStaticParams() {
  const metas = await getAllPostsMeta()
  return metas.map(meta => {
    return { slug: meta.id }
  })
}
```

## 添加样式

mdx文件中的元素，可以通过`components`属性传入`<MDXRemote />`组件，然后在`components`对象中添加样式。
```tsx
const components = {
  // 代码高亮
  code: CodeBlock,
  img: Image,
  // 为其元素添加自定义类名
  h1: tagRenderer('h1'),
  h2: tagRenderer('h2'),
  h3: tagRenderer('h3'),
  h4: tagRenderer('h4'),
  h5: tagRenderer('h5'),
  h6: tagRenderer('h6'),
  p: tagRenderer('p'),
  a: tagRenderer('a'),
  blockquote: tagRenderer('blockquote'),
  table: tagRenderer('table'),
  thead: tagRenderer('thead'),
  tbody: tagRenderer('tbody'),
  tr: tagRenderer('tr'),
  th: tagRenderer('th'),
  td: tagRenderer('td'),
  em: tagRenderer('em'),
  strong: tagRenderer('strong'),
  del: tagRenderer('del'),
}
```

## 总结
app router用起来感觉别扭，比如文章生成那块，默认的MDXRemote组件并不是`RSC(react-server-component)`，导致外面引用该组件的组件必须使用'use client'，因为nextjs在服务端渲染不了useState函数，这样就导致了组件的复用性变差，不过这个问题可以通过自定义MDXRemote组件解决，官方示例有提供方法，这样就不如page router方便了。

另外，使用motion添加动画效果的时候，必须添加'use client'，这样就导致了动画效果在服务端渲染的时候不会生效，这个问题我还没想到解决方法。
