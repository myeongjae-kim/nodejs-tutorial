---
title: 3. 애플리케이션 컨텍스트
author: Myeongjae Kim
date: 2022-03-12
category: Tutorial
layout: post
---

## 의존관계 그래프

애플리케이션 컨텍스트는 말 그대로 애플리케이션을 실행하기 위한 맥락입니다. 제가 작성한 `ApplicationByStateManager` 클래스를 인스턴스화 하려면
생성자 매개변수로 `StateManager`, `ArticleQueryViewController`, `ArticleCommandViewController`의 인스턴스를 전달해야 합니다.
이 3개의 인스턴스를 만드려면 또 필요한 인스턴스들이 있고, 그 인스턴스들을 만들기 위해 더 필요한 인스턴스들이 있고... 자료구조를 공부하셨다면 이쯤에서 아마
'그래프' 자료구조가 생각이 나실 수도 있습니다.

<figure>
    <a href="/res/4-dependency-graph.svg" target="_blank">
        <img alt="Dependency Graph" src="/res/4-dependency-graph.svg" style="width: 100%;max-width: 100% !important"/>
    </a>
    <figcaption>
        <a href="https://stackoverflow.com/a/47909020/14659782" target="_blank">https://stackoverflow.com/a/47909020/14659782</a>
        <pre>npx depcruise --exclude \"^node_modules\" --ts-pre-compilation-deps --output-type dot src/index.ts | dot -T svg -o dependencies.svg</pre>
    </figcaption>
</figure>

저는 어떤 클래스가 다른 클래스의 인스턴스를 의존해야 할 때 모두 생성자에서 인스턴스들을 주입받도록 했습니다. 어떤 클래스의 인스턴스가 만들어지려면 다른
클래스의 인스턴스가 필요하기 때문에, 만약 클래스들이 생성자 매개변수로 서로를 의존하게 된다면(=환형 의존, Cycle Dependency) 인스턴스를 만들 수 없습니다.
결국 제가 작성한 클래스들의 의존 관계를 그래프로 그려보면 DAG(Directed Acyclic Graph)가 됩니다. 위 그래프에 선이 많지만... 잘 보면 좌->우 방향의
화살표만 있고 우->좌 방향의 화살표는 없기 때문에 환형 의존이 발생하지 않는다는 것을 알 수 있습니다.

## 환형 의존 관계와 교착 상태(Deadlock)

환형 의존 관계 때문에 인스턴스 생성이 불가능한 상황은, 동시성 프로그래밍이나 데이터베이스를 공부하다가 마주치게 되는 교착 상태 문제와 완전히 동일합니다.

<figure>
    <img alt="Deadlock Crossroad" src="/res/5-deadlock.jpeg" />
    <figcaption><a href="https://preamtree.tistory.com/18">https://preamtree.tistory.com/18</a></figcaption>
</figure>

`A`가 작업을 수행하기 위해 `B`를 기다리는데 `B`가 작업을 수행하기 위해 `A`를 기다리게 되는 환형 의존 관계가 발생했을 때는 서로를 가리키는 화살표중 하나를
없애서 대처하면 됩니다. `A`와 `B`중 둘 중 하나를 죽여버려야 하는데 무엇을 죽일지는 상황에 따라 달라집니다. 둘 다 죽여버리고 새로 시작하는 방법도 있습니다.

<figure>
    <img alt="Deadlock Crossroad" src="/res/6-cycle-dependency.svg" style="width: initial" />
    <figcaption><pre>echo "digraph { A -> B \n B -> A}" | dot -Tsvg > cycle-dependency.svg</pre></figcaption>
</figure>

### 마이크로서비스의 환형 의존 관계

여러 개의 백엔드 애플리케이션이 협업하면서 작동하는 마이크로서비스 패턴에서도 환형 의존 관계는 큰 문제입니다. 배포할 때 가장 곤란합니다. 보통 마이크로서비스
형태의 애플리케이션들을 배포할 때는 의존 당하는 서비스를 먼저 배포합니다. `A` -> `B` 방향으로 의존하고 있다면, `B` 서비스를 배포하고 `A`를 배포합니다.
이 때 새로운 버전인 `B'`은 `A`의 요청도 소화할 수 있어야 하고 새로운 버전인 `A'`의 요청도 소화할 수 있어야 합니다. 단방향 의존이 아니라 양방향 의존을
하고 있다면 문제가 심각해집니다. `A`와 `B`중에서 무엇을 먼저 배포해야 할까요? 기능 설계 단계에서부터 `A`와 `B`의 환형 의존 관계를 고려하지 않는다면 배포
때마다 공포에 사로잡혀서 기도해야 할지도 모릅니다. 단방향 관계의 복잡도가 `A + B`라면, 양방향 관계의 복잡도는 `A x B`가 됩니다.

