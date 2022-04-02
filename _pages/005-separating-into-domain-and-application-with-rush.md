---
title: 5. 도메인과 애플리케이션의 분리 (with Rush)
author: Myeongjae Kim
date: 2022-03-12
category: Tutorial
layout: post
---

## Lerna와 Rush

node 기반 프로젝트를 모노레포로 구성할 수 있게 도와주는 도구가 여럿 있습니다. 저는 실무에서 2020년에 [Lerna](https://lerna.js.org)를 사용해
모노레포로 프로젝트를 구성한 경험이 있습니다. JVM 진영의 [gradle](https://gradle.org)의 멀티모듈 구조에 익숙해 있던 터라 모노레포의 개념을 이해하기는
쉬웠지만, Lerna로 프로젝트를 구성해보니 설정이 복잡했고 `npm`만으로는 해결할 수 없는 문제가 있어서
[직접 심볼릭 링크를 업데이트하는 쉘 스크립트를 작성해서 사용](https://myeongjae.kim/blog/2020/06/12/copy-symbolic-link-of-relative-path)했습니다(이 문제는 Lerna와 yarn workspace를 함께 사용하는 방식으로도 해결이 가능합니다).

에듀테크 스타트업 밀당의 정성대님이 작성하신 [Rush로 프론트엔드 모노레포 도입기](https://medium.com/mildang/rush로-프론트엔드-모노레포-도입기-5da0c5bc9b30)
에서는 제가 말씀드릴 수 있는 것보다 훨씬 풍부하고 깊은 이야기가 있습니다. 정성대님의 글에서는 여러 개의 모노레포 관리 도구들을 비교하고
[Rush](https://rushjs.io)를 선택해서 React 버전을 점진적으로 업데이트합니다. 저도 2020년에 Lerna와 Rush중에 고민하다가 Rush가 더 나아보여서
Rush로 프로젝트 구성을 시도해봤는데, 이 때는 Next.js와 Rush가 호환이 되지 않아 Lerna를 사용했었습니다. 정성대님의 예시에서는 Rush에서
Next.js가 잘 작동합니다(그냥 그 때 제가 설정을 제대로 못한 것 같습니다). 

Rush는 [npm](https://docs.npmjs.com), [pnpm](https://pnpm.io), [yarn](https://yarnpkg.com) 모두 사용할 수 있지만 pnpm을
권장합니다 ([NPM vs PNPM vs Yarn \| Rush](https://rushjs.io/pages/maintainer/package_managers/)). npm은 옛날 버전인 4.5.0을
권장하고, yarn은 사용은 가능하지만 [왠지 자신없어 하는 모습](https://rushjs.io/pages/maintainer/package_managers/#considerations-for-yarn)입니다.
우리의 게시판 애플리케이션 정도는 아마 npm 최신 버전에서도 잘 작동할 것 같아서 일단 프로젝트는 Rush와 npm을 사용한 뒤, 모노레포 구성 후에
npm에서 pnpm으로 마이그레이션을 해보겠습니다.

### PNPM?

pnpm은 npm같은 패키지 매니저입니다. pnpm은 npm을 사용하면서 겪을 수 있는
[Phantom dependencies](https://rushjs.io/pages/advanced/phantom_deps/) 문제와
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

`app/board-cli/dist/node_modules`를 확인해보면 의존성이 심볼릭 링크로 연결된 것을 확인할 수 있습니다.

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

- `domain/board-domain/article/domain`
- `domain/board-domain/article/application`

그냥 그대로 옮기면 이렇게 두 개의 디렉토리가 생길텐데, 도메인 패키지의 `domain` 디렉토리라니.. 디렉토리 경로에서 중복이 발생하는 것 같아 영 찜찜하니까
`domain` 디렉토리를 `model`로 변경하겠습니다.

- `domain/board-domain/article/model`
- `domain/board-domain/article/application`

`model` 디렉토리는 '도메인 모델'을 의미하게 됩니다. 즉 엔티티만 있는 디렉토리입니다. `application` 디렉토리는 엔티티를 바탕으로 수행하는 비즈니스
로직이 들어있는 디렉토리가 됩니다. 그리고 디렉토리 구조에서 `application`이라는 단어는 이미 `app`이라는 디렉토리로 `app/board-cli`처럼 실행 가능한
애플리케이션이 있는 패키지의 부모 디렉토리로 사용하고 있으니, 도메인 패키지에서 `application` 디렉토리가 존재하는 것도 영 찜찜합니다.

도메인패키지의 `application` 디렉토리에는 `incoming` 포트와 `outgoing` 포트, 그리고 `incoming` 포트 인터페이스를 구현하는 서비스들이
들어있습니다. 외부 의존성 없이 엔티티와 포트로만 수행하는 비즈니스 로직은 도메인의 범주 안에 있기 때문에 굳이 `application` 디렉토리 아래로 넣어놓지 말고
그냥 밖으로 꺼내놓으면 어떨까요? 아래와 같은 트리 구조도 괜찮아 보입니다.

```
domain/board-domain/article
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

패캐지마다 의존성 버전이 다르면 예상치 못한 문제가 발생할 수 있기 때문에 Rush에서는 [`common/config/rush/common-versions.json`](https://rushjs.io/pages/configs/common-versions_json/)을 제공해서 의존성 버전을 통일할 수 있게 도와줍니다.
`dependencies`에서 사용하는 의존성은 `range` 없이 특정한 버전을, `devDependencies`에서 사용하는 의존성은 자동으로 `MINOR`패치까지 업데이트
하는 caret(`^`)을 사용하도록 지정합니다.

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
    "clean": "rm -rf dist/*"
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
    <figcaption><code>board-cli</code>패키지에서 <code>board-domain</code>패키지의 코드를 사용할 수 있다.</figcaption>
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

### `board-cli`에서 도메인 분리하기

[이전에 설계했던 대로](/pages/005-separating-into-domain-and-application-with-rush/#도메인-패키지-디렉토리-설계) `board-cli`에 있는
도메인 영역을 `board-domain`패키지로 가져옵니다.

TODO

## 중복 설정 제거하기

## PNPM 적용
