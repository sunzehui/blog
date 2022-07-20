---
title: 在nestjs中使用websocket并实现登录鉴权
tags:
  - nestjs
abbrlink: d0e90396
date: 2022-07-19 11:08:27
---

考试系统需要记录用户作答时间等等，选择使用websocket，并验证用户token。



<!--more-->

最近在忙着写毕业论文，选题《基于nodejs的在线考试系统》，九月截稿，现在投了开题报告，万幸没有被退回来。

用了一周时间把教师端做得差不多了，不过还有些统计的API，都是细节，先把框架跑通。有时间总结一下。

![image-20220719112455264](在nestjs中使用websocket并实现登录鉴权/image-20220719112455264.png)

现在需要搞`考场状态查询`之类的，简单点就手动轮询了，它是种反模式，辩证使用吧。

我的情况是：业务复杂，需要实时监测考生作答情况，所以采用`websocket`





## 实现

首先用脚手架创建CURD模块，选择`WebSockets`

```bash
# examination_backend/
❯ nest g res exam-clock
? What transport layer do you use?
  REST API
  GraphQL (code first)
  GraphQL (schema first)
  Microservice (non-HTTP)
> WebSockets
```

在`exam-clock.gateway.ts`中，在连接后验证token

```typescript
import {
    WebSocketGateway,
    SubscribeMessage,
    MessageBody,
    OnGatewayDisconnect,
    WebSocketServer,
    OnGatewayConnection,
    ConnectedSocket,
} from '@nestjs/websockets';
import { UnauthorizedException } from '@nestjs/common';
import { Socket, Server } from 'socket.io';

export class ExamClockGateway
    implements OnGatewayConnection
{
    constructor(private readonly examClockService: ExamClockService) {}

    async handleConnection(socket: Socket) {
        try {
            // 此处调用 service 验证token
            const user = await this.examClockService.getUserFromSocket(socket);
            socket.emit('connected', user);
        } catch (e) {
            // 验证失败
            return ExamClockGateway.disconnect(socket);
        }
    }
}
```

在`exam-clock.service.ts`中，通过token验证用户信息

```typescript
@Injectable()
export class ExamClockService {
    constructor(
     private readonly authService: AuthService,
     private readonly userService: UserService,
    ) {}

    async getUserFromSocket(socket): Promise<User> {
        // 在 socket 中拿到消息头
        const token = socket.handshake.headers.authorization;
        // 调用 authService 提取 token payload
        const decodedToken = await this.authService.verifyJwt(token);
        // 去数据库查用户信息
        const user = await this.userService.findOneById(+decodedToken.id);
        if (!user) {
            throw new UnauthorizedException();
        }
        return user;
    }
}
```

上面提取headers这个写法是`socket.io`提供的。

添加查询某用户所有考场信息服务，该服务使用`ExamRoomModule`暴露出来的方法

```typescript
// exam-lock.service.ts
import { ExamRoomService } from '@/exam-room/exam-room.service';

@Injectable()
export class ExamClockService {
    constructor(
    private readonly examRoomService: ExamRoomService
    ) {}

    async findAll(userId: number) {
        return await this.examRoomService.findAll(userId);
    }
}
```

在`exam-lock.gateway.ts`中，获取该用户信息并传入`service`

```typescript
export class ExamClockGateway
    implements OnGatewayConnection
{
    constructor(private readonly examClockService: ExamClockService) {}

    @SubscribeMessage('findAllExamClock')
    async findAll(@ConnectedSocket() socket) {
        const user = await this.examClockService.getUserFromSocket(socket);
        const result = await this.examClockService.findAll(+user.id);
        return ResultData.ok(result);
    }
}

```

到这里查询某用户所有考场信息的后端程序就写完了，继续看前端

前端很简单，用`vite`创建了个原生项目

```javascript
import { io } from "socket.io-client";

const socket = io("http://localhost:3000", {
    extraHeaders: {
        authorization: "token"
    },
});
socket.emit("findAllExamClock", console.log);

socket.on("Error", console.log);

socket.on("connected", console.log);
```

关于 extraHeaders：[Client options | Socket.IO](https://socket.io/docs/v4/client-options/#extraheaders)



![image-20220719120942731](在nestjs中使用websocket并实现登录鉴权/image-20220719120942731.png)

## 优化

现在来优化一下代码：

在`exam-clock.service.ts`中，通过token验证用户信息，如果有多个`gateway`，每个`gateway`都需要验证用户信息，这样会造成很多冗余，且不利于新增`feature`后维护，所以将验证用户信息提取出来，我的思路：

写个`guard`（守卫），验证`user`后挂到`socket`上，为了处理异常，创建一个全局异常过滤器，这个是基于`BaseWsExceptionFilter`的，跟`HTTP`还不太一样。



验证用户信息守卫

```typescript
// exam-clock/guards/authGuard
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { WsException } from '@nestjs/websockets';
import * as _get from 'lodash/get';
import { AuthService } from '@/common/module/auth/auth.service';
import { UserService } from '@/common/module/user/user.service';

@Injectable()
export class AuthGuard implements CanActivate {
  constructor(
    private readonly authService: AuthService,
    private readonly userService: UserService,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const socket = context.switchToWs().getClient();
    const token = _get(socket, 'handshake.headers.authorization');
    try {
      const decodedToken = await this.authService.verifyJwt(token);
      socket.user = await this.userService.findOneById(+decodedToken.id);
    } catch (e) {
      throw new WsException(e.message);
    }

    return true;
  }
}
```

全局异常拦截器

```typescript
import { TokenExpiredError, verify } from 'jsonwebtoken';
import {
  ArgumentsHost,
  Catch,
  UnauthorizedException,
} from '@nestjs/common';
import { BaseWsExceptionFilter, WsException } from '@nestjs/websockets';
import { Socket } from 'socket.io';

@Catch(TokenExpiredError, UnauthorizedException)
export class UnauthorizedErrorFilter extends BaseWsExceptionFilter {
  catch(
    exception: TokenExpiredError | UnauthorizedException,
    host: ArgumentsHost,
  ) {
    const client = host.switchToWs().getClient() as Socket;
    const error =
      exception instanceof WsException ? exception.getError() : exception;
    const details = error instanceof Object ? { ...error } : { message: error };
    client.emit('exception', details);
  }
}
```

在`gateway`中使用

```typescript
import { ExamClockService } from './exam-clock.service';
import { UnauthorizedException, UseFilters, UseGuards } from '@nestjs/common';
import { ResultData } from '@/common/utils/result';
import { AuthGuard } from '@/exam-clock/guards/auth.guard';
import { UnauthorizedErrorFilter } from '@/common/filter/unauthorized.filter';

// 使用过滤器（也可以放到全局）
@UseFilters(new UnauthorizedErrorFilter())
export class ExamClockGateway {
    // ...省略
    
    // jwt守卫
    @UseGuards(AuthGuard)
    @SubscribeMessage('findAllExamClock')
    async findAll(@User('id') userId: string) {
        const result = await this.examClockService.findAll(+userId);
        return ResultData.ok(result);
    }
}
```

为方便提取`userId`，遂写装饰器：User

```typescript
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const User = createParamDecorator((data: any, ctx: ExecutionContext) => {
    const req = ctx.switchToHttp().getRequest();
    // if route is protected, there is a user set in auth.middleware
    if (!!req.user) {
        return !!data ? req.user[data] : req.user;
    }
});
```

可以跑通，效果同上



## 总结

参考：[API with NestJS #26. Real-time chat with WebSockets (wanago.io)](https://wanago.io/2021/01/25/api-nestjs-chat-websockets/)