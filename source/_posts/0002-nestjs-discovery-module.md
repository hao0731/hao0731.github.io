---
title: 你可能不知道的 NestJS 隱藏技巧：Discovery Module
date: 2023-09-17 17:35:00
tags:
  - Backend
  - NestJS
categories:
  - ['Backend', 'NestJS', 'Advanced']
---

在某些應用場景下，可能會需要去遍歷封裝於 **模組（Module）** 內的元件，比如：找出帶有特定 **裝飾器（Decorator）** 的元件，甚至是元件底下的方法，來預先處理一些事情，最典型的案例就是 `EventEmitterModule`，當某個事件觸發時，會呼叫帶有特定裝飾器的方法。

> **NOTE**：關於 `EventEmitterModule` 可以參考[官方文件](https://docs.nestjs.com/techniques/events)的說明。

下方是官方 `EventEmitterModule` 的範例，透過 `EventEmitter2` 發送 `order.created` 事件時，會呼叫帶有 `@OnEvent` 裝飾器且值為 `order.created` 的方法：

```typescript
this.eventEmitter.emit(
  'order.created',
  new OrderCreatedEvent({
    orderId: 1,
    payload: {},
  }),
);
```

```typescript
// ...
@Injectable()
export class OrderListener {
  // ...
  @OnEvent('order.created')
  handleOrderCreatedEvent(payload: OrderCreatedEvent) {
    // handle and process "OrderCreatedEvent" event
  }
}
```

那麼 `EventEmitterModule` 是如何做到這件事情的呢？它是透過一個叫 `DiscoveryModule` 的模組來找出所有元件底下含有 `@OnEvent` 裝飾器的方法，並根據帶入的值，來決定該方法在哪個事件下會被觸發。

> **NOTE**：`DiscoveryModule` 並沒有收錄在 NestJS 官方文件中。

## 深入 Discovery Module

> **NOTE**：以下範例採用 NestJS 10 來撰寫。

`DiscoveryModule` 是一個 NestJS 內建的模組，無須安裝套件，使用方式如下：

```typescript
import { Module } from '@nestjs/common';
import { DiscoveryModule } from '@nestjs/core';
// ...

@Module({
  // ...
  imports: [DiscoveryModule],
  // ...
})
export class AppModule {}
```

引入模組後，可以透過 `DiscoveryService` 來取得封裝於模組底下的 Controller 或 Provider：

```typescript
import { Module } from '@nestjs/common';
import { DiscoveryModule, DiscoveryService } from '@nestjs/core';
// ...

@Module({
  // ...
  imports: [DiscoveryModule],
  // ...
})
export class AppModule implements OnModuleInit {
  constructor(
    private readonly discoveryService: DiscoveryService
  ) {}

  onModuleInit() {
    // 遍歷所有模組，以取得所有 Controller
    const controllers = this.discoveryService.getControllers();
    // 遍歷所有模組，以取得所有 Provider
    const providers = this.discoveryService.getProviders();
  }
}
```

這裡需特別注意，取得的 **不是** Controller、Provider 本身，而是一個型別為 `InstanceWrapper` 的 Wrapper，若要拿到它們的本身的 **實例（Instance）**，只需要透過 `instance` 屬性即可取得，如下所示：

```typescript
// 將 instance 從 `InstanceWrapper` 取出
const instances = this.discoveryService.getControllers().map(({ instance }) => instance);
```

### 限縮遍歷範圍

如果想要限制遍歷的模組範圍，`getControllers` 跟 `getProviders` 有提供相關參數，透過指定 `include` 來決定要遍歷哪些模組：

```typescript
import { Module, OnModuleInit } from '@nestjs/common';
import { DiscoveryModule, DiscoveryService } from '@nestjs/core';
// ...

@Module({
  // ...
  imports: [TodoModule, DiscoveryModule],
  // ...
})
export class AppModule implements OnModuleInit {
  constructor(
    private readonly discoveryService: DiscoveryService
  ) {}

  onModuleInit() {
    // 遍歷 `TodoModule`底下的元件，以取得底下的所有 Provider
    const providers = this.discoveryService.getProviders({
      include: [TodoModule],
    });
    // 遍歷 `TodoModule`底下的元件，以取得底下的所有 Controller
    const controllers = this.discoveryService.getControllers({
      include: [TodoModule],
    });
  }
}
```

### 過濾別名 Provider 的技巧

由於 `getProviders` 會拿到所有 Provider，所有裡面會含有 Alias Provider，在某些情境下有可能會導致相同的東西被處理一次以上，所以在預處理前，要先進行過濾：

```typescript
import { Module, OnModuleInit } from '@nestjs/common';
import { DiscoveryModule, DiscoveryService } from '@nestjs/core';
// ...

@Module({
  // ...
  imports: [DiscoveryModule],
  // ...
})
export class AppModule implements OnModuleInit {
  constructor(
    private readonly discoveryService: DiscoveryService,
    private readonly metadataScanner: MetadataScanner
  ) {}

  onModuleInit() {
    const providers = this.discoveryService.getProviders();
    const controllers = this.discoveryService.getControllers();
    [...providers, ...controllers]
      // 根據 `instance` 是否存在以及 `isAlias` 為 `false` 來過濾 Alias Provider
      .filter((wrapper) => wrapper.instance && !wrapper.isAlias)
      .forEach((wrapper) => {
        // do something
      });
  }
}
```

## 與 Metadata Scanner 共舞

現在知道要如何透過 `DiscoveryModule` 遍歷所有元件了，那有什麼方法可以取得元件底下所有的方法呢？NestJS 有提供一個叫 `MetadataScanner` 的 Provider，讓我們可以去掃描元件下的所有方法，使用方式如下：

```typescript
import { Module, OnModuleInit } from '@nestjs/common';
import { MetadataScanner } from '@nestjs/core';
// ...

@Module({
  // ...
})
export class AppModule implements OnModuleInit {
  constructor(
    private readonly metadataScanner: MetadataScanner
  ) {}

  onModuleInit() {
    const instance = new Component();
    // 取得元件底下的所有方法名稱
    const methodNames = this.metadataScanner.getAllMethodNames(instance);
  }
}
```

那麼加上 `DiscoveryModule`，就可以遍歷所有元件底下的方法名稱了：

```typescript
import { Module, OnModuleInit } from '@nestjs/common';
import {
  DiscoveryModule,
  DiscoveryService,
  MetadataScanner
} from '@nestjs/core';
// ...

@Module({
  // ...
  imports: [DiscoveryModule],
  // ...
})
export class AppModule implements OnModuleInit {
  constructor(
    private readonly discoveryService: DiscoveryService,
    private readonly metadataScanner: MetadataScanner
  ) {}

  onModuleInit() {
    const providers = this.discoveryService.getProviders();
    const controllers = this.discoveryService.getControllers();

    [...providers, ...controllers]
      .filter((wrapper) => wrapper.instance && !wrapper.isAlias)
      .forEach((wrapper) => {
        const { instance } = wrapper;
        const methodNames = this.metadataScanner.getAllMethodNames(instance);
      });
  }
}
```

### 搭配 Reflector 打出連續技

假設現在需要抓取所有元件下帶有 `HelloWorld` 裝飾器的方法，可以運用 `DiscoveryModule` 先遍歷所有的元件，再透過 `MetadataScanner` 掃出每個元件下的方法名稱，最後再使用 `Reflector` 篩選出最終結果。

假設現在有一個 `@HelloWorld` 裝飾器：

```typescript
import { SetMetadata } from '@nestjs/common';

export const HELLO_WORLD_KEY = 'custom:hello-word';

export const HelloWorld = () => SetMetadata(HELLO_WORLD_KEY, 'Hello World');
```

並且只在 `TodoModule` 底下的 `TodoController` 中使用：

```typescript
import { Controller, Get } from '@nestjs/common';
// ...

@Controller('todos')
export class TodoController {
  @HelloWorld()
  @Get()
  getTodos() {
    return [];
  }
}
```

這時可以運用 `Reflector` 的 `get` 方法，來判斷元件底下的方法是否有使用 `@HelloWorld` 裝飾器：

```typescript
import { Module, OnModuleInit } from '@nestjs/common';
import {
  DiscoveryModule,
  DiscoveryService,
  MetadataScanner,
  Reflector,
} from '@nestjs/core';
// ...

@Module({
  // ...
  imports: [TodoModule, DiscoveryModule],
  // ...
})
export class AppModule implements OnModuleInit {
  constructor(
    private readonly discoveryService: DiscoveryService,
    private readonly metadataScanner: MetadataScanner,
    private readonly reflector: Reflector,
  ) {}

  onModuleInit() {
    const providers = this.discoveryService.getProviders();
    const controllers = this.discoveryService.getControllers();
    
    [...providers, ...controllers]
      .filter((wrapper) => wrapper.instance && !wrapper.isAlias)
      .forEach((wrapper) => {
        const { instance } = wrapper;
        const methodNames = this.metadataScanner.getAllMethodNames(instance);
        methodNames
          .filter(
            (methodName) =>
              this.reflector.get<string>(
                HELLO_WORLD_KEY,
                instance[methodName],
              ) === 'Hello World',
          )
          .forEach((methodName) => {
            console.log(methodName); // 'getTodos'
          });
      });
  }
}
```

## 結論

`DiscoveryModule` 是一個蠻好用的內建模組，尤其是針對一些事件驅動的情境特別適合，比如說：使用第三方的 SDK，它收到某個事件時可以呼叫我們帶有特定裝飾器的方法等。

## 參考資料

- [EventEmitterModule](https://github.com/nestjs/event-emitter/tree/master)
