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

제가 느낀 가장 큰 장점은, GUI(Graphic User Interface)나 http, 데이터베이스같은 외부 의존성을 비즈니스 로직과 철저하게 분리할 수 있다는 것입니다. 비즈니스 로직이 세부사항에 오염되지
않도록 순수하게 유지해야 한다는 것은 로버트 마틴이 『클린 아키텍처』[^2]에서 말했던 바이고, 에릭 에반스의 『도메인 주도 설계』[^3]의 내용과도 일맥상통합니다.

> 육각형 아키텍처는 앨리스테어 코어번(Alistair Cockburn)이 2005년에 정의했습니다. 코어번은 이후에 이 아키텍처의 이름을 "Port And Adapter Pattern"이라고 
> 이름을 붙혔지만 많은 사람들이 여전히 육각형 아키텍처라는 이름을 사용하기를 선호합니다.
> 이 주제에 대한 [원본 문서는 여기](https://web.archive.org/web/20060711221010/http://alistair.cockburn.us:80/index.php/Hexagonal_architecture)에서 볼 수 있습니다.
> 
> \- [Tugce Konuklar, Hexagonal (Ports & Adapters) Architecture](https://medium.com/idealo-tech-blog/hexagonal-ports-adapters-architecture-e3617bcf00a0#8ad5)[^4]

코어번이 작성한 원본 문서는 web.archive.org에서 확인할 수 있습니다. 1차자료를 꼭 직접 읽어보세요!

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

---

[^1]: [톰 홈버그, 박소은. _만들면서 배우는 클린 아키텍처_. 파주: 위키북스, 2021](https://wikibook.co.kr/clean-architecture/)
[^2]: [로버트 C. 마틴, 송준의. _클린 아키텍처_. 서울: 인사이트, 2019](http://ebook.insightbook.co.kr/book/69)
[^3]: [에릭 에반스, 이대엽. _도메인 주도 설계_. 파주: 위키북스, 2011](https://wikibook.co.kr/domain-driven-design/)
[^4]: [Konuklar, Tugce. _“Hexagonal (Ports & Adapters) Architecture”_. Idealo Tech Blog (blog), 2021년 4월 20일.](https://medium.com/idealo-tech-blog/hexagonal-ports-adapters-architecture-e3617bcf00a0#8ad5)

