---
title: 2. 육각형 아키텍처
author: Myeongjae Kim
date: 2022-03-12
category: Tutorial
layout: post
---

## 비즈니스 로직을 순수하게

저는 육각형 아키텍처(Hexagonal Architecture)를 책 『만들면서 배우는 클린 아키텍처』[^1] 에서 처음 알게 되었습니다.
고전적인 엔터프라이즈 아키텍처인 [3 Layered Architecture](https://www.ecanarys.com/Blogs/ArticleID/76/3-Layered-Architecture) 보다
훨씬 장점이 많다고 느껴서 실무에도 적극적으로 사용하고 있습니다.

제가 느낀 가장 큰 장점은, GUI(Graphic User Interface)나 http, 데이터베이스같은 외부 의존성을 비즈니스 로직과 철저하게 분리할 수 있다는 것입니다. 비즈니스 로직이 세부사항[^2]에 오염되지
않도록 순수하게 유지하라는 말은 로버트 마틴이 『클린 아키텍처』[^3]에서 언급했던 바이고, 에릭 에반스의 『도메인 주도 설계』[^4]의 내용과도 일맥상통합니다.

> 육각형 아키텍처는 앨리스테어 코어번(Alistair Cockburn)이 2005년에 정의했습니다. 코어번은 이후에 이 아키텍처의 이름을 "Port And Adapter Pattern"이라고 
> 이름을 붙혔지만 많은 사람들이 여전히 육각형 아키텍처라는 이름을 사용하기를 선호합니다.
> 이 주제에 대한 [원본 문서는 여기](https://web.archive.org/web/20060711221010/http://alistair.cockburn.us:80/index.php/Hexagonal_architecture)에서 볼 수 있습니다.
> 
> \- [Tugce Konuklar, Hexagonal (Ports & Adapters) Architecture](https://medium.com/idealo-tech-blog/hexagonal-ports-adapters-architecture-e3617bcf00a0#8ad5)[^5]

코어번이 작성한 원본 문서는 [web.archive.org](https://web.archive.org/web/20060711221010/http://alistair.cockburn.us:80/index.php/Hexagonal_architecture)에서 확인할 수 있습니다.
육각형 아키텍처의 창시자가 직접 작성한 1차자료를 꼭 직접 읽어보세요!

## 육각형 아키텍처를 사용하는 디렉토리(패키지) 구조

책 『만들면서 배우는 클린 아키텍처』[^1]에서는 마틴의 클린 아키텍처 원칙을 따르면서 코어번의 육각형 아키텍처를 응용해 아래처럼 디렉토리 구조를 제안합니다. 

<figure>
    <img alt="Packages for Hexagonal Architecture" src="/res/1-package-hexagonal.jpg" />
    <figcaption>『만들면서 배우는 클린 아키텍처』, p27</figcaption>
</figure>

비즈니스 로직은 `application` 패키지의 `SendMoneyService`와 `domain` 패캐지의 `Account, Activity`에서 수행합니다.
육각형 아키텍처의 궁극적인 목적은 이 3개의 클래스가 세부사항에 오염되지 않도록 보호하는 것입니다. 이를 위해 `SendMoneyService` 클래스는
`SendMoneyUseCase`라는 인터페이스를 구현합니다. `application` 패키지 바깥에서는 SendMoneyService를 직접 의존할 수 없고
포트 역할을 하는 `SendMonyUseCase` 인터페이스만을 의존할 수 있습니다.

코어번의 글을 읽어보면 'primary adapter'와 'secondary adapter'라는 개념이 등장합니다. primary adapter는 UseCase를 사용하는 역할이고,
secondary adapter는 UseCase에 의해서 사용당하는 역할입니다. `adapter/in/web/AccountController`가 primary adapter,
`adapter/out/persistence` 이하의 adapter가 secondary adapter입니다.

사용자가 돈을 보내려고 `AccountController`를 호출합니다. `AccountController`는 `SendMoneyUseCase` 인터페이스를 의존하고 있으므로,
이 인터페이스에 돈을 보내라는 메시지(≒함수 호출)을 보냅니다. 이 인터페이스의 자리에 `SendMoneyService`가 플러그인 되어있으므로 `SendMoneyService` 클래스는
메시지를 받아서 돈을 보내는 비즈니스 로직을 수행합니다.

돈을 보냈다는 기록은 메모리상에만 존재하면 안되고 당연히 영구적으로 데이터를 보관할 수 있는 저장장치에 기록을 해야 합니다. `SendMoneyServcice`는
특정 저장장치와 관련된 클래스 구현체를 의존할 수도 있지만, 위에서 말씀드렸던 것처럼 비즈니스 로직을 담은 클래스는 세부사항에 의존하지 않아야 하므로
`SendMoneyService`는 `LoadAccountPort`와 `UpdateAccountPort`라는 인터페이스를 의존합니다.

이 2개의 포트 인터페이스에 대한 구현체는 `AccountPersistenceAdapter` 클래스 입니다. `SendMoneyService`에서 `LoadAccountPort`와
`UpdateAccountPort`에 메시지를 보내면 이 포트 인터페이스 자리에 플러그인 되어있는 `AccountPersistenceAdapter`가 메시지를 받아서
`SpringDataAccountRepository`를 사용해 송금 기록을 영구적인 저장장치에 기록합니다.

아래의 다이어그램에서 속이 꽉 차있는 화살표 →는 의존관계(혹은 연관관계)를 의미하고, 속이 비어있는 화살표 ⇾는 구현관계(혹은 일반화 관계)를 의미합니다.
`application` 패키지는 그 자체로서 완결성을 가지고, 이 패키지 바깥(=세부사항)을 의존하지 않습니다. 비즈니스 로직을 담당하는 패키지에
구멍(포트, 인터페이스)을 만들어놓고 외부에서 구현체를 꼽아넣음(plugin)으로서 `application` 패키지가 세부사항에 오염되지 않도록 순수하게 유지합니다.

<figure>
    <img alt="Class diagram" src="/res/2-dependency-diagram.jpg" />
    <figcaption>『만들면서 배우는 클린 아키텍처』, p31</figcaption>
</figure>

코어번은 이 장황한 설명을 아래와 같은 육각형의 다이어그램을 사용해서 표현했습니다. 더 자세한 이야기가 궁금하다면 꼭 『만들면서 배우는 클린 아키텍처』[^1]를
구매해서 읽어보세요!

<figure>
    <img alt="Hexagonal Diagram of Cockburn" src="/res/3-original-diagram.gif" width="974px" />
    <figcaption><a target="_blank" href="https://web.archive.org/web/20071009083921/http://alistair.cockburn.us/index.php/Image:Image177.gif">http://alistair.cockburn.us/index.php/Image:Image177.gif</a></figcaption>
</figure>

## 우리의 게시판 애플리케이션

챕터 1의 '명령줄 인터페이스 애플리케이션'을 구현하면서 저는 아래와 같은 디렉토리 구조를 사용했습니다.

[https://github.com/myeongjae-kim/nodejs-tutorial-example/tree/8ec18ea489b72b1481dbb6ae0e3f8700de366acd/src/article](https://github.com/myeongjae-kim/nodejs-tutorial-example/tree/8ec18ea489b72b1481dbb6ae0e3f8700de366acd/src/article))

```
article
├── adapter
│   └── outgoing
│       └── persistence
│           ├── ArticleInMemoryRepository.ts (Class)
│           └── ArticlePersistenceAdapter.ts (Class)
├── application
│   ├── ArticleCommandService.ts (Class)
│   ├── ArticleQueryService.ts (Class)
│   └── port
│       ├── incoming
│       │   ├── ArticleCreateUseCase.ts (Interface)
│       │   ├── ArticleGetUseCase.ts (Interface)
│       │   ├── ArticleListUseCase.ts (Interface)
│       │   ├── ArticleRequestDto.ts (Class)
│       │   └── ArticleResponseDto.ts (Class)
│       └── outgoing
│           ├── ArticleLoadPort.ts (Interface)
│           └── ArticleSavePort.ts (Interface)
├── domain
│   ├── Article.ts (Interface)
│   └── ArticleImpl.ts (Class)
└── view
    └── cli
        ├── ArticleCommandViewController.ts (Class)
        ├── ArticlePrinter.ts (Class)
        ├── ArticleQueryViewController.ts (Class)
        ├── MenuPrinter.ts (Class)
        └── state-modules
            ├── State.ts (Interface)
            ├── View.ts (Type)
            ├── mobx
            │   └── MobxRootState.ts (Class)
            ├── redux
            │   ├── MyStore.ts (Type)
            │   └── redux-module.ts (module)
            └── vanila
                └── StateManager.ts (Class)
```

### Domain

```
article/domain
├── Article.ts
└── ArticleImpl.ts
```

도메인에는 `Article` 인터페이스와 그 구현체인 `ArticleImpl`이 있습니다. 이렇게 나누어놓은 이유는 JPA(Java Persistence Api)를 사용하면서 생긴 습관인데,
JPA는 엔티티에 `@Entity` 애노테이션을 붙이도록 강제합니다. JPA의 구현체중 대표격인 Hibernate는 JPA의 애노테이션에 따라 AOP(Aspect Oriented Programming)
방식으로 Proxy를 사용해서 엔티티 클래스에 부가 기능을 붙이게 되는데요, 이 결과로 엔티티 클래스의 인스턴스가 순수한 자바 객체가 아니라 Hibernate에 오염되어버립니다.
때문에 캐싱을 하려고 직렬화/역직렬화를 할 때 순수한 자바 객체(POJO, Pure Old Java Object)가 아니라서 직렬화 과정과 역직렬화 과정이 제대로 이루어지지 않아
JPA 애노테이션이 붙어있는 엔티티의 인스턴스를 캐싱할 수 없었습니다. 그래서 `ArticleCached`같은 이름으로 순수한 자바 객체를 만들어 캐싱을 하곤 했었습니다.
이 과정에서 `ArticleImpl`과 `ArticleCached` 모두 `Article` 인터페이스를 구현하도록 해서 도메인 영역 바깥에서는 `Article` 인터페이스를 참조하게 만들었고요.

자바 이야기가 너무 길었네요. 어쨌든 엔티티를 인터페이스와 구현체로 나누어서 구현해놓으면 언젠가 도움이 됩니다. 혹시 모르죠, 우리 튜토리얼에서도 TypeORM같은
ORM 프레임워크를 사용해볼 기회가 있을지...?

### Application

```
article/application
├── ArticleCommandService.ts (Class)
├── ArticleQueryService.ts (Class)
└── port
    ├── incoming
    │   ├── ArticleCreateUseCase.ts (Interface)
    │   ├── ArticleGetUseCase.ts (Interface)
    │   ├── ArticleListUseCase.ts (Interface)
    │   ├── ArticleRequestDto.ts (Class)
    │   └── ArticleResponseDto.ts (Class)
    └── outgoing
        ├── ArticleLoadPort.ts (Interface)
        └── ArticleSavePort.ts (Interface)
```

`domain` 디렉토리와 `application` 디렉토리를 합쳐서 광의(廣義)의 도메인 영역이라고 할 수 있습니다. 오직 비즈니스만을 위한 코드가 존재하는,
세부사항[^2]에 오염되지 않도록 순수하게 유지해야 하는 영역입니다.

제일 먼저 봐야 하는 부분은 `port/incoming`의 유스케이스 인터페이스들입니다. 저는 아티클의 생성, 개별조회, 목록조회를 각각
`ArticleCreateUseCase`, `ArticleGetUseCase`, `ArticleListUseCase`라는 이름의 인터페이스로 만들었습니다.
개별 기능마다 각각 UseCase라는 단어를 붙였습니다 (톰 홈버그[^1]를 따라서). `article/application` 디렉토리 바깥에서 위 기능들을 사용하려면
기능의 구현체를 직접 의존하지 말고 유스케이스 인터페이스만을 의존해야 합니다.

코어번은 유스케이스에 대해서 아래와 같이 말했습니다.

> 육각형 아키텍처 패턴을 선호되는(the preferred way, 뭐라고 번역해야 좋을까요..) 유스케이스 작성법을 강화하기 위해 사용하면 유용하다.
> 
> 개별 포트의 바깥 부분에서 사용하는 기술과 밀접하게 연결된 지식을 유스케이스를 작성할 때 포함하는 것은 흔히 저지르는 실수다.
> 이런 (포트 바깥쪽 기술에 대한 지식을 포함하는) 유스케이스들은 길고, 읽기 어럽고, 지루하고, 불안정하고, 유지보수 비용이 많이 드는 것으로 악명을 떨쳐왔다.
> 
> '포트와 어댑터 아키텍처'를 이해한다면, 우리는 유스케이스들이 애플리케이션 경계(육각형의 안쪽)에 작성해야 한다는 것을 알 수 있다.
> 이렇게 함으로써 외부 기술과 관계 없이 애플리케이션에 의해 지원되는 기능과 이벤트를 지정(specify)할 수 있다. 이런 유스케이스들은 짧고, 읽기 쉽고,
> 유지보수 비용이 적게 들고, 시간이 지나도 안정적이다. 
> 
> \- [앨리스테어 코어번, 유스케이스와 애플리케이션 경계, _육각형 아키텍처_, 2005](https://web.archive.org/web/20060711221010/http://alistair.cockburn.us:80/index.php/Hexagonal_architecture#Use_Cases_And_The_Application_Boundary)

#### 유스케이스와 구현체

TODO

`ArticleCommandService`

구현체, Command와 Query. Query가 2개의 인터페이스를 구현한다. ISP 원칙을 지킨 것. 구현체에 여러 개의 기능이 있을 때 기능별로 인터페이스를 만들면
기능을 의존하는 쪽에서 쪼개놓은 인터페이스를 의존해서 불필요한 기능은 의존하지 않고 꼭 필요한 기능만 의존하도록 할 수 있습니다.

Command와 Query 구현체를 합쳐놓지 않고 나눠놓은 것은 버트런드 마이어의 CQS 원칙을 지킨 것. CQS와 CQRS에 대해서 더 알고싶다면 CQRS 글 보도록 링크하기.

command는 상태를 변경하는 책임을, query는 상태를 조회하는 책임을 진다. 이 관점에서 보면 객체 하나가 하나의 책임을 지는 것처럼 볼 수 있어서 SRP를 만족하는 것 같다.

하지만 조금 더 잘게 나눠보면 query는 목록조회와 개별조회로 나뉠 수 있다. 이 관점에서 보면 query가 두 개의 책임을 지므로 SRP 위반이다.
QueryService가 2개의 유스케이스를 구현하도록 만들었는데, 곰곰이 생각해보면 이는 SRP원칙 위반이다.

그러면 query 클래스를 나눠야 할까? 여기서부터는 trade-off의 영역이라고 생각한다. 클래스를 하나로 작성해서 보일러플레이트 코드를 줄일지, 클래스를 나누어서
좀 더 철저하게 SRP원칙을 지킬지. 저는 보통의 경우 Command와 Query를 나누는 정도로 서비스들을 작성하지만, 서비스의 기능(=책임) 하나하나가 커질 때 클래스가
하나의 기능만 갖도록 리팩토링 한다.

그러면 굳이 Query와 Command 객체를 나누어야 하냐고 의문이 들 수 있는데, 합쳐도 상관없지만 저는 존경하는 버트런드 마이어를 위해서 그의 CQS를 객체수준까지
확장해서 지키는 편입니다. 프로그래밍의 모든 비극은 상태를 변경하면서 발생하기 때문에 상태의 변경 여부에 따라 코드를 나누어 놓으면 마음이 한결 편합니다.







---

[^1]: [톰 홈버그, 박소은. _만들면서 배우는 클린 아키텍처_. 파주: 위키북스, 2021](https://wikibook.co.kr/clean-architecture/)
[^2]: 로버트 마틴의 책 _클린 아키텍처_ 에는 다음과 같은 챕터가 있다: '30장 데이터베이스는 세부사항이다', '31장 웹은 세부사항이다', '32장 프레임워크는 세부사항이다'
[^3]: [로버트 C. 마틴, 송준의. _클린 아키텍처_. 서울: 인사이트, 2019](http://ebook.insightbook.co.kr/book/69)
[^4]: [에릭 에반스, 이대엽. _도메인 주도 설계_. 파주: 위키북스, 2011](https://wikibook.co.kr/domain-driven-design/)
[^5]: [Konuklar, Tugce. _“Hexagonal (Ports & Adapters) Architecture”_. Idealo Tech Blog (blog), 2021년 4월 20일.](https://medium.com/idealo-tech-blog/hexagonal-ports-adapters-architecture-e3617bcf00a0#8ad5)

