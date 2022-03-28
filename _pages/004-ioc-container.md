---
title: 4. 제어의 역전 컨테이너 (with InversifyJS)
author: Myeongjae Kim
date: 2022-03-12
category: Tutorial
layout: post
---

클래스들의 의존관계에 대한 정보를 코드상에서 다루는 방법이 있을까요? 다른 말로 이야기하면, 프로그램(=클래스)을 프로그래밍할 수 있을까요? 이런 방식을
메타프로그래밍[^1]이라고 부릅니다. 메타프로그래밍의 일종인 리플렉션[^2]을 사용하면 클래스의 인스턴스를 생성하는 작업(=`new`)을 일반화할 수 있습니다.

```typescript
const context = createApplicationContext();

/*
const applicationByStateManager = new ApplicationByStateManager(
  context.stateManager,
  context.articleQueryViewController,
  context.articleCommandViewController
);
*/

const clazz = ApplicationByStateManager;
const arg0 = context.stateManager;
const arg1 = context.articleQueryViewController;
const arg2 = context.articleCommandViewController;
const applicationByStateManager = Reflect.construct(clazz, [arg0, arg1, arg2]);

applicationByStateManager.run();
```

`ApplicationByStateManager` 클래스의 인스턴스를 `new` 예약어 대신 `Reflect.consturct` 함수를 사용해서 생성했습니다.
클래스는 `clazz`에, 생성자 매개변수는 `arg0`, `arg1`, `arg2`에 할당하고 `Reflect.construct(clazz, [arg0, arg1, arg2])`를 호출합니다.

어떤 방법으로든 **클래스**와 **생성자에 필요한 매개변수**에 접근할 수 있으면 리플렉션으로 인스턴스를 생성할 수 있습니다. 생성한 인스턴스가 **다른 클래스 생성자의
몇 번째 매개변수**로 들어간다는 정보만 추가로 알 수 있으면 우리는 애플리케이션 컨텍스트를 생성할 때 했던 작업들을 자동화할 수 있습니다. 클래스 인스턴스의
조립을 담당하는 컨테이너 클래스를 만들고, 컨테이너 클래스의 인스턴스에 개별 클래스들을 등록하고 이 클래스의 인스턴스들이 어떻게 엮이게 되는지까지의 정보를
제공하면 됩니다.

## InversifyJS

이미 똑똑한 누군가가 클래스들을 조립하는 컨테이너를 미리 만들어놓지 않았을까요? 자바스크립트 진영에는 InversifyJS라는 걸출한 제어의 역전 컨테이너가
있습니다. '제어의 역전'이라는 단어에 대해서는 추후에 말씀드리겠습니다. 일단 제어의 역전 컨테이너는 설계도대로 조립해주는 기계라고 생각하시면 됩니다.

<figure>
    <img alt="Npmtrends of IoC Contaienrs" src="/res/9-ioc-containers.png"/>
    <figcaption>
        <a href="https://www.npmtrends.com/inversify-vs-tsyringe-vs-typedi-vs-typescript-ioc" target="_blank">https://www.npmtrends.com/inversify-vs-tsyringe-vs-typedi-vs-typescript-ioc</a>
    </figcaption>
</figure>

### 인스턴스 생성하기

지금의 목표는 리플렉션으로 생성했던 `ApplicationByStateManager` 인스턴스를 InversifyJS의 제어의 역전 컨테이너를 통해서 생성하는 것입니다.
`inversify`와 `reflect-metadata` 의존성을 추가합니다.

```bash
npm install inversify reflect-metadata
```

그리고 `tsconfig.json`에서 `experimentalDecorators`, `emitDecoratorMetadata` 옵션을 켜줍니다.

```json
{
  ...,
  "experimentalDecorators": true,
  "emitDecoratorMetadata": true
}
```

`Constants`클래스에 `SERVICE_ID`라는 스태틱 프로퍼티를 추가합니다. `SERVICE_ID`는 제어의 역전 컨테이너 안에서 객체들을 관리하기 위해 사용하는
식별자가 됩니다.

```typescript
import os from "os";

export class Constants {
  private constructor() {
    // static 값용 클래스
  }

  public static LINE_BREAK = os.EOL;
  public static GO_BACK_COMMAND = "x";

  public static SERVICE_IDS = {
    StateManager: "StateManager",
    ArticleQueryViewController: "ArticleQueryViewController",
    ArticleCommandViewController: "ArticleCommandViewController",
    ApplicationByStateManager: "ApplicationByStateManager",
  };
}
```

