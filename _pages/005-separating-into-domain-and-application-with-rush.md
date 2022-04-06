---
title: 5. 도메인과 애플리케이션의 분리 (with Rush)
author: Myeongjae Kim
date: 2022-03-12
category: Tutorial
layout: post
---

## Lerna와 Rush

node 기반 프로젝트를 모노레포로 구성할 수 있게 도와주는 도구가 여럿 있습니다. 저는 실무에서 2020년에 [Lerna](https://lerna.js.org)를 사용해
모노레포로 프로젝트를 구성한 경험이 있습니다. JVM 진영의 [gradle](https://gradle.org) 멀티모듈 구조에 익숙해 있던 터라 모노레포의 개념을 이해하기는
쉬웠지만, Lerna로 프로젝트를 구성해보니 설정이 복잡했고 `npm`만으로는 해결할 수 없는 문제가 있어서
[직접 심볼릭 링크를 업데이트하는 쉘 스크립트를 작성해서 사용](https://myeongjae.kim/blog/2020/06/12/copy-symbolic-link-of-relative-path)했습니다(이 문제는 Lerna와 yarn workspace를 함께 사용하는 방식으로도 해결이 가능합니다).

에듀테크 스타트업 밀당의 정성대님이 작성하신 [Rush로 프론트엔드 모노레포 도입기](https://medium.com/mildang/rush로-프론트엔드-모노레포-도입기-5da0c5bc9b30)
에서는 제가 말씀드릴 수 있는 것보다 훨씬 풍부하고 깊은 이야기가 있습니다. 정성대님의 글에서는 여러 개의 모노레포 관리 도구들을 비교하고
[Rush](https://rushjs.io)를 선택해서 React 버전을 점진적으로 업데이트합니다. 저도 2020년에 Lerna와 Rush중에 고민하다가 Rush가 더 나아보여서
Rush로 프로젝트 구성을 시도해봤는데, 이 때는 Next.js와 Rush가 호환이 되지 않아 Lerna를 사용했었습니다. 정성대님의 예시에서는 Rush에서
Next.js가 잘 작동합니다(그냥 그 때 제가 설정을 제대로 못한 것 같습니다). 

### PNPM?

pnpm은 npm같은 패키지 매니저입니다. Rush는 [npm](https://docs.npmjs.com), [pnpm](https://pnpm.io), [yarn](https://yarnpkg.com) 모두 사용할 수 있지만 pnpm을
권장합니다 ([NPM vs PNPM vs Yarn \| Rush](https://rushjs.io/pages/maintainer/package_managers/)). npm은 옛날 버전인 4.5.0을
권장하고, yarn은 사용은 가능하지만 [왠지 자신없어 하는 모습](https://rushjs.io/pages/maintainer/package_managers/#considerations-for-yarn)입니다.
우리의 게시판 애플리케이션 정도는 아마 npm 최신 버전에서도 잘 작동할 것 같아서 일단 프로젝트는 Rush와 npm을 사용한 뒤, 모노레포 구성 후에
npm에서 pnpm으로 마이그레이션을 해보겠습니다.

pnpm은 npm을 사용하면서 겪을 수 있는 [Phantom dependencies](https://rushjs.io/pages/advanced/phantom_deps/) 문제와
[NPM doppelgangers](https://rushjs.io/pages/advanced/npm_doppelgangers/) 문제를 해결했습니다. 이 문제들은 npm이 모든 의존성을
`node_modules`에 평평flat하게(=의존성의 의존성, 의존성의 의존성의 의존성... 이 모두 `node_modules` 디렉토리로 올라오는hoisted 것) 구성하기
때문에 발생합니다. 우리 프로젝트도 `dependencies`와 `devDependencies` 합쳐서 18개의 의존성을 사용하지만 `node_modules`에는 435개의 디렉토리가
있습니다.

<figure>
    <img alt="Super Massive node_modules" src="/res/11-super-massive-node-modules.webp" />
    <figcaption>초 거대 <del>블랙홀</del>node_modules<br />출처: <a target="_blank" href="https://dev.to/leoat12/the-nodemodules-problem-29dc">https://dev.to/leoat12/the-nodemodules-problem-29dc</a></figcaption>
</figure>

```json-doc
{
  ...,
  "dependencies": {
    "@reduxjs/toolkit": "^1.8.0",
    "inversify": "^6.0.1",
    "mobx": "^6.4.2",
    "node": "^17.5.0",
    "reflect-metadata": "^0.1.13"
  },
  "devDependencies": {
    "@johanblumenberg/ts-mockito": "^1.0.32",
    "@types/jest": "^27.4.1",
    "@typescript-eslint/eslint-plugin": "^5.14.0",
    "@typescript-eslint/parser": "^5.14.0",
    "dependency-cruiser": "^11.4.1",
    "eslint": "^8.11.0",
    "eslint-config-prettier": "^8.5.0",
    "jest": "^27.5.1",
    "nodemon": "^2.0.15",
    "prettier": "2.5.1",
    "ts-jest": "^27.1.3",
    "ts-node": "^10.7.0",
    "typescript": "^4.6.2"
  }
}
```

```
$ tree node_modules -L 1
node_modules
├── @ampproject
├── @babel
...
├── yargs
├── yargs-parser
└── yn

435 directories, 0 files
```

pnpm은 `node_modules` 디렉토리를 평평하지 않게non-flat 구성해서 [Phantom dependencies](https://rushjs.io/pages/advanced/phantom_deps/)
문제와 [NPM doppelgangers](https://rushjs.io/pages/advanced/npm_doppelgangers/) 문제를 해결합니다. `node_modules/.pnpm`에
의존성들을 설치하고, `node_modules`이하에는 심볼릭 링크를 생성해서 `node_modules/.pnpm`이하의 의존성에 연결합니다:
[https://pnpm.io/motivation#creating-a-non-flat-node_modules-directory](https://pnpm.io/motivation#creating-a-non-flat-node_modules-directory)

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">🟢Node.js Tip<br><br>Why store separate copies of packages if you can have a `single` referencable one?<br><br>That&#39;s the idea behind PNPM which makes it a performant, space-efficient, and faster alternative to NPM/Yarn.<br><br>Trust me, your hard disk will thank you for this weight loss exercise <a href="https://t.co/mltti8CFAX">pic.twitter.com/mltti8CFAX</a></p>&mdash; Hem (@HemSays) <a href="https://twitter.com/HemSays/status/1434921646083563525?ref_src=twsrc%5Etfw">September 6, 2021</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

## Rush를 사용해보자

아래 내용은 Rush의 [Maintainer Tutorials](https://rushjs.io/pages/maintainer/setup_new_repo/)를 참고해서 작성했습니다.

### 디렉토리 구조 변경

새로운 디렉토리를 만들어 `rush init`을 합니다. 비어있는 디렉토리가 아니면 `rush init`은 불가능합니다.

```
~/nodejs-tutorial-example$ npm install -g @microsoft/rush
~/nodejs-tutorial-example$ cd .. && mkdir nodejs-tutorial-example-rush && cd nodejs-tutorial-example-rush
~/nodejs-tutorial-example-rush$ rush init
```

아래처럼 파일들이 생성됩니다.

```
nodejs-tutorial-example-rush
├── .gitattributes
├── .gitignore
├── .travis.yml
├── common
│   ├── config
│   │   └── rush
│   │       ├── .npmrc
│   │       ├── .npmrc-publish
│   │       ├── .pnpmfile.cjs
│   │       ├── artifactory.json
│   │       ├── build-cache.json
│   │       ├── command-line.json
│   │       ├── common-versions.json
│   │       ├── experiments.json
│   │       ├── rush-plugins.json
│   │       └── version-policies.json
│   └── git-hooks
│       └── commit-msg.sample
└── rush.json
```

npm을 사용하므로 `rush.json`을 열어서 `pnpmVersion` 대신 `npmVersion`을 사용하도록 변경합니다.

```
$ npm --version
6.14.16
```

```json-doc
// rush.json
{
  ...,
  // "pnpmVersion": "6.7.1",

  "npmVersion": "6.14.16",
  ...
}
```

그리고 `nodejs-example-tutorial` 프로젝트 전체를 `app/board-cli`(명렁줄 인터페이스로 작동하는 게시판 애플리케이션이라는 의미) 디렉토리 이하로
복사합니다.

```
~/nodejs-tutorial-example-rush$ mkdir app && cd app
~/nodejs-tutorial-example-rush/app$ cp -R ../../nodejs-tutorial-example board-cli
~/nodejs-tutorial-example-rush/app$ cd board-cli
```

그리고 `~/nodejs-tutorial-example-rush/app/board-cli/package.json` 파일을 열어서 `name` 속성을 `nodejs-tutorial-example`에서
`app-board-cli`로 변경합니다.

```json-doc
// ~/nodejs-tutorial-example-rush/app/board-cli/package.json
{
  "name": "nodejs-tutorial-example",
  ...
}
```

`~/nodejs-tutorial-example-rush/app/board-cli/.git` 디렉토리를 `~/nodejs-tutorial-example-rush/.git`으로 옮겨서 `git` 설정과
히스토리를 보존하고 변경사항을 커밋합니다.

```
~/nodejs-tutorial-example-rush/app/board-cli$ mv .git ../../
~/nodejs-tutorial-example-rush/app/board-cli$ cd ../../
~/nodejs-tutorial-example-rush$ git add .
~/nodejs-tutorial-example-rush$ git commit
```

### 애플리케이션 빌드 및 실행

`rush.json`의 `projects` 배열에 아래와 같이 프로젝트를 추가합니다.

```json-doc
{
  ...,
  "projects": [
    {
      "packageName": "app-board-cli",
      "projectFolder": "app/board-cli"
    }
  ]
}
```

이후에 `rush update`를 실행합니다.

```
~/nodejs-tutorial-example-rush$ rush update
```

아래와 같은 메시지가 나오면 성공입니다.

```
LINKING: app-board-cli
Purging ../nodejs-tutorial-example-rush/app/board-cli/node_modules

Linking finished successfully. (9.18 seconds)

Next you should probably run "rush build" or "rush rebuild"

Rush update finished successfully. (52.95 seconds)
```

`app/board-cli/node_modules`를 확인해보면 의존성이 심볼릭 링크로 연결된 것을 확인할 수 있습니다.

```
~/nodejs-tutorial-example-rush$ ls -al app/board-cli/node_modules
...
lrwxr-xr-x 36 mj  2 Apr 15:57 ws -> ../../../common/temp/node_modules/ws
lrwxr-xr-x 45 mj  2 Apr 15:57 xdg-basedir -> ../../../common/temp/node_modules/xdg-basedir
lrwxr-xr-x 52 mj  2 Apr 15:57 xml-name-validator -> ../../../common/temp/node_modules/xml-name-validator
lrwxr-xr-x 42 mj  2 Apr 15:57 xmlchars -> ../../../common/temp/node_modules/xmlchars
lrwxr-xr-x 38 mj  2 Apr 15:57 y18n -> ../../../common/temp/node_modules/y18n
lrwxr-xr-x 41 mj  2 Apr 15:57 yallist -> ../../../common/temp/node_modules/yallist
lrwxr-xr-x 39 mj  2 Apr 15:57 yargs -> ../../../common/temp/node_modules/yargs
lrwxr-xr-x 46 mj  2 Apr 15:57 yargs-parser -> ../../../common/temp/node_modules/yargs-parser
lrwxr-xr-x 36 mj  2 Apr 15:57 yn -> ../../../common/temp/node_modules/yn
```

빌드 결과물이 담기는 디렉토리인 `app/board-cli/dist` 디렉토리를 제거한 뒤 `rush build`를 하고 애플리케이션을 실행해봅시다.

```
~/nodejs-tutorial-example-rush$ rm -rf app/board-cli/dist
~/nodejs-tutorial-example-rush$ rush build
~/nodejs-tutorial-example-rush$ app/board-cli
~/nodejs-tutorial-example-rush$ rushx start
1) 목록 조회
2) 쓰기
x) 종료

선택: 
```

실행이 잘 됩니다.

이제부터 `npm`의 커맨드는 잊으시는게 좋습니다. [`npm run` 대신 `rushx`](https://rushjs.io/pages/developer/everyday_commands/#rushx)를, 
[`npm install`대신 `rush add`](https://rushjs.io/pages/developer/modifying_package_json/)를 사용합니다. Rush로 관리하는
프로젝트에서 어떻게 개발하는지는 [Developer tutorials](https://rushjs.io/pages/developer/new_developer/)를 읽어보세요.

지금까지 작성한 코드는 [nodejs-tutorial-example:chapter-5-rush-setup](https://github.com/myeongjae-kim/nodejs-tutorial-example/tree/chapter-5-rush-setup)에서 확인할 수 있습니다.

## 도메인 패키지 분리

### 도메인 패키지 디렉토리 설계

챕터 2에서 육각형 아키텍처의 목적은 [비즈니스 로직을 순수하게 유지하는 것](/pages/002-hexagonal-architecture/#비즈니스-로직을-순수하게)이라고
말씀드렸습니다.

> `domain` 디렉토리와 `application` 디렉토리를 합쳐서 광의(廣義)의 도메인 영역이라고 할 수 있습니다. 오직 비즈니스만을 위한 코드가 존재하는,
> 세부사항에 오염되지 않도록 순수하게 유지해야 하는 영역입니다.
> 
> \- [2.육각형 아키텍처/Application](/pages/002-hexagonal-architecture/#application)

지금까지는 하나의 패키지 안에 사용자 인터렉션을 처리하는 `View`부터 순수한 도메인 영역까지 모두 관리했습니다. 이제 Rush로 프로젝트를 구성하도록
변경했으므로 도메인 영역을 다른 패키지로 분리해서 의존성의 방향을 이전보다 더 엄격하게 관리할 수 있습니다. 도메인 영역만 분리된 패키지로 구성할 수 있으므로
도메인 패키지를 재사용해서 여러 개의 애플리케이션을 만들 수도 있습니다.

`domain/board-domain` 패키지를 추가해서 `app/board-cli/src/article`의 `domain`과 `application` 디렉토리를 옮겨 도메인 패키지를
만든다면 디렉토리 구조를 어떻게 만들어야 할까요?

- `domain/board-domain/src/article/domain`
- `domain/board-domain/src/article/application`

그냥 그대로 옮기면 이렇게 두 개의 디렉토리가 생길텐데, 도메인 패키지의 `domain` 디렉토리라니.. 디렉토리 경로에서 중복이 발생하는 것 같아 영 찜찜하니까
`domain` 디렉토리를 `model`로 변경하겠습니다.

- `domain/board-domain/src/article/model`
- `domain/board-domain/src/article/application`

`model` 디렉토리는 '도메인 모델'을 의미하게 됩니다. 즉 엔티티만 있는 디렉토리입니다. `application` 디렉토리는 엔티티를 바탕으로 수행하는 비즈니스
로직이 들어있는 디렉토리가 됩니다. 그리고 디렉토리 구조에서 `application`이라는 단어는 이미 `app`이라는 디렉토리로 `app/board-cli`처럼 실행 가능한
애플리케이션이 있는 패키지의 부모 디렉토리로 사용하고 있으니, 도메인 패키지에서 `application` 디렉토리가 존재하는 것도 영 찜찜합니다.

도메인패키지의 `application` 디렉토리에는 `incoming` 포트와 `outgoing` 포트, 그리고 `incoming` 포트 인터페이스를 구현하는 서비스들이
들어있습니다. 외부 의존성 없이 엔티티와 포트로만 수행하는 비즈니스 로직은 도메인의 범주 안에 있기 때문에 굳이 `application` 디렉토리 아래로 넣어놓지 말고
그냥 밖으로 꺼내놓으면 어떨까요? 아래와 같은 트리 구조도 괜찮아 보입니다.

```
domain/board-domain/src/article
├── ArticleCommandService.ts
├── ArticleQueryService.ts
├── port
│    ├── incoming
│    │   ├── ArticleCreateUseCase.ts
│    │   ├── ArticleGetUseCase.ts
│    │   ├── ArticleListUseCase.ts
│    │   ├── ArticleRequest.ts
│    │   └── ArticleResponse.ts
│    └── outgoing
│        ├── ArticleLoadPort.ts
│        └── ArticleSavePort.ts
└── model
    ├── Article.ts
    └── ArticleImpl.ts
```

### 도메인 패키지 추가

어떤 구조로 도메인 패키지를 구성할지는 정해졌으니 `rush.json`에 `domain/board-domain` 디렉토리를 `board-domain`이라는 패키지 이름으로
추가합니다.

```json-doc
{
  ...,
  "projects": [
    {
      "packageName": "app-board-cli",
      "projectFolder": "app/board-cli"
    },
    {
      "packageName": "board-domain",
      "projectFolder": "domain/board-domain"
    }
  ]
}
```

`domain/board-domain` 디렉토리를 생성합니다.

```
~/nodejs-tutorial-example-rush$ mkdir -p domain/board-domain && cd domain/board-domain
```

`npm init`을 입력해 `domain/board-domain/package.json` 파일을 생성합니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ npm init
```

`entry-point`(`main` 속성)만 `dist/index.js`로 지정합니다.

```json-doc
// domain/board-domain/package.json
{
  "name": "board-domain",
  "version": "1.0.0",
  "description": "",
  "main": "dist/index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC"
}
```

`rush.json`에 프로젝트를 추가하고 `domain/board-domain/package.json`을 추가했으니 `rush update`를 실행합니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ rush update
```

`typescript`, `eslint`, `prettier`, `jest` 설정은 공통으로 사용하는 방법이 있을 것 같기도 하지만, 일단 `app/board-cli` 디렉토리에서 그대로
복사해오고 `rush add`로 `devDependencies` 의존성도 설치합니다. (개발 의존성을 여러 패키지에서 공통으로 사용하는 방법이 있는지 찾아봤는데 없다고
합니다: [https://stackoverflow.com/a/69803173/14659782](https://stackoverflow.com/a/69803173/14659782) 수동으로 개발 의존성을
모듈마다 설치하면 해당 패키지의 `package.json` 만 보고 의존성을 명확하게 파악할 수 있다는 장점이 있습니다.)

```
~/nodejs-tutorial-example-rush/domain/board-domain$ cd ../../app/board-cli
~/nodejs-tutorial-example-rush/app/board-cli$ cp .eslintignore .eslintrc.js .prettierignore .prettierrc.json jest.config.js tsconfig.json ../../domain/board-domain
~/nodejs-tutorial-example-rush/app/board-cli$ cd ../../domain/board-domain
~/nodejs-tutorial-example-rush/domain/board-domain$ rush add -p @johanblumenberg/ts-mockito -p @types/jest -p @typescript-eslint/eslint-plugin -p @typescript-eslint/parser -p eslint -p eslint-config-prettier -p jest -p prettier -p ts-jest -p ts-node -p typescript --dev --caret
```

`domain/board-domain/package.json`의 `devDependencies`에 아래와 같이 의존성이 추가되었습니다.

```json-doc
{
  ...,
  "devDependencies": {
    "@johanblumenberg/ts-mockito": "^1.0.32",
    "@types/jest": "^27.4.1",
    "@typescript-eslint/eslint-plugin": "^5.17.0",
    "@typescript-eslint/parser": "^5.17.0",
    "eslint": "^8.12.0",
    "eslint-config-prettier": "^8.5.0",
    "jest": "^27.5.1",
    "prettier": "^2.6.1",
    "ts-jest": "^27.1.4",
    "ts-node": "^10.7.0",
    "typescript": "^4.6.3"
  }
}
```

프로젝트가 특정 의존성의 과거 버전을 사용해야만 한다거나, 여러 프로젝트가 사용하는 의존성의 버전이 프로젝트마다 다른 경우 예상치 못한 문제가 발생할 수 있기
때문에 Rush에서는 [`common/config/rush/common-versions.json`](https://rushjs.io/pages/configs/common-versions_json/)을
제공해서 의존성 버전을 통일할 수 있게 도와줍니다. `dependencies`에서 사용하는 의존성은 `range` 없이 특정한 버전을, `devDependencies`에서 사용하는
의존성은 자동으로 `MINOR`패치까지 업데이트 하는 caret(`^`)을 사용하도록 지정합니다.

```json-doc
// common/config/rush/common-versions.json
{
  ...,
  "preferredVersions": {
    // dependencies
    "@reduxjs/toolkit": "1.8.0",
    "inversify": "6.0.1",
    "mobx": "6.4.2",
    "node": "17.5.0",
    "reflect-metadata": "0.1.13",

    // devDependencies
    "@johanblumenberg/ts-mockito": "^1.0.32",
    "@types/jest": "^27.4.1",
    "@typescript-eslint/eslint-plugin": "^5.14.0",
    "@typescript-eslint/parser": "^5.14.0",
    "dependency-cruiser": "^11.4.1",
    "eslint": "^8.11.0",
    "eslint-config-prettier": "^8.5.0",
    "jest": "^27.5.1",
    "nodemon": "^2.0.15",
    "prettier": "2.5.1",
    "ts-jest": "^27.1.3",
    "ts-node": "^10.7.0",
    "typescript": "^4.6.2"
  }
}
```

지금보니 `app/board-cli/package.json`에서 `dependencies`에도 caret을 사용하고 있었네요. 동일하게 `dependencies`에서만 caret을 제거합니다.

저는 `dependencies`에서는 caret(`^`)이나 tilde(`~`)를 사용하지 않고 특정한 버전을 지정해서 사용하는 편이 좋다고 생각합니다. 관련된 더 깊은 이야기는
[Should you Pin your JavaScript Dependencies?](https://docs.renovatebot.com/dependency-pinning/)를 읽어보시길 추천드립니다.

`common-versions.json`을 변경했으니 `rush update --full`를 실행합니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ rush update --full
```

`src/index.ts` 파일에 `foo` 함수를 추가해서 로그를 출력하도록 합니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ mkdir src
~/nodejs-tutorial-example-rush/domain/board-domain$ echo 'export const foo = () => console.log("Hello, board-domain!")' > src/index.ts
```

`domain/board-domain/package.json`의 `scripts` 속성에 아래와 같이 커맨드를 추가합니다.

```json-doc
{
  ...,
  "scripts": {
    "build": "npm run prettier && npm run lint && tsc",
    "test": "jest",
    "report": "jest unit --collect-coverage && open coverage/lcov-report/index.html",
    "lint": "eslint . --ext .ts,.tsx",
    "prettier": "prettier --write .",
    "clean": "rm -rf dist"
  }
}
```

`rush build`를 입력해서 빌드가 잘 되는지 확인합니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ rush build
==[ board-domain ]=================================================[ 1 of 2 ]==
"board-domain" completed successfully in 4.86 seconds.

==[ app-board-cli ]================================================[ 2 of 2 ]==
"app-board-cli" completed successfully in 6.46 seconds.



==[ SUCCESS: 2 operations ]====================================================

These operations completed successfully:
  app-board-cli    6.46 seconds
  board-domain     4.86 seconds


rush build (6.59 seconds)
```

두 개의 모듈이 모두 빌드에 성공했습니다. 이제 `app-board-cli`에서 `board-domain`을 사용할 수 있도록 `app/board-cli/package.json`의
`dependencies`에 `"board-domain": "1.0.0"`을 추가합니다.

```json-doc
// app/board-cli/package.json
{
  ...,
  "dependencies": {
    "@reduxjs/toolkit": "1.8.0",
    "inversify": "6.0.1",
    "mobx": "6.4.2",
    "node": "17.5.0",
    "reflect-metadata": "0.1.13",
    "board-domain": "1.0.0"
  }
}
```

패키지간 의존성을 추가했으므로 `app/board-cli/node_modules`를 업데이트하기 위해 `rush update --full`을 합니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ rush update --full
...
Linking local projects

LINKING: app-board-cli
Purging /Users/mj/projects/nodejs-tutorial-example-rush/app/board-cli/node_modules

LINKING: board-domain
Purging /Users/mj/projects/nodejs-tutorial-example-rush/domain/board-domain/node_modules

Linking finished successfully. (2.05 seconds)

Next you should probably run "rush build" or "rush rebuild"


Rush update finished successfully. (9.28 seconds)
```

`app/board-cli/node_modules/board-domain` 심볼릭 링크가 생성된 것을 확인할 수 있습니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ ls -al ../../app/board-cli/node_modules | grep board-domain
lrwxr-xr-x 28 mj  2 Apr 20:25 board-domain -> ../../../domain/board-domain
```

`rush build`를 합니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ rush build
```

`app/board-cli/src/index.ts`에서 `board-domain`의 `foo`함수를 가져올 수 있는지 확인해봅니다.

<figure>
    <img alt="No exported types" src="/res/12-no-exported-types.png" />
    <figcaption>타입이 없어요!</figcaption>
</figure>

타입이 없다고 나오는군요. `domain/board-domain/dist`를 확인해보면 `index.js`만 있고 타입 관련 파일은 없습니다. 이전까지는 컴파일한 결과물이
자바스크립트로서 실행만 되면 충분했기 때문에 타입스크립트 컴파일러가 타입 정보는 출력하지 않도록 하고 있었습니다. 하지만 `board-domain`패키지는
`board-cli` 패키지에서 `import`해서 사용하기 때문에 컴파일 할 때 타입정보까지 출력하도록 변경해야 합니다.

`domain/board-domain/tsconfig.json`에서 `declaration`과 `declarationMap` 속성을 `true`로 변경합니다.

```json-doc
{
  ...,
  "declaration": true,
  "declarationMap": true,
  ...
}
```

다시 빌드합니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ rush build
```

`dist` 디렉토리에 `.d.ts`와 `.d.ts.map` 파일도 생성됩니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ tree dist
dist
├── index.d.ts
├── index.d.ts.map
└── index.js
```

`index.d.ts`에는 `foo()`함수에 대한 타입이 선언되어 있습니다. `index.d.ts.map`은 `index.ts`에서 `index.js`로 컴파일할 때 타입스크립트 소스가
자바스크립트의 어느 부분으로 변환되었는지를 알려줍니다. 실행만을 위해서는 필요 없는 파일이지만 디버깅할 때 유용하게 쓸 수 있으므로 같이 포함해서 내보냅니다.

`app/board-cli/src/index.ts`를 열었던 소스 코드 편집기(제 경우 VSCode)를 재시작하면 `board-domain` 패키지의 타입 관련 에러가 사라진 것을
확인할 수 있습니다.

<figure>
    <img alt="Type found" src="/res/13-type-fixed.png" />
    <figcaption>타입이 생겼어요!</figcaption>
</figure>

`index.d.ts.map` 덕분에 `foo()` 함수의 선언을 VSCode에서 추적하면 `index.d.ts`가 아니라 원본 소스인 `index.ts`로 추적을 해줍니다.

<figure>
    <img alt="Jumping to function declaration" src="/res/14-go-to-definition.png" />
    <figcaption>F12를 누르면 원본 소스로 이동합니다.</figcaption>
</figure>

`applicationByStateManager.run()` 대신 `foo()`를 호출하면 `Hello, board-domain!` 문자열이 출력이 되는 것을 확인할 수 있습니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ cd ../../app/board-cli 
~/nodejs-tutorial-example-rush/app/board-cli$ rush build && rushx start
```

<figure>
    <img alt="Hello, board-domain!" src="/res/15-log-printed.png" />
    <figcaption><code>board-cli</code>패키지에서 <code>board-domain</code>패키지의 코드를 사용할 수 있습니다.</figcaption>
</figure>

`domain/board-domain/package.json` 패키지를 보면 `devDependencies` 속성만 있고 `dependencies`는 없습니다. 도메인 패키지에서는 최대한
보수적으로 외부 의존성을 추가해야 합니다. 아직은 `console.log`밖에 없지만, `board-cli`의 도메인 관련 코드를 옮겨올 때도 외부 의존성이 없을 수
있을지... 만들어봅시다.

일단 그 전에 `.gitignore`파일을 수정하고 커밋해야 합니다. `app/board-cli/.gitignore`의 내용을 프로젝트 root의 `.gitignore`에 추가하고
`app/board-cli/.gitignore`를 제거한 뒤에 커밋합니다. 저는 아래와 같이 `.gitignore`에 내용을 추가했습니다.

```
#Custom
*.js
dist

!jest.config.js
!.eslintrc.js

dependencies.svg
```

지금까지 작성한 코드는 [nodejs-tutorial-example:chapter-5-add-domain-package](https://github.com/myeongjae-kim/nodejs-tutorial-example/tree/chapter-5-add-domain-package)에서 확인할 수 있습니다.

### `rush clean` 커맨드 추가

이전까지는 소스코드가 하나의 패키지 안에 있었고 빌드를 하면 `dist` 디렉토리 안에 빌드 결과물이 생성되었습니다. 그러나 패키지가 두 개로 나뉜 순간부터는
빌드 결과물이 두 군데로 흩어지기 때문에 이전의 빌드 결과물을 완전히 제거하려면 `app/board-cli/dist`와 `domain/board-domain/dist` 두 군데를
모두 제거해야 합니다.

[Rush는 사용자가 Custom commands를 추가할 수 있도록 허용](https://rushjs.io/pages/maintainer/custom_commands/)하고 있습니다.
빌드 결과물의 제거를 위해 `package.json`에 `clean` 스크립트를 정의해놨으니 `rush clean` 커맨드를 추가해서 두 패키지의 빌드 결과물을 한 번에
제거하겠습니다.

`common/config/rush/command-line.json`의 `commands` 속성에 아래와 같이 추가합니다.

```json-doc
{
  ...,
  "commands": [
    {
      "commandKind": "bulk",
      "name": "clean",
      "summary": "Delete build results of each project.",
      "description": "Delete build results of each project by removing dist directory",
      "enableParallelism": true
    }
  ]
}
```

`commandKind`는 `bulk`와 `global`이 있습니다. `bulk`는 커맨드를 모든 프로젝트마다 한 번씩 실행하고, `global`은 저장소 전체에 대해서 한 번만
커맨드를 실행합니다. `rush clean`을 하면 `dist` 디렉토리 이하의 모든 파일과 디렉토리를 제거합니다.

```
~/nodejs-tutorial-example$ rush clean

Starting "rush clean"

Executing a maximum of 16 simultaneous processes...

==[ board-domain ]=================================================[ 1 of 2 ]==
"board-domain" completed successfully in 0.13 seconds.

==[ app-board-cli ]================================================[ 2 of 2 ]==
"app-board-cli" completed successfully in 0.06 seconds.



==[ SUCCESS: 2 operations ]====================================================

These operations completed successfully:
  app-board-cli    0.06 seconds
  board-domain     0.13 seconds


rush clean (0.22 seconds)

~/nodejs-tutorial-example$ ls app/board-cli/dist       # 아무것도 출력하지 않는다.
~/nodejs-tutorial-example$ ls domain/board-domain/dist # 아무것도 출력하지 않는다.
```

### `board-cli`에서 도메인 분리하기

[이전에 설계했던 대로](/pages/005-separating-into-domain-and-application-with-rush/#도메인-패키지-디렉토리-설계) `board-cli`에 있는
도메인 영역을 `board-domain`패키지로 가져옵니다. 음.. 어떻게 하면 좋을까요? 파일을 이동하면 IDE에서 `import`를 자동으로 변경하면서 뭔가 꼬일 것
같으니 `application`과 `domain` 디렉토리를 복사해서 `board-domain` 패키지에 붙여넣고 설계한 대로 디렉토리를 변경하겠습니다: [https://github.com/myeongjae-kim/nodejs-tutorial-example/commit/6e9642e87566dc9637f2cbaf6340f04bbbfcad35](https://github.com/myeongjae-kim/nodejs-tutorial-example/commit/6e9642e87566dc9637f2cbaf6340f04bbbfcad35)

기존에 작성했던 테스트들은 `board-domain`에서도 통과합니다. 도메인 영역을 외부 의존성 없이 순수한 타입스크립트로만 작성했기 떄문에 `board-domain`에
의존성을 추가하지 않아도 됩니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ rushx test
Test Suites: 5 passed, 5 total
Tests:       7 passed, 7 total
Snapshots:   0 total
Time:        3.41 s, estimated 4 s
Ran all test suites.
```

`rush build`를 실행해서 `domain/board-domain/dist` 디렉토리를 업데이트한 뒤에 `app/board-cli/src/application`,
`app/board-cli/src/domain` 디렉토를 과감하게 제거하고 오류가 발생하는 `import`에 대해서 `board-domain`을 의존하도록 변경합니다.
`board-domain/src/index.ts`의 `foo()` 함수는 더 이상 필요 없으니 제거합니다: [https://github.com/myeongjae-kim/nodejs-tutorial-example/commit/145a603e565294b357834a083bd711c5610ec554](https://github.com/myeongjae-kim/nodejs-tutorial-example/commit/145a603e565294b357834a083bd711c5610ec554)

`rush clean`을 해서 기존 빌드를 지우고 `rush build`가 잘 되는지 확인합니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ rush build

src/article/view/cli/MenuPrinter.ts(2,33): error TS2307: Cannot find module 'board-domain/dist/article/port/incoming/ArticleResponse' or its corresponding type declarations.

Operations failed.

rush build (5.36 seconds)
```

안되네요... `app/board-cli`에서 `domain/board-domain`의 빌드 결과물을 찾지 못합니다. 도메인 패키지의 빌드 결과를 확인해보니 소스 코드와 디렉토리
구조가 약간 다릅니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ tree . -I node_modules -I __test__ -L 3 -d
.
├── dist
│   ├── model
│   └── port
│       ├── incoming
│       └── outgoing
└── src
    └── article
        ├── model
        └── port

12 directories
```

`src/article`의 내용물이 `dist/article`이 아니라 그냥 `dist`로 올라와버렸네요. `src` 디렉토리가 비어있기 때문에 이런 현상이 발생합니다.
`tsconfig.json`에서 `rootDir`를 `./src`로 지정해주면 `src`의 구조 그대로 `dist`에 컴파일 결과물을 생성합니다.

```json-doc
// doamin/board-domain/tsconfig.json
{
  "compilerOptions": {
    ...,
    "rootDir": "./src",
    ...
  }
}
```

빌드를 합니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ rush build

==[ board-domain ]=================================================[ 1 of 2 ]==
"board-domain" completed successfully in 4.39 seconds.

==[ app-board-cli ]================================================[ 2 of 2 ]==
"app-board-cli" completed successfully in 5.91 seconds.



==[ SUCCESS: 2 operations ]====================================================

These operations completed successfully:
  app-board-cli    5.91 seconds
  board-domain     4.39 seconds


rush build (10.34 seconds)
```

빌드가 잘 되네요. 트리를 출력해보면 확인해보면 `src`와 `dist`의 구조가 동일합니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ tree . -I node_modules -I __test__ -L 3 -d
.
├── dist
│   └── article
│       ├── model
│       └── port
└── src
    └── article
        ├── model
        └── port

8 directories
```

`app/board-cli` 디렉토리로 이동해서 애플리케이션 실행이 잘 되는지 `rushx start`로 확인합니다.

```
~/nodejs-tutorial-example-rush/domain/board-domain$ cd ../../app/board-cli
~/nodejs-tutorial-example-rush/app/board-cli$ rushx start
1) 목록 조회
2) 쓰기
x) 종료

선택: 
```

실행도 잘 됩니다.

지금까지 작성한 코드는 [nodejs-tutorial-example:chapter-5-use-domain-package](https://github.com/myeongjae-kim/nodejs-tutorial-example/tree/chapter-5-use-domain-package)에서 확인할 수 있습니다.

### 배포용 `tar` ball 만들기

로컬에서 빌드하고 실행까지 잘 되는건 확인했습니다. 배포용 압축 파일은 어떻게 만들 수 있을까요? `node_modules`의 디렉토리가 심볼릭 링크로 연결되어 있기
때문에 `app/board-domain/node_modules`가 압축 파일에 포함되면 `app/board-domain/node_modules/board-domain`도 포함되고, 다시
`app/board-domain/node_modules/board-domain/node_modules`도 포함되고... 압축파일의 크기가 무진장 늘어나게 됩니다. 그렇다고 심볼릭 링크만
압축해버리면 원본을 찾을수가 없으니, 원본을 포함해서 압축하려면 프로젝트 전체를 압축해야 합니다. 세상에.. 아무도 그걸 원하진 않을거에요. 그리고 개발용
의존성을 제거하고 실행용 의존성만 `node_modules`에 포함해서 배포해야 할 코드의 양도 줄여야 합니다.

이 모든것을 위해서 [Rush는 `rush deploy`라는 커맨드를 준비](https://rushjs.io/pages/maintainer/deploying/)해놨습니다. Rush가 Lerna에
비해서 편한 부분이 많지만 정말 배포용 빌드를 만드는 과정은 Lerna에 비해서 훨씬 편합니다.

먼저 `rush init-deploy`로 배포 설정을 추가합니다.

```
~/nodejs-tutorial-example-rush$ rush init-deploy --project app-board-cli

Starting "rush init-deploy"

Creating scenario file: .../nodejs-tutorial-example-rush/common/config/rush/deploy.json

File successfully written. Please review the file contents before committing.
```

`common/config/rush/deploy.json`파일이 생성되었습니다. 이 파일을 따로 수정해주진 않아도 되고, 배포용 빌드에 `app/board-cli/dist/index.js`
뿐만 아니라 `app/board-cli/dist` 디렉토리 전체가 포함되도록 `app/board-cli/package.json`에 `files` 속성을 추가합니다. 

```json-doc
// app/board-cli/package.json
{
  ...,
  "files": [
    "dist"
  ],
  ...
}
```

`rush deploy`를 입력하면 `common/config/rush/deploy.json`을 바탕으로 배포용 빌드를 `common/deploy/app/board-cli`에 생성합니다.

```
~/nodejs-tutorial-example-rush$ rush deploy

Starting "rush deploy"

Loading deployment scenario: /Users/mj/projects/nodejs-tutorial-example-rush/common/config/rush/deploy.json
Deploying to target folder:  /Users/mj/projects/nodejs-tutorial-example-rush/common/deploy
Main project for deployment: app-board-cli

Analyzing project: app-board-cli

Copying folders...
Writing deploy-metadata.json
Creating symlinks...

The operation completed successfully.

~/nodejs-tutorial-example-rush$ node common/deploy/app/board-cli/dist/index.js
1) 목록 조회
2) 쓰기
x) 종료

선택: 
```

실행이 잘 됩니다. `node_modules`를 확인해보면 `devDependencies`의 개발 의존성 없이 `dependencies`의 의존성만 포함되어 있음을 확인할 수 있습니다.

```
common/deploy
├── app
│   └── board-cli
│       ├── dist
│       └── node_modules
│           ├── @reduxjs
│           ├── board-domain -> ../../../domain/board-domain
│           ├── inversify -> ../../../common/temp/node_modules/inversify
│           ├── mobx -> ../../../common/temp/node_modules/mobx
│           └── reflect-metadata -> ../../../common/temp/node_modules/reflect-metadata
├── common
│   └── temp
│       └── node_modules
│           ├── @babel
│           ├── @reduxjs
│           ├── immer
│           ├── inversify
│           ├── mobx
│           ├── redux
│           ├── redux-thunk
│           ├── reflect-metadata
│           ├── regenerator-runtime
│           └── reselect
└── domain
    └── board-domain
        ├── dist
        └── src
```

`board-domain`은 `dependencies`가 없어서 `node_modules` 디렉토리 자체가 없군요. `board-cli/node_modules`를 보면 심볼릭 링크로
`common/temp/node_modules`이하의 디렉토리와 `domain/board-domain` 디렉토리를 참조하는 것을 확인할 수 있습니다. 결국 `common/deploy`
디렉토리는 그 자체로 완결성을 가지므로 빌드 결과물과 호환되는 버전의 node가 설치되어 있는 환경이라면 어디서든지 `app/board-cli/dist/index.js`를
실행해서 우리의 게시판 애플리케이션을 실행할 수 있게 됩니다.

```
~/nodejs-tutorial-example-rush$ tar -czvf bundle.tar common/deploy
~/nodejs-tutorial-example-rush$ mv bundle.tar ~/Downloads && cd ~/Downloads
~/Downloads$ tar -xzvf bundle.tar
~/Downloads$ node common/deploy/app/board-cli/dist/index.js # 이하 4개의 커맨드는 모두 동일한 의미를 가진다.
~/Downloads$ node common/deploy/app/board-cli/dist/index    # index.js 파일을 실행한다.
~/Downloads$ node common/deploy/app/board-cli/dist          # dist 디렉토리의 index.js 파일을 실행한다.
~/Downloads$ node common/deploy/app/board-cli               # package.json의 `main` 속성 값인 dist/index.js 파일을 실행한다.
1) 목록 조회
2) 쓰기
x) 종료

선택: 
```

빌드 결과물을 압축해서 `~/Downlaods` 디렉토리에 배포하고 실행해봤습니다.

모노레포에서 복수의 애플리케이션을 관리해야 한다면, 예를들어 게시판 애플리케이션을 `app/board-react` 디렉토리에 리액트로 구현했다면
[기존의 `deploy.json`의 `"deploymentProjectNames"`에 프로젝트 이름을 추가](https://rushjs.io/pages/maintainer/deploying/#multiple-deployments-using-the-same-config-file)할 수도 있고,
[`deploy-app-board-api.json`같이 새로운 배포 설정용 json을 추가](https://rushjs.io/pages/maintainer/deploying/#multiple-deployments-using-different-config-files)해서 빌드를 할 수 도 있습니다.

### 현실세계의 모노레포

현실에선 지금보다 더 많은 패키지가 필요합니다. 프로젝트 전체에서 공통으로 쓰이는 `core` 패키지, 데이터베이스나 메시지 큐등 외부 의존성에 대한 설정이
담겨 있는 `configruation/xxx` 패키지, 혹은 원격 API를 호출하기 위한 `client/xxx-client` 등 필요에 따라 추가로 패키지를 구성하면 됩니다. 

노드 진영에서는 하나의 저장소 안에 여러 개의 프로젝트가 있는 구조를 모노레포라 부르지만 JVM 진영에서는 멀티모듈이라고 부릅니다. 권용근님의 글
[멀티모듈 설계 이야기 with Spring, Gradle](https://techblog.woowahan.com/2637/)은 제가 실무에서 프로젝트를 구성하면서 모듈을 나누는 기준을
세울 때 정말 많은 도움을 받았던 글입니다.

클래스든 패키지든 모듈이든 아키텍처의 목표는 동일합니다: **높은 응집도와 낮은 결합도를 가지도록 코드를 나누고 비즈니스 로직이 세부사항에 오염되지 않도록 보호하기.**
아키텍처가 목표를 제대로 달성한다면 우리는 [**비즈니스 요구사항에 빠르게 대응할 수 있는 유연한 프로그램**](/#결국엔-비즈니스)이라는 이상에 한 걸음 다가갈 수 있게 됩니다.

지금까지 작성한 코드는 [nodejs-tutorial-example:chapter-5-deploy](https://github.com/myeongjae-kim/nodejs-tutorial-example/tree/chapter-5-deploy)에서 확인할 수 있습니다.

## PNPM 사용하기

다행히 npm을 사용해도 우리에게 필요한 Rush의 기능은 정상작동을 합니다. 하지만 [속도도 빠르고 디스크 용량도 적게 차지하고 Rush에서 권장하는 pnpm](#pnpm)을 써보면
어떨까요? [https://pnpm.io/installation](https://pnpm.io/installation)을 참고해서 pnpm을 설치합니다. 저는 macOS를 사용하고 있으므로
`brew install pnpm`으로 설치하겠습니다.

```
~/nodejs-tutorial-example-rush$ brew install pnpm
...
~/nodejs-tutorial-example-rush$ pnpm --version
6.32.4
```

프로젝트 root에 있는 `rush.json`에서 `"npmVersion": "6.14.16"` 대신 `"pnpmVersion": "6.32.4"`를 사용하도록 변경합니다.

```json-doc
// rush.json
{
  ...,
  "pnpmVersion": "6.32.4",
  // "npmVersion": "6.14.16",
  ...
}
```

[`rush update --full --purge`를 입력](https://rushjs.io/pages/maintainer/package_managers/#specifying-your-package-manager)합니다.

```
~/nodejs-tutorial-example-rush$ rush update --full --purge

ERROR: An unrecognized file "npm-shrinkwrap.json" was found in the Rush config folder:
/Users/mj/projects/nodejs-tutorial-example-rush/common/config/rush
```

음.. `npm-shrinkwrap.json` 때문에 실패하는군요. 이 파일은 뭘까요?

> What is this "shrinkwrap file"?
> 
> Most projects don't specify an exact version such as 1.2.3 for a dependency, but instead specify SemVer range such as 1.x or ^1.2.3. By itself, this would mean that what gets installed depends on the latest version at the time. Such nondeterminism is bad: It would be maddening for a Git branch that built on Monday to mysteriously be failing on Tuesday because of a new release of a library. The shrinkwrap file solves this problem by storing a complete installation plan in a large file that is tracked by Git.
> 
> The shrinkwrap file has different names depending on the package manager that your repo is using: shrinkwrap.yaml, npm-shrinkwrap.json, or yarn.lock
> 
> \- [Everyday commands \| Rush](https://rushjs.io/pages/developer/everyday_commands/)


의존성 버전에 caret(`^`)이나 tilde(`~`)등을 사용하면 동일한 버전이라도 언제 `npm install`을 하는가에 따라서 다른 버전이 설치될 수 있기 때문에
앱이 제대로 작동하지 않을 수 있습니다. 이 문제를 해결하기 위해 npm은 `package-lock.json`을 추가해서 `package-lock.json`이 변경되지 않는 이상
같은 버전이 설치되도록 제한합니다. Rush에서는 `npm-shrinkwrap.json`이 동일한 역할을 합니다. pnpm을 사용하면 `pnpm-lock.yaml`을 사용합니다.
`rush.json`은 pnpm을 사용하도록 변경했지만 `npm-shrinkwrap.json`파일이 이미 있으니까 `rush update --full --purge`가 실패했습니다.

저희는 [도메인 패키지를 추가](#도메인-패키지-추가)하면서 `dependencies`의 버전이 여러 개로 해석될 수 없게끔 변경했으니 마음놓고
`npm-shrinkwrap.json`을 삭제해도 됩니다.

```
~/nodejs-tutorial-example-rush$ rm common/config/rush/npm-shrinkwrap.json
~/nodejs-tutorial-example-rush$ rush update --full --purge

The command failed:
 /Users/mj/projects/nodejs-tutorial-example-rush/common/temp/pnpm-local/node_modules/.bin/pnpm install --store /Users/mj/projects/nodejs-tutorial-example-rush/common/temp/pnpm-store --no-prefer-frozen-lockfile --recursive --link-workspace-packages falseERROR: Error: The command failed with exit code 1

Giving up after 3 attempts
```

그래도 저는 뭐가 잘 안되네요... 기존에 설치되어있던 node_modules 디렉토리 때문에 충돌이 발생하는 것 같으니 찾아서 모두 제거하겠습니다.

```
~/nodejs-tutorial-example-rush$ find . -type d -name "node_modules" -prune
./app/board-cli/node_modules
./common/temp/node_modules
./common/deploy/app/board-cli/node_modules
./common/deploy/common/temp/node_modules
./domain/board-domain/node_modules

# node_modules를 잘 찾는 것을 확인했으니 과감히 rm -rf로 지워준다.
~/nodejs-tutorial-example-rush$ find . -type d -name "node_modules" -prune | xargs rm -rf
~/nodejs-tutorial-example-rush$ rush update --full --purge
...
 WARN  Issues with peer dependencies found
../../app/board-cli
└─┬ ts-node
  └── ✕ missing peer @types/node@"*"
Peer dependencies that should be installed:
  @types/node@"*"  

../../domain/board-domain
└─┬ ts-node
  └── ✕ missing peer @types/node@"*"

Peer dependencies that should be installed:
  @types/node@"*"  

Copying "/Users/mj/projects/nodejs-tutorial-example-rush/common/temp/pnpm-lock.yaml"
  --> "/Users/mj/projects/nodejs-tutorial-example-rush/common/config/rush/pnpm-lock.yaml"


Rush update finished successfully. (27.10 seconds)
```

드디어 `rush update`가 성공을 했습니다. 하지만 Peer dependency 관련 경고가 발생했습니다. `ts-node`가 `@types/node`를 사용하지만 저희가
명시적으로 의존성을 추가하지 않았기 때문에 발생하는 경고입니다. [Rush는 pnpm을 사용하는 경우 `rush.json`에 `"strictPeerDependencies"` 옵션을 켜도록 권장](https://rushjs.io/pages/maintainer/recommended_settings/#strictpeerdependencies)하고 있습니다.
옵션을 켜고 다시 `rush update --full`을 해보겠습니다.

```json-doc
// rush.json
{
  ...,
  "pnpmOptions": {
    "strictPeerDependencies": true
  }
}
```

```
~/nodejs-tutorial-example-rush$ rush update --full

...
 ERR_PNPM_PEER_DEP_ISSUES  Unmet peer dependencies

../../app/board-cli
└─┬ ts-node
  └── ✕ missing peer @types/node@"*"
Peer dependencies that should be installed:
  @types/node@"*"
  
../../domain/board-domain
└─┬ ts-node
  └── ✕ missing peer @types/node@"*"
Peer dependencies that should be installed:
  @types/node@"*"
```

이번에는 경고가 아니라 에러가 발생하면서 `rush update --full`이 실패합니다. 의존성 관리를 좀 더 빡빡하게 할 수 있어서 아주 만족스럽습니다.
`app/board-cli`와 `domain/board-domain`의 `devDependencies`에 `@types/node`를 추가한 뒤 `rush update --full`을 입력합니다. 
`npm show @types/node version`를 입력하면 `@types/node`의 최신버전을 알 수 있습니다. 저는 `"@types/node": "^17.0.23"`를 추가하겠습니다.


```
~/nodejs-tutorial-example-rush$ rush update --full

...

Scope: all 3 workspace projects
.                                        | +559 ++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Packages are hard linked from the content-addressable store to the virtual store.
  Content-addressable store is at: .../nodejs-tutorial-example-rush/common/temp/pnpm-store/v3
  Virtual store is at:             node_modules/.pnpm
Progress: resolved 559, reused 559, downloaded 0, added 559, done
node_modules/.pnpm/nodemon@2.0.15/node_modules/nodemon: Running postinstall script, done in 179ms

Copying ".../nodejs-tutorial-example-rush/common/temp/pnpm-lock.yaml"
  --> ".../nodejs-tutorial-example-rush/common/config/rush/pnpm-lock.yaml"

Rush update finished successfully. (22.03 seconds)
```

이제 에러 없이 잘 되는군요! `rush build && rush deploy`로 빌드까지 잘 되는지 확인해보겠습니다.

```
~/nodejs-tutorial-example-rush$ rush build && rush deploy

> app-board-cli@1.0.0 lint /Users/mj/projects/nodejs-tutorial-example-rush/app/board-cli
> eslint . --ext .ts,.tsx

src/article/view/cli/state-modules/redux/MyStore.ts(1,42): error TS2307: Cannot find module 'redux' or its corresponding type declarations.


Operations failed.

rush build (10.35 seconds)
```

빌드 과정에서 문제가 발생했습니다. `MyStore.ts`에서 사용하는 `redux` 모듈을 찾을 수 없다고 나옵니다. `app/board-cli/package.json`의 의존성을
보면 `@reduxjs/toolkit`만 있을 뿐 `redux`는 없는데, 지금까지 어떻게 `redux`를 사용하고 있었을까요?

```typescript
import { ActionFromReducer, Store } from "redux"; // pnpm 덕분에 잘못된 import를 발견!
import * as reduxModule from "./redux-module";

export type MyStore = Store<
  reduxModule.State,
  ActionFromReducer<typeof reduxModule.reducer>
>;
```

```json-doc
// app/board-cli/package.json
{
  ...,
  "dependencies": {
    "@reduxjs/toolkit": "1.8.0", // 이 의존성만 있고 redux는 없다.
    "inversify": "6.0.1",
    "mobx": "6.4.2",
    "reflect-metadata": "0.1.13",
    "board-domain": "workspace:*" // pnpm을 사용하도록 변경하고 rush update --full --purge를 하면서 "1.0.0"대신 "workspace:*"로 변경되었다.
  },
  "devDependencies": {
    "@johanblumenberg/ts-mockito": "^1.0.32",
    "@types/jest": "^27.4.1",
    "@types/node": "^17.0.23",
    "@typescript-eslint/eslint-plugin": "^5.14.0",
    "@typescript-eslint/parser": "^5.14.0",
    "dependency-cruiser": "^11.4.1",
    "eslint": "^8.11.0",
    "eslint-config-prettier": "^8.5.0",
    "jest": "^27.5.1",
    "nodemon": "^2.0.15",
    "prettier": "2.5.1",
    "ts-jest": "^27.1.3",
    "ts-node": "^10.7.0",
    "typescript": "^4.6.2"
  }
}
```

[`@reduxjs/toolkit`의 의존성](https://www.npmjs.com/package/@reduxjs/toolkit?activeTab=dependencies)을 확인해보면 `redux`를
찾을 수 있습니다. npm을 사용중이었다면 의존성(`@reduxjs/toolkit`)의 의존성(`redux`)까지 `node_modules` 디렉토리에 평평하게 펼쳐지기 때문에
이제까지는 직접 `redux`를 의존하지 않더라도 `redux` 패키지를 사용할 수 있었습니다. 이것이 [Phantom Dependencies](https://rushjs.io/pages/advanced/phantom_deps/)
문제 입니다.

pnpm을 사용하게 되면 `app/board-cli/node_modules`에는 `redux` 없이 `@reduxjs/toolkit`만 설치가 됩니다.

```
~/nodejs-tutorial-example-rush$ tree app/board-cli/node_modules -d
app/board-cli/node_modules
├── @johanblumenberg
│   └── ts-mockito -> ../../../../common/temp/node_modules/.pnpm/...
├── @reduxjs
│   └── toolkit -> ../../../../common/temp/node_modules/.pnpm/...
├── @types
│   ├── jest -> ../../../../common/temp/node_modules/.pnpm/...
│   └── node -> ../../../../common/temp/node_modules/.pnpm/...
├── @typescript-eslint
│   ├── eslint-plugin -> ../../../../common/temp/node_modules/.pnpm/...
│   └── parser -> ../../../../common/temp/node_modules/.pnpm/...
├── board-domain -> ../../../domain/board-domain
├── dependency-cruiser -> ../../../common/temp/node_modules/.pnpm/...
├── eslint -> ../../../common/temp/node_modules/.pnpm/...
├── eslint-config-prettier -> ../../../common/temp/node_modules/.pnpm/...
├── inversify -> ../../../common/temp/node_modules/.pnpm/...
├── jest -> ../../../common/temp/node_modules/.pnpm/...
├── mobx -> ../../../common/temp/node_modules/.pnpm/...
├── nodemon -> ../../../common/temp/node_modules/.pnpm/...
├── prettier -> ../../../common/temp/node_modules/.pnpm/...
├── reflect-metadata -> ../../../common/temp/node_modules/.pnpm/...
├── ts-jest -> ../../../common/temp/node_modules/.pnpm/...
├── ts-node -> ../../../common/temp/node_modules/.pnpm/...
└── typescript -> ../../../common/temp/node_modules/.pnpm/...

23 directories
```

`node_modules`에 23개의 디렉토리밖에 없다니... [이전에 확인했을 때는 435개의 디렉토리 있었는데](#pnpm) 참 대조적이네요. 디렉토리들도 모두 심볼릭
링크로 연결되어 있기 때문에 모노레포 상황이더라도 동일한 의존성이 여러 `node_modules`에 설치되는 현상을 해결할 수 있게 되었습니다. 디스크 사용량도
줄어들고 의존성을 설치할 때도 한 곳(`common/temp/node_modules/.pnpm`)에만 설치하기 때문에 속도도 빠릅니다. IDE에서도 `import` 문을 자동으로
추가할 때 인덱싱해야 할 `node_modules` 의존성 개수가 확 줄어들기 때문에 개발 환경도 쾌적해집니다.

`MyStore.ts`에서 `redux`대신 `@reduxjs/toolkit`을 사용하도록 변경하고 다시 빌드합니다.

```typescript
import { ActionFromReducer, Store } from "@reduxjs/toolkit"; // redux 대신 @reduxjs/toolkit을 사용
import * as reduxModule from "./redux-module";

export type MyStore = Store<
  reduxModule.State,
  ActionFromReducer<typeof reduxModule.reducer>
>;
```

```
~/nodejs-tutorial-example-rush$ rush build && rush deploy

Starting "rush deploy"

Loading deployment scenario: /Users/mj/projects/nodejs-tutorial-example-rush/common/config/rush/deploy.json
Deploying to target folder:  /Users/mj/projects/nodejs-tutorial-example-rush/common/deploy
Main project for deployment: app-board-cli


ERROR: The deploy target folder is not empty. You can specify "--overwrite" to recursively delete all folder contents.
```

빌드 과정은 성공했지만 배포 디렉토리가 비어있지 않다고 에러가 발생했습니다. `--overwrite` 옵션을 추가해서 배포하고 애플리케이션을 실행해보겠습니다.

```
~/nodejs-tutorial-example-rush$ rush deploy --overwrite

...
The operation completed successfully.
~/nodejs-tutorial-example-rush$ node common/deploy/app/board-cli
1) 목록 조회
2) 쓰기
x) 종료

선택: 
```

실행이 잘 됩니다.

지금까지 작성한 코드는 [nodejs-tutorial-example:chapter-5-use-pnpm](https://github.com/myeongjae-kim/nodejs-tutorial-example/tree/chapter-5-use-pnpm)에서 확인할 수 있습니다.
