---
title: uniapp资源引用
date: 2022-12-08 06:12:59
tags:
- coding
---

小程序有打包资源大小限制2M，像是图片这种超级大的文件一般不会存到代码上面，通常放到服务器上。但是开发的时候又不想麻烦，还是想用本地的图片。

<!--more-->

vite 打包器里有两种文件夹，assets 和 static *(public)*，区别在于：

- assets 被import引用后，rollup 可以对图片等其他资源做处理（压缩，转base64等），没有引用则不会打包进dist 
- static *(public)* 目录通常一般不用import引入，（在img的src或其他）使用绝对路径引入，例如`/images/1.png`代表`/static/images/1.png`，rollup 不会对该目录处理，而是整体将该目录完封不动地移动到dist目录。

好消息是，uniapp可以用vite打包，但是在uniapp环境里，将图片放到static目录下，直接写绝对路径小程序不认。

使用 `imgResolver`：

```javascript
import { getUploadUrl } from '@/utils/env'

const url = getUploadUrl()
export function imgResolver(path: string) {
  return url + path
}

/**
 * @description: Get environment VITE_UPLOAD_URL value
 * @returns:
 * @example:
 */
export function getUploadUrl(): string {
  return getEnvValue<string>('VITE_UPLOAD_URL');
}


/**
 * @description: Get environment variables
 * @returns:
 * @example:
 */
export function getEnvValue<T = any>(key: string): T {
  // @ts-ignore
  return import.meta.env[key];
}
```

区分不同的打包模式，设置不同的`VITE_UPLOAD_URL`

```bash
# .env.production
VITE_UPLOAD_URL='https://sunzehui-bucket.aliyuncs.com/'

# .env.development
VITE_UPLOAD_URL='/static'
```

使用时，如果是小icon那么直接放到assets使用import引入

```javascript
import imgBackIcon from '@/assets/icon/back.png'
```

如果是背景图（体积很大的图片）则使用`imgResolver`

```javascript
import { imgResolver } from '@/utils/imgResolver';
const imgNavBg = imgResolver('/images/nav-bg.png');
```

开发时编译路径：

`/static/images/nav-bg.png`

打包后路径

`https://sunzehui-bucket.aliyuncs.com/images/nav-bg.png`

上传源码时将static的nav-bg.png清掉就可以了。





更多阅读：

https://www.jianshu.com/p/cbe5459fba63

