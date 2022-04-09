---
title: 서문
author: Myeongjae Kim
date: 2022-03-11
category: Tutorial
layout: post
---

안녕하세요, 김명재입니다.

저는 우아한형제들의 웹툰 플랫폼 만화경을 개발하는 3년 6개월동안 자바와 코틀린으로 백엔드 애플리케이션을 만들었습니다. 그중 2년은
[Next.js](https://nextjs.org)와 [Material-UI](https://mui.com)로 웹 프론트엔드 개발도 함께 했었습니다. 만화경 백엔드는 프로젝트 시작부터
마이크로서비스 형태를 갖추고 있었고, JVM(Java Virtual Machine) 진영의 빌드 툴인 gradle을 사용해서 멀티모듈(≒모노레포)을 구성해 여러 개의 백엔드
서비스를 하나의 저장소에서 관리했습니다.

<figure>
    <a href="https://www.manhwakyung.com" target="_blank">
        <img alt="Dependency Graph" src="/res/manhwakyung.jpg" style="border-radius: 5px"/>
    </a>
    <figcaption>
        <a href="https://www.manhwakyung.com" target="_blank">https://www.manhwakyung.com</a>
    </figcaption>
</figure>

서비스가 성장하면서 만화경 백오피스 웹 프론트엔드 애플리케이션의 코드량도 많아지고, UI는 비슷하지만 다른 종류의 백오피스 웹 프론트엔드 애플리케이션이 필요하게
되었습니다. 코드를 재사용하기 위해 npm으로 관리하던 백오피스 웹 프론트엔드 저장소도 백엔드 저장소와 유사하게 모노레포를 도입했습니다.

모노레포 도입 시기에 [Lerna(러나)](https://lerna.js.org)와 [Rush(러쉬)](https://rushjs.io)를 두고 고민했었습니다. Lerna는 사용법이
간단하고 학습량이 적은 대신 사용자가 스크립트를 작성해서 해결해야 하는 부분이 있었던 반면, Rush는 기능이 많고 문서도 많고 학습량도 많지만 초반의 러닝 커브만
잘 뛰어넘는다면 Lerna보다 훨씬 편하게 사용할 수 있어보였습니다.

그러나 Rush의 국내 인지도는 Lerna보다 낮았고 사용 사례도 거의 없었으며,
[npm trends상에서도 워낙 Lerna가 독보적](https://www.npmtrends.com/@microsoft/rush-vs-lerna)이라 회사 프로젝트에서는 보수적인
관점에서 Lerna를 선택해서 사용했습니다. 2019년엔 정말 차이가 많이(20배 차이) 났었는데 요새는 Rush도 많이 올라왔습니다(3배 차이).

마침 최근에 새로운 환경에서 모노레포를 구성해야 할 필요가 있어서 지난번에 도전하지 않았던 Rush를 공부해서 예제 프로젝트를 구성해보았고, Lerna보다
만족스러운 부분이 많아서 매우 만족하고 있습니다.

<div style="display: flex; justify-content: center; margin: 16px">
    <div class="lerna-logo-background" style="border-radius: 5px; padding: 16px 16px 0 16px; margin: 8px">
        <img alt="Lerna logo" src="/res/lerna.svg" width="200px">
    </div>
    <div style="display: inline-flex; margin: 8px">
        <img alt="Rush logo" src="/res/rush.svg" width="200px">
    </div>
</div>

작년 8월에 정성대님이 작성한 ['Rush로 프론트엔드 모노레포 도입기'](https://medium.com/mildang/rush로-프론트엔드-모노레포-도입기-5da0c5bc9b30)에
'Lerna + Yarn workspace는 어플리케이션 개발보다는 오픈소스에 더 유용하다'라고 말씀하신 부분이 있습니다. 저도 여기에 동감하는 바이고, Rush는
마이크로소프트 Share Point 팀에서 수백 개의 NPM 패키지를 관리하면서(Rush was created by the platform team for
[Microsoft SharePoint](http://aka.ms/spfx). We build hundreds of production NPM packages every day,
[https://rushjs.io/pages/intro/welcome/](https://rushjs.io/pages/intro/welcome/)) 탄생한 도구이기 때문에 예제 프로젝트를 만들어보면서
충분히 학습한다면 실제 업무 환경에 도입하기에는 문제 없어 보입니다.

'Node.js 객체지향 모노레포 튜토리얼'에서는 '읽기, 쓰기만 가능한 게시판'이라는 간단한 도메인가진 명령줄 인터페이스(Command Line Interface, CLI)
애플리케이션을 만들게 됩니다. 객체지향의 원칙을들을 지키며 클래스 기반으로 애플리케이션을 구성하고, 객체지향 프로그래밍을 하다보면 자연스럽게 만나게 되는
클래스간의 의존성 관리 문제를 InversifyJS로 해결합니다. 그리고 더 큰 규모로 애플리케이션을 확장하기 위해 Rush와 Heft를 사용해서 모노레포 환경을
구성합니다.

모노레포 이야기를 많이 말씀드렸지만, 튜토리얼에는 모노레포 뿐만 아니라 제가 3년 6개월동안 일하면서 공부하고 경험해서 얻은 지식들도 많이 적어놨습니다.
커리어의 시작이 JVM을 사용하는 백엔드 개발자라 객체지향 프로그래밍을 공부하게 되면 자연스럽게 만나는 지식들이 대부분이지만, '읽기, 쓰기만 가능한 게시판'이라는
도메인을 중심에 두고 하나의 흐름을 유지해가며 지식들을 정리했다는 것에 저는 의미를 두고 있습니다.

이 지식들은 Next.js와 Material-UI로 프론트엔드 애플리케이션을 만들면서도 잘 써먹었습니다. JVM이든 Node.js든 결국 '비즈니스 요구사항에 빠르게 대응할
수 있는 유연한 프로그램'이라는 이상향을 추구한다면 비슷한 아키텍처를 가질 수 밖에 없겠다고 생각하는 편이고, 로버트 마틴도 그의 책 『클린 아키텍처』에서 같은
이야기를 합니다.

모노레포 구성을 고민하고 계실 정도면 이미 어느정도 프로그래밍을 잘 하실 테니 기초적인 내용은 적지 않았습니다. 객체지향 프로그래밍을 이해하고 타입스크립트를
사용하실 수 있다면 튜토리얼을 무리없이 이해하실 수 있을 거라고 생각합니다.

도움이 됐으면 좋겠습니다. 감사합니다.