`ApplicationByStateManager.ts`에 `import "reflect-metadata"`를 추가하고 클래스와 생성자 매개변수에 데코레이터를 달아줍니다.

```typescript
import { ArticleCommandViewController } from "./article/view/cli/ArticleCommandViewController";
import { ArticleQueryViewController } from "./article/view/cli/ArticleQueryViewController";
import { StateManager } from "./article/view/cli/state-modules/vanila/StateManager";
import "reflect-metadata";
import { inject, injectable } from "inversify";
import { Constants } from "./Constants";

@injectable()
export class ApplicationByStateManager {
  constructor(
    @inject(Constants.SERVICE_IDS.StateManager)
    private readonly stateManager: StateManager,
    @inject(Constants.SERVICE_IDS.ArticleQueryViewController)
    private readonly cliQueryController: ArticleQueryViewController,
    @inject(Constants.SERVICE_IDS.ArticleCommandViewController)
    private readonly cliCommandController: ArticleCommandViewController
  ) {}

  public run = async () => {
  ...
}
```

클래스에 달려있는 `@injectable()` 데코레이터는 이 클래스가 다른 클래스에 주입된다는 의미도 있지만, 이 클래스의 인스턴스를 제어의 역전 컨테이너 안에서
생성하고 관리한다는 의미도 있습니다. 생성자 매개변수의 `@inject()` 데코레이터는 생성자 매개변수에 어떤 클래스 인스턴스를 주입해야 하는지를 나타냅니다.

이제 `src/index.ts`에서 `SERVICE_ID`와 인스턴스를 짝지어주기만 하면 `ApplicationByStateManager` 인스턴스를 제어의 역전 컨테이너에서 생성할
수 있게 됩니다.

```typescript
import { Container } from "inversify";
import { ApplicationByStateManager } from "./ApplicationByStateManager";
import { createApplicationContext } from "./applicationContext";
import { Constants } from "./Constants";

const context = createApplicationContext();

/*
const applicationByStateManager = new ApplicationByStateManager(
  context.stateManager,
  context.articleQueryViewController,
  context.articleCommandViewController
);
*/

const bind = (container: Container) => {
  container
    .bind(Constants.SERVICE_IDS.StateManager)
    .toConstantValue(context.stateManager);
  container
    .bind(Constants.SERVICE_IDS.ArticleQueryViewController)
    .toConstantValue(context.articleQueryViewController);
  container
    .bind(Constants.SERVICE_IDS.ArticleCommandViewController)
    .toConstantValue(context.articleCommandViewController);
  container
    .bind(Constants.SERVICE_IDS.ApplicationByStateManager)
    .to(ApplicationByStateManager);
};

// create IoC Container and bind Constants.SERVICE_IDS to instances or classes
const container = new Container();
bind(container);

// get the application
const applicationByStateManager = container.get<ApplicationByStateManager>(
  Constants.SERVICE_IDS.ApplicationByStateManager
);

applicationByStateManager.run();
```

`bind` 함수를 보면 `toConstantValue()`를 호출해서 우리가 이전에 만들었던 애플리케이션 컨텍스트에서 생성한 인스턴스를 `SERVICE_ID`에 짝지어줍니다.
그리고 마지막으로 `SERVICE_ID.ApplcationByStateManager`에 `ApplicationByStateManager` 클래스를 짝지어주면 제어의 역전 컨테이너에서
`ApplicationByStateManager` 클래스의 인스턴스를 생성하기 위한 모든 정보를 컨테이너에 제공하게 됩니다.