양방향 관계를 해소하기 위해 `A`와 `B`를 하나로 합칠 수도 있고, `C`라는 새로운 무언가를 도입해서 `C`가 `A`와 `B`를 의존하게 만드는 방법도 있습니다.
반대로 `A`와 `B`가 `C`를 의존하게 할 수도 있습니다. 마이크로서비스 환경이라면 `C`라는 새로운 백엔드 애플리케이션을 만들어서 `A`와 `B`를 호출해서 기능을
수행할 수 있습니다. 아니면 `C`를 메시지 큐(e.g. [Kafka](https://kafka.apache.org))로 구성해서 `A`와 `B`가 메시지 큐에 메시지를 보내고 상대가 보낸 메시지를 구독하는 방식으로도
양방향 관계를 해소할 수 있습니다.

<figure>
    <img alt="Deadlock Crossroad" src="/res/7-avoid-cycle-1.svg" style="width: initial" />
    <img alt="Deadlock Crossroad" src="/res/8-avoid-cycle-2.svg" style="width: initial" />
    <figcaption><pre>echo "digraph { C -> A \n C -> B}" | dot -Tsvg > avoid-cycle-1.svg<br>echo "digraph { C <- A \n C <- B}" | dot -Tsvg > avoid-cycle-2.svg</pre></figcaption>
</figure>

### 클래스간의 환형 의존 관계

마찬가지로 클래스가 서로를 의존한다는 것은 설계상에 뭔가 문제가 있다는 신호입니다. 서로를 의존하는 클래스들은 코드상으로는 나뉘어져있지만 그냥 하나의 커다란
덩어리로 간주해야 합니다. 하나의 클래스에서 다뤄야 할 책임을 부주의하게 두 개로 나눈 경우가 될 수도 있고, 개발자가 보기엔 아무리 봐도 별개의 책임인데
비즈니스적인 요구때문에 어쩔 수 없이 의존해야 하는 상황이 발생할 수도 있습니다.

객체 의존관계에서 꼭 환형 의존이 필요하다면 생성자 주입 방식이 아니라 setter 방식으로 주입을 하면 됩니다. 객체를 생성한 뒤에 setter를 통해서 추가로
의존성을 주입합니다. 저는 환형 의존 관계가 필요한 상황을 마주칠 때마다 리팩토링을 해서 생성자 주입 방식으로만 프로그램을 작성해왔습니다. 함께하다 보면 서로를
의존하기 마련인데 컴퓨터 세상은 단방향의 사랑만 허락하는 것 같습니다..

## 애플리케이션 컨텍스트 생성하기

다시 우리의 애플리케이션으로 돌아와서, 저는 `ApplicationContext`라는 인터페이스를 만들고 `createApplicationContext`라는 함수에서 클래스들을
조립해서 `ApplicationContext` 인터페이스를 만족하는 객체를 리턴하도록 만들었습니다.

```typescript
import { createStore } from "@reduxjs/toolkit";
import readline from "readline";
import { ArticleInMemoryRepository } from "./article/adapter/outgoing/persistence/ArticleInMemoryRepository";
import { ArticlePersistenceAdapter } from "./article/adapter/outgoing/persistence/ArticlePersistenceAdapter";
import { ArticleCommandService } from "./article/application/ArticleCommandService";
import { ArticleQueryService } from "./article/application/ArticleQueryService";
import { ArticleCreateUseCase } from "./article/application/port/incoming/ArticleCreateUseCase";
import { ArticleGetUseCase } from "./article/application/port/incoming/ArticleGetUseCase";
import { ArticleListUseCase } from "./article/application/port/incoming/ArticleListUseCase";
import { ArticleImpl } from "./article/domain/ArticleImpl";
import { ArticlePrinter } from "./article/view/cli/ArticlePrinter";
import { MyStore } from "./article/view/cli/state-modules/redux/MyStore";
import * as reduxModule from "./article/view/cli/state-modules/redux/redux-module";
import { StateManager } from "./article/view/cli/state-modules/vanila/StateManager";
import { MobxRootState } from "./article/view/cli/state-modules/mobx/MobxRootState";
import { MenuPrinter } from "./article/view/cli/MenuPrinter";
import { CliInOut } from "./common/view/cli/CliInOut";
import { ArticleQueryViewController } from "./article/view/cli/ArticleQueryViewController";
import { ArticleCommandViewController } from "./article/view/cli/ArticleCommandViewController";

export interface ApplicationContext {
  articleGetUseCase: ArticleGetUseCase;
  articleListUseCase: ArticleListUseCase;
  articleCreateUseCase: ArticleCreateUseCase;

  stateManager: StateManager;
  store: MyStore;
  mobxRootState: MobxRootState;

  articleQueryViewController: ArticleQueryViewController;
  articleCommandViewController: ArticleCommandViewController;
}

export const createApplicationContext = (
  initialArticles: Record<number, ArticleImpl> = {}
): ApplicationContext => {
  const articleInMemoryRepository = new ArticleInMemoryRepository(
    initialArticles
  );

  const articlePersistenceAdapter = new ArticlePersistenceAdapter(
    articleInMemoryRepository
  );

  const articleQueryService = new ArticleQueryService(
    articlePersistenceAdapter
  );
  const articleCommandService = new ArticleCommandService(
    articlePersistenceAdapter
  );

  const menuPrinter = new MenuPrinter(new ArticlePrinter());

  const cliInOut = new CliInOut(
    readline.createInterface({
      input: process.stdin,
      output: process.stdout,
    })
  );

  const articleQueryViewController = new ArticleQueryViewController(
    cliInOut,
    menuPrinter,
    articleQueryService,
    articleQueryService
  );

  const articleCommandViewController = new ArticleCommandViewController(
    cliInOut,
    menuPrinter,
    articleCommandService
  );

  return {
    articleGetUseCase: articleQueryService,
    articleListUseCase: articleQueryService,
    articleCreateUseCase: articleCommandService,

    stateManager: new StateManager(),
    store: createStore(reduxModule.reducer),
    mobxRootState: new MobxRootState(),

    articleQueryViewController,
    articleCommandViewController,
  };
};
```

그리고 `src/index.ts`에서 애플리케이션 컨텍스트를 생성하고, `ApplicationByStateManager`에 필요한 인스턴스들을 애플리케이션 컨텍스트에서 꺼내
`ApplicationByStateManager`를 생성합니다.

```typescript
import { ApplicationByStateManager } from "./ApplicationByStateManager";
import { createApplicationContext } from "./applicationContext";

const context = createApplicationContext();

const applicationByStateManager = new ApplicationByStateManager(
  context.stateManager,
  context.articleQueryViewController,
  context.articleCommandViewController
);
applicationByStateManager.run();

/*
const applicationByRedux = new ApplicationByRedux(
  context.store,
  context.articleQueryViewController,
  context.articleCommandViewController
);
applicationByRedux.run();

const applicationByMobx = new ApplicationByMobx(
  context.mobxRootState,
  context.articleQueryViewController,
  context.articleCommandViewController
);
applicationByMobx.run();
 */
```

굳이 애플리케이션 컨텍스트를 만드는 부분과 실제 실행할 애플리케이션 클래스를 생성하는 부분을 나눠야 할 필요가 있을까요? 애플리케이션 컨텍스트가 모든 걸
알고있으니 컨텍스트를 만들면서 애플리케이션도 만들 수 있으니까요. 설계에 정답은 없습니다. 저는 상태 관리에 어떤 기술을 쓰느냐에 따라 애플리케이션 클래스를
다르게 작성했기 때문에 컨텍스트와 애플리케이션 클래스를 완전히 분리해서 구성했습니다. 애플리케이션 컨텍스트는 애플리케이션 클래스를 모르게 되고,
`src/index.ts`에서 다시 애플리케이션 컨텍스트와 애플리케이션 클래스를 조립하고 실행합니다.

만일 하나의 애플리케이션 클래스만 작성했다면, `createApplicationContext` 함수 대신 `createApplication` 함수를 작성해서 애플리케이션 클래스의
인스턴스를 리턴하도록 하고, `src/index.ts` 에서 `createApplication().run()`과 같은 방식으로 애플리케이션을 실행하게 했을 것 같습니다. 진입
지점(보통 `main` 함수라고 불리는 entry point)은 깔끔할 수록 좋으니까요.

## 복잡하고 불안한 애플리케이션 컨텍스트

우리는 비즈니스 로직이 '세부사항'을 모르게 하기 위해 육각형 아키텍처를 선택했습니다. 비즈니스 로직이 의도적으로 피했던 정보(=세부사항)들이 애플리케이션
컨텍스트에 모이게 됩니다. 모든 것을 알아야 하는 애플리케이션 컨텍스트는 수많은 클래스와 인터페이스를 의존하게 되고, 팬 아웃(fan-out) 숫자가 큰
매우 불안정한 상태로 기능하게 됩니다.[^1]

간단한 게시판 애플리케이션임에도 불구하고 애플리케이션 컨텍스트의 `import`문의 수가 아찔합니다. 애플리케이션의 규모가 커지고 클래스가 많아지면 클래스를
인스턴스화하고 조립하는 코드를 작성하는 일도 점점 더 복잡해집니다.

애플리케이션을 조립하는 부분의 코드를 보면 뻔함의 반복입니다. 그저 인스턴스들을 생성하고 알맞은 곳에 끼워넣어줄 뿐.
**클래스들의 의존관계에 대한 정보를 코드상에서 다룰 수 있으면** 클래스들을 자동으로 조립할 수도 있을 것 같은데... 

여기서 제어의 역전(IoC, Inversion of Control) 컨테이너가 등장합니다.

---

지금까지 작성한 코드는 [nodejs-tutorial-example:chapter-3-done](https://github.com/myeongjae-kim/nodejs-tutorial-example/tree/chapter-3-done)에서 확인할 수 있습니다.

---

[^1]: [[소프트웨어 공학] 팬인(Fan-In)/ 팬아웃(Fan-Out)](https://post.naver.com/viewer/postView.nhn?volumeNo=27424202&memberNo=26040503)