`container.get()`을 호출해서 컨테이너 안에서 애플리케이션 인스턴스를 찾아서 `run()` 메서드를 호출하면 애플리케이션이 정상적으로 작동하는 것을 확인할
수 있습니다: [nodejs-tutorial-example:chapter-4-ioc-with-decorator](https://github.com/myeongjae-kim/nodejs-tutorial-example/tree/chapter-4-ioc-with-decorator)

### 데코레이터가 없었으면 좋겠는데

`ApplicationByStateManager` 클래스에 직접 데코레이터를 달면서 `ApplicationByStateManager.ts` 파일이 InversifyJS를 의존하게
되었습니다. 다음 작업으로 애플리케이션 컨텍스트에서 수동으로 생성한 인스턴스들도 제어의 역전 컨테이너를 통해 생성하려고 하는데요, 클래스에 직접 데코레이터를
추가하는 방식을 사용하면 `src/article/application` 디렉토리 안에서도 InversifyJS를 의존하게 됩니다. 비즈니스 로직에 직접적인 영향을 주는 것은
아니지만 InversifyJS라는 세부사항이 도메인 영역으로 침투하기 때문에 육각형 안쪽의 순수함이 오염되어버립니다.

데코레이터는 클래스들의 관계에 대한 정보를 제어의 역전 컨테이너에 제공하기 위한 수단입니다. 어떤 방법을 사용하든 제어의 역전 컨테이너에 이 정보들을 제공하기만
하면 컨테이너에서 클래스들을 조립해서 인스턴스들을 생성할 수 있습니다. 다행히 InversifyJS는 타입스크립트 없이 자바스크립트만으로도 작동할 수 있게 하기 위해
`@`를 사용하는 데코레이터 문법(타입스크립트에만 존재합니다) 없이도 클래스들의 관계를 컨테이너에게 알려줄 수 있는 방법이 있습니다:
[https://github.com/inversify/InversifyJS/blob/master/wiki/basic_js_example.md](https://github.com/inversify/InversifyJS/blob/master/wiki/basic_js_example.md)

```typescript
const decorateClasses = () => {
  decorate(injectable(), ApplicationByStateManager);
  decorate(
    inject(Constants.SERVICE_IDS.StateManager),
    ApplicationByStateManager,
    0
  );
  decorate(
    inject(Constants.SERVICE_IDS.ArticleQueryViewController),
    ApplicationByStateManager,
    1
  );
  decorate(
    inject(Constants.SERVICE_IDS.ArticleCommandViewController),
    ApplicationByStateManager,
    2
  );
};
```

거창하게 말했지만... 클래스에 데코레이터를 직접 달지 말고 바깥에서 달아주면 됩니다. 이제 `tsconfing.json`의 데코레이터 관련 옵션을 다시 꺼버려도 됩니다.
마찬가지로 동일하게 작동이 잘 된다는 것을 확인할 수 있습니다: [nodejs-tutorial-example:chapter-4-ioc-without-decorator:src/index.ts](https://github.com/myeongjae-kim/nodejs-tutorial-example/blob/chapter-4-ioc-without-decorator/src/index.ts) 

### `SERVICE_ID`에 대해서

InvresifyJS에서는 제어의 역전 컨테이너 안의 인스턴스들을 관리하기 위해 부여하는 식별자를 `ServiceIdentifier`라고 부릅니다.
`ServiceIdentifier`는 `string`, `symbol`, `Newable<T>`, `Abstract<T>`의 타입을 가질 수 있습니다: [https://github.com/inversify/InversifyJS/blob/772ea8ef53b17ac00a35df45d00bfc2f1ca53d07/src/interfaces/interfaces.ts#L47](https://github.com/inversify/InversifyJS/blob/772ea8ef53b17ac00a35df45d00bfc2f1ca53d07/src/interfaces/interfaces.ts#L47)

- `string`: 그냥 문자열입니다.
- `symbol`: ES6에서 새로 추가된 원시 자료형(primitive type)입니다.
- `Newable<T>`: 클래스 생성자를 의미합니다. 문자열 대신 클래스 자체를 ServiceIdentifier로 사용할 수 있습니다.
  - e.g.) `inject(Constants.SERVICE_IDS.StateManager)` 대신에 `inject(StateManager)`를 사용한다. `StateManager`는 `new StateManager()`와 같은 형태로 사용하는 클래스 생성자다.
- `Abstract<T>`: 추상 클래스를 의미합니다. `Newable<T>`은 일반 클래스를 ServiceIdentifier로 사용할 수 있게 해주는 반면, `Abstract<T>`는 추상 클래스를 ServiceIdentifier로 사용할 수 있게 해줍니다.
  - [https://github.com/inversify/InversifyJS/commit/38b9a8a888ef4430bc60cda900abe118d225c35d](https://github.com/inversify/InversifyJS/commit/38b9a8a888ef4430bc60cda900abe118d225c35d)

타입스크립트를 자바스크립트로 컴파일하면 `class`에 대한 정보는 남아있지만 `interface`와 `type` 정보는 사라집니다. 생성자 매개변수를 `class`대신
`interface`나 `type`으로 선언한다면 의존성 주입을 위해서 필수로 ServiceIdentifier를 사용해야 합니다. 자바스크립트가 아니라 타입스크립트를 실행할
수 있는 엔진이 탄생해서 런타임에 `interface`와 `type`정보에 접근할 수 있다면 `InversifyJS`보다 더 진보된 프레임워크가 등장할 수 있습니다.

공식 문서에서는 ServiceIdentifier로 `symbol`을 사용하기를 권장하지만, 저는 `string`을 사용하는 편이 좋다고 생각합니다.

> 매우 큰 애플리케이션에서 InversifyJS로 타입을 주입하기 위한 식별자를 문자열로 사용하면 이름 충돌(naming collisions)이 발생할 수 있다.
> InversifyJS는 문자열 대신 Symbol을 식별자로 사용할 수 있고 이를 권장한다.
> 
> In very large applications using strings as the identifiers of the types to be injected by the InversifyJS can lead
> to naming collisions. InversifyJS supports and recommends the usage of Symbols instead of string literals.
> 
> \- [https://github.com/inversify/InversifyJS/blob/e2cf550fc09fafc6ed17db9b4eb7bcce9a8d1f9c/wiki/symbols_as_id.md](https://github.com/inversify/InversifyJS/blob/e2cf550fc09fafc6ed17db9b4eb7bcce9a8d1f9c/wiki/symbols_as_id.md)

`symbol`은 ES6에서 추가된 원시 자료형(primitive type)입니다. `symbol`은  `string`, `number`와 같은 자바스크립트의 기본 자료형입니다.
문자열이나 숫자를 쓰면 되지 왜 `symbol`을 만들었을까요? mdn web docs에는 아래처럼 나와있습니다.

> Symbol은 내장형 객체인데, Symbol 생성자는 **유일함을 보장**하는 `symbol` 원시 값을 리턴한다. 이 원시 값은 Symbol 값으로도 불리기도 하고 그냥
> Symbol이라고도 불린다. Symbol은 주로 기존 객체에 있는 속성 키(key)들과 충돌하지 않는 유일한 속성을 추가하기 위해 사용한다. 다른 코드에서 객체에
> 접근하는 일반적인 방법(mechanisms)에 대해서는 Symbol로 만든 속성 키가 숨겨지기 때문에 약한 캡슐화나 약한 정보 은닉의 형태를 취할 수 있다. 
> 
> Symbol is a built-in object whose constructor returns a symbol primitive — also called a Symbol value or just a Symbol
> — **that's guaranteed to be unique**. Symbols are often used to add unique property keys to an object that won't collide
> with keys any other code might add to the object, and which are hidden from any mechanisms other code will typically
> use to access the object. That enables a form of weak encapsulation, or a weak form of information hiding.
> 
> \- [https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol)

`Symbol()` 생성자에 동일한 값을 넣어도 매번 다른 `symbol` 객체가 탄생합니다.

```typescript
const same1 = Symbol("same")
const same2 = Symbol("same")
assert(same1 !== same2);
```

이런 특성 때문에 매우 큰 애플리케이션에서는 `symbol`로 ServiceIdentifier를 사용하면 동일한 문자열을 생성자 매개변수로 넣더라도
ServiceIdentifier가 충돌하지 않을 수 있습니다. 하지만 일반적 애플리케이션에서는 오히려 이름 충돌이 발생하도록 `string`으로 ServiceIdentifier를
사용하는 것이 나을 수도 있습니다. 보통의 경우 실수로 같은 이름의 ServiceIdentifier를 사용하는 경우가 많기 때문에 `symbol`을 사용해서 이름 충돌의
가능성을 없애버리다가 동일한 이름의 다른 `symbol`을 사용하게 되면 프로그램이 오작동할 수 있기 때문입니다.

문서에서 말하는 `very large application`은 한 두 사람이 애플리케이션 전체를 파악할 수 없는, 여러 개의 큰 도메인을 다루는 애플리케이션일 것 같습니다.
ServiceIdentifier들이 서로 멀리 떨어져 있어 실수로라도 다른 값을 `import`할 수 없을 정도라면 문자열 대신 `symbol`을 사용하는 것이 나을 수도
있습니다. 당연히 우리 게시판 애플리케이션은 그 정도가 아니므로 저는 문자열만 사용하겠습니다.

추가로 `Symbol()`대신 `Symbol.for()`를 사용하는 경우는 `symbol`을 사용하지만 '매번 유일함(guaranteed to be unique)'의 의미가 퇴색이 됩니다.
`Symbol.for()`는 전역 `symbol` 공간에 매개변수의 값이 있으면 `symbol`을 생성하지 않고 이미 생성되어 있는 `symbol`을 리턴합니다. 매개변수로 처음
들어오는 값이라면 새로운 `symbol`을 생성합니다.

```typescript
const same1 = Symbol.for("same")
const same2 = Symbol.for("same")
assert(same1 === same2);
```

`Symbol.for()`로 ServiceIdentifier를 만들어서 사용한다면 일반 문자열을 사용하는 것과 차이가 없습니다.

## 애플리케이션 컨텍스트에서 IoC 컨테이너로

InversifyJS 사용법을 어느 정도 알게 되었으니 수동으로 작성했던 애플리케이션 컨텍스트를 제어의 역전 컨테이너로 다시 작성해보겠습니다. InversifyJS를
사용하려면 3가지가 필요합니다.

1. ServiceIdentifier
2. `decorateClass(): void` 함수
3. `bind(c: Container): void` 함수

아래처럼 `DiConfig`라는 인터페이스를 만들어 사용합니다. `DiConfig` 인터페이스를 구현하는 클래스를 만들어서 의존성 주입 설정을 합니다.
설정들을 모아서 `decorateClass()`를 호출해 클래스들의 의존관계에 대한 정보를 추가하고 `bind()` 함수를 호출해서 컨테이너에게 ServiceIdentifier가
어떤 클래스를 의미하는지 알려줍니다. 제어의 역전 컨테이너와 관련된 설정은 `adapter` 디렉토리에 넣겠습니다.

```typescript
// src/common/adapter/DiConfig.ts

import { Container } from "inversify";

export interface DiConfig {
  decorateClass(): void;
  bind(c: Container): void;
}
```

저는 6개의 `DiConfig` 구현체를 만들었습니다.

- [src/article/adapter/ArticleIncomingConfig.ts](https://github.com/myeongjae-kim/nodejs-tutorial-example/blob/chapter-4-complete/src/article/adapter/ArticleIncomingConfig.ts)
- [src/article/adapter/ArticleOutgoingConfig.ts](https://github.com/myeongjae-kim/nodejs-tutorial-example/blob/chapter-4-complete/src/article/adapter/ArticleOutgoingConfig.ts)
- [src/article/adapter/ArticleStateManagerConfig.ts](https://github.com/myeongjae-kim/nodejs-tutorial-example/blob/chapter-4-complete/src/article/adapter/ArticleStateManagerConfig.ts)
- [src/article/adapter/ArticleUiConfig.ts](https://github.com/myeongjae-kim/nodejs-tutorial-example/blob/chapter-4-complete/src/article/adapter/ArticleUiConfig.ts)
- [src/common/adapter/ApplicationConfig.ts](https://github.com/myeongjae-kim/nodejs-tutorial-example/blob/chapter-4-complete/src/common/adapter/ApplicationConfig.ts)
- [src/common/adapter/CliConfig.ts](https://github.com/myeongjae-kim/nodejs-tutorial-example/blob/chapter-4-complete/src/common/adapter/CliConfig.ts)

그리고 `src/initialize-container.ts`에서 아래처럼 컨테이너를 준비합니다.

```typescript
import { Container } from "inversify";
import "reflect-metadata";
import { ArticleIncomingConfig } from "./article/adapter/ArticleIncomingConfig";
import { ArticleOutgoingConfig } from "./article/adapter/ArticleOutgoingConfig";
import { ArticleStateManagerConfig } from "./article/adapter/ArticleStateManagerConfig";
import { ArticleUiConfig } from "./article/adapter/ArticleUiConfig";
import { ApplicationConfig } from "./common/adapter/ApplicationConfig";
import { CliConfig } from "./common/adapter/CliConfig";
import { DiConfig } from "./common/adapter/DiConfig";

let decorated = false;

export const initializeContainer = (): Container => {
  const diConfigs: DiConfig[] = [
    new ArticleIncomingConfig(),
    new ArticleOutgoingConfig(),
    new ArticleStateManagerConfig(),
    new ArticleUiConfig(),
    new ApplicationConfig(),
    new CliConfig(),
  ];

  // initialize classes with decorators only once
  if (!decorated) {
    diConfigs.forEach((it) => it.decorateClass());
    decorated = true;
  }

  // create IoC Container and bind service identifiers to classes or instances
  const container = new Container({
    defaultScope: "Singleton",
  });
  diConfigs.forEach((it) => it.bind(container));

  return container;
};
```

`decorateClass()` 함수는 단 한번만 호출해야 합니다. 두 번 호출하게 되면 아래처럼 예외가 발생합니다.

```
.../nodejs-tutorial-example/node_modules/reflect-metadata/Reflect.js:541
                var decorated = decorator(target);
                                ^
Error: Cannot apply @injectable decorator multiple times.
    at .../nodejs-tutorial-example/node_modules/inversify/src/annotation/injectable.ts:8:13
    at DecorateConstructor (/Users/mj/projects/nodejs-tutorial-example/node_modules/reflect-metadata/Reflect.js:541:33)
```

`initializeContainer()` 함수 바깥에 `decorated` 플래그를 선언해서 `decorateClass()`가 단 한 번만 불리도록 보장했습니다.

`src/index.ts`에서 컨테이너를 생성하고 `ApplicationByStateManager` 인스턴스를 찾아서 실행합니다. 컨테이너를 생성할 때 `defaultScope`를
`Singleton`으로 지정합니다. 우리가 생성할 클래스 인스턴스들은 게시판 애플리케이션의 시작부터 끝까지 하나씩만 있으면 됩니다. 떄문에 제어의 역전 컨테이너
안에서 여러 개의 인스턴스가 생성되지 않게끔 기본 스코프를 `Singleton`으로 지정했습니다.

컨테이너를 생성할 때 기본 스코프를 지정하지 않으면 `Transient`로 지정됩니다. 이 상태로 애플리케이션을 실행하면 글을 작성해도 목록에서 확인할 수 없게
됩니다. `ArticleInMemoryRepository` 객체를 하나만 생성하는게 아니라 컨테이너가 요청을 받을 때마다 생성하기 때문에 작성, 목록조회, 개별조회용으로
총 3개의 `ArticleInMemoryRepository`의 인스턴스가 생성이 됩니다. 인스턴스마다 `Article`을 따로 관리하므로 글을 작성하더라도 목록조회 화면에서
글을 볼 수 없게 됩니다.

```typescript
import { ApplicationByStateManager } from "./ApplicationByStateManager";
import { Constants } from "./Constants";
import { initializeContainer } from "./initialize-container";

const container = initializeContainer();

const applicationByStateManager = container.get<ApplicationByStateManager>(
  Constants.SERVICE_IDS.ApplicationByStateManager
);

applicationByStateManager.run();
```

이로써 수많은 클래스와 인터페이스를 의존하던 애플리케이션 컨텍스트를 제어의 역전 컨테이너로 대체했습니다.

## `ArticleIncomingConfig` 살펴보기

[위에서 말씀드린 것처럼](#애플리케이션-컨텍스트에서-ioc-컨테이너로) ServiceIdentifier, `decorateClass()`, `bind()` 세 가지만 알면
InversifyJS를 사용할 수 있습니다. `DiConfig`를 구현한 `ArticleIncomingConfig`도 동일하게 세 부분으로 나누어집니다.

```typescript
import { Container, decorate, inject, injectable } from "inversify";
import { DiConfig } from "../../common/adapter/DiConfig";
import { ArticleCommandService } from "../application/ArticleCommandService";
import { ArticleQueryService } from "../application/ArticleQueryService";
import { ArticleCreateUseCase } from "../application/port/incoming/ArticleCreateUseCase";
import { ArticleGetUseCase } from "../application/port/incoming/ArticleGetUseCase";
import { ArticleListUseCase } from "../application/port/incoming/ArticleListUseCase";
import { ArticleOutgoingConfig } from "./ArticleOutgoingConfig";

export class ArticleIncomingConfig implements DiConfig {
  private static SERVICE_ID_PRIVATE = {
    ArticleCommandService: "ArticleCommandService",
    ArticleQueryService: "ArticleQueryService",
  };

  public static SERVICE_ID = {
    ArticleCreateUseCase: "ArticleCreateUseCase",
    ArticleGetUseCase: "ArticleGetUseCase",
    ArticleListUseCase: "ArticleListUseCase",
  };

  public decorateClass(): void {
    decorate(injectable(), ArticleCommandService);
    decorate(
      inject(ArticleOutgoingConfig.SERVICE_ID.ArticleSavePort),
      ArticleCommandService,
      0
    );

    decorate(injectable(), ArticleQueryService);
    decorate(
      inject(ArticleOutgoingConfig.SERVICE_ID.ArticleLoadPort),
      ArticleQueryService,
      0
    );
  }
  public bind(c: Container): void {
    // private
    c.bind<ArticleCommandService>(
      ArticleIncomingConfig.SERVICE_ID_PRIVATE.ArticleCommandService
    ).to(ArticleCommandService);

    c.bind<ArticleQueryService>(
      ArticleIncomingConfig.SERVICE_ID_PRIVATE.ArticleQueryService
    ).to(ArticleQueryService);

    // public
    c.bind<ArticleCreateUseCase>(
      ArticleIncomingConfig.SERVICE_ID.ArticleCreateUseCase
    ).toService(ArticleIncomingConfig.SERVICE_ID_PRIVATE.ArticleCommandService);
    c.bind<ArticleGetUseCase>(
      ArticleIncomingConfig.SERVICE_ID.ArticleGetUseCase
    ).toService(ArticleIncomingConfig.SERVICE_ID_PRIVATE.ArticleQueryService);
    c.bind<ArticleListUseCase>(
      ArticleIncomingConfig.SERVICE_ID.ArticleListUseCase
    ).toService(ArticleIncomingConfig.SERVICE_ID_PRIVATE.ArticleQueryService);
  }
}
```

ServiceIdentifier를 `SERVICE_ID_PRIVATE`과 `SERVICE_ID`로 나누었습니다. `SERVICE_ID_PRIVATE`은 구현체를 위한 식별자이고 설정 내부에서만
사용하기 위해 `private`으로 선언했습니다. `SERVICE_ID`는 인터페이스를 위한 식별자입니다.

`decorateClass()`는 [이전에 말씀드린 것](#데코레이터가-없었으면-좋겠는데)과 동일합니다. `bind()`는 두 부분으로 나누어집니다. 구현체에 대한 정보를
컨테이너 입력할 때는 `c.bind(식별자).to(클래스)`으로 등록합니다. 인터페이스에 대한 ServiceIdentifer는
`c.bind(식별자).toService(구현체식별자)`로 컨테이너에 등록합니다. `to()`로 인스턴스를 생성하는 대신, 직전에 `bind()`한 구현체 식별자를
`toService()`의 매개변수에 넣어서 인터페이스 식별자와 연결합니다. `ArticleIncomingConfig` 바깥에서는 구현체에 대한 ServiceIdentifier에
접근할 수 없고 인터페이스에 대한 ServiceIdentifier만 접근할 수 있습니다.

`decorateClass()`를 보면 `ArticleSavePort`와 `ArticleLoadPort`의 ServiceIdentifier를 사용해서 `ArticleCommandService`와
`ArticleQueryService`의 생성자에 어떤 객체를 주입해야 하는지에 대한 정보를 추가합니다. `AticleSavePort`와 `ArticleLoadPort`
ServiceIdentifier에 대한 객체를 컨테이너에 요청하면 동일한 `ArticlePersistenceAdapter` 클래스의 인스턴스를 얻게됩니다.
[`ArticleOutgoingConfig`](https://github.com/myeongjae-kim/nodejs-tutorial-example/blob/chapter-4-complete/src/article/adapter/ArticleOutgoingConfig.ts)에서도
마찬가지로 `bind()`에서 구현체용 식별자와 인터페이스용 식별자를 구별했습니다.

## 제어의 역전? 의존성 주입?

저는 [위에서 '제어의 역전 컨테이너는 설계도대로 조립해주는 기계'](#inversifyjs)라고 대충 설명했습니다.

제어의 역전(IoC, Inversion of Control)은 프로그램의 실행 흐름에 대한 권한을 뒤집는 것을 말합니다.
[챕터 2에서 Callback과 Promise를 설명하면서 제어의 역전을 아래와 같이 말씀드렸습니다.](/pages/002-hexagonal-architecture/#callback을-promise로)

> 동기적으로 하던 일을 비동기적으로 처리하기 위해 콜백 함수를 만들었습니다. 이것이 첫 번째 제어의 역전입니다. 콜백 함수가 언제 불릴지는 우리가 결정할 수
> 없고 콜백 함수를 매개변수로 받는 함수에서 결정합니다. 우리는 프라미스를 사용해서 콜백 함수 내부에서 하던 일을 밖으로 빼냅니다. 이것이 두 번째 제어의
> 역전입니다. 콜백 함수는 결과를 전달하는 역할만 할 뿐, 그 이후의 과정은 콜백함수 바깥에서 진행합니다. 우리는 프라미스를 통해서 잃었던 제어권을 다시 찾아올
> 수 있습니다.

제어의 역전 컨테이너는 인스턴스가 언제 생성되고 주입되는지에 대한 제어를 우리에게서 빼앗아갑니다. 애플리케이션 컨텍스트를 수동으로 직접 작성하던 시절에는
우리가 애플리케이션의 모든 흐름에 대한 제어권을 가지고 있었습니다. 하지만 애플리케이션에 기능이 늘어날수록 애플리케이션 컨텍스트를 작성하고 유지보수하기가
어려워지기 때문에 제어의 역전 컨테이너를 도입해서 클래스간의 의존관계만 입력하면 자동으로 클래스들을 조립하도록 했습니다. 대신에 우리는 언제 클래스 인스턴스가
생성되는지에 대한 제어권을 잃게 됩니다.

클래스간의 의존관계에 따라 인스턴스를 주입하기 때문에 제어의 역전 컨테이너는 다른 이름으로 의존성 주입 컨테이너(DI Container, Dependency Injection Container)
라고도 불립니다. 의존성 주입은 제어의 역전 기법의 범주에 포함됩니다.[^3] 마틴 파울러는 '제어의 역전'의 의미가 모호하다고 생각했기 때문에 '의존성 주입'이라는
단어를 선호합니다.[^4]

## 확장성을 고려하면

InversifyJS는 의존성 주입 컨테이너이므로 제어의 역전 컨테이너라고 부를 수도 있습니다. InversifyJS 소개 페이지에 'A powerful and lightweight
inversion of control container for JavaScript & Node.js apps powered by TypeScript.' 라고 적혀있으므로 챕터 이름을 제어의 역전
컨테이너라고 지었습니다. 제어의 역전 컨테이너를 처음 접해본다면 복잡하고 어렵게 느껴지는게 당연합니다. 그냥 애플리케이션 컨텍스트를 수동으로 작성해서 관리하는게
나아보일 수도 있습니다. 하지만 확장성(Scalability) 관점에서 본다면 수동으로 관리하는 애플리케이션 컨텍스트는 금방 한계를 맞게 됩니다.

작은 애플리케이션에 알맞은 아키텍처가 있고 대규모 애플리케이션에 알맞은 아키텍처가 있습니다. 적은 트래픽에 알맞은 백엔드 아키텍처가 있고 대용량 트래픽에 알맞은
백엔드 아키텍처가 있습니다. 보통 규모가 작을 때 통하는 방법은 쉽고 간단하고, 규모가 클 때 통하는 방법은 보다 복잡하고 어렵습니다. 간단한 게시판 애플리케이션에
InversifyJS를 사용하는 것은 과하지만 학습을 위해 사용해봤습니다. InversifyJS를 사용하면 애플리케이션의 규모가 커지더라도 일관적인 방법으로 클래스들의
관계를 관리할 수 있기 때문에, 많은 기능이 추가되어서 코드베이스가 커지더라도 현재의 구조를 꽤 큰 규모까지 유지할 수 있을 거라고 생각합니다.

만약 애플리케이션이 너무너무 커지거나 동일한 비즈니스 로직에 대해서 여러 개의 애플리케이션을 만들어야 한다면 어떨까요? 여기서 모노레포(Monorepo)가
등장합니다.


---

지금까지 작성한 코드는 [nodejs-tutorial-example:chapter-4-done](https://github.com/myeongjae-kim/nodejs-tutorial-example/tree/chapter-4-done)에서 확인할 수 있습니다.

---

[^1]: [https://ko.wikipedia.org/wiki/메타프로그래밍](https://ko.wikipedia.org/wiki/메타프로그래밍)
[^2]: [https://en.wikipedia.org/wiki/Reflective_programming](https://en.wikipedia.org/wiki/Reflective_programming)
[^3]: [https://stackoverflow.com/a/6551303/14659782](https://stackoverflow.com/a/6551303/14659782)
[^4]: [https://martinfowler.com/articles/injection.html#InversionOfControl](https://martinfowler.com/articles/injection.html#InversionOfControl) <br />'제어의 역전이라는 단어의 역사에 대해서는 다음 페이지 하단의 어원학(Etymology)를 참고하세요: [https://martinfowler.com/bliki/InversionOfControl.html](https://martinfowler.com/bliki/InversionOfControl.html) <br /> 위 문서에서는 라이브러리와 프레임워크의 차이에 대해서도 알 수 있습니다. 웹 프론트엔드 라이브러리중에서 [VueJS](https://vuejs.org)는 프레임워크, [React](https://reactjs.org)는 라이브러리라고 자신들을 소개하고 있습니다.
