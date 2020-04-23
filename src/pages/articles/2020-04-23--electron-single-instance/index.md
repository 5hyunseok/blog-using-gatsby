---
title: Electron 에서 single instance 만 실행되도록 하기
date: "2020-04-23T15:32:00.000Z"
layout: post
draft: false
path: "/posts/electron/single-instance"
category: "electron"
tags: 
  - "All"
description: ""
---


## Single instance?

프로그램이 한 프로세스만 켜지도록 만들고 싶을 때가 있다. 예를 들면 슬랙 데스크탑 앱의 경우 (windows 에서) 앱을 켜서 사용하고 있을 때 또 앱을 실행하게 되면 켜져 있던 앱을 focus 해줄 뿐 새로 앱을 띄워주지는 않는다.

electron 으로 구현된 앱 중에 atom[1] 같은 텍스트 에디터나 hyper[2] 같은 터미널 앱은 같은 프로그램을 여러 프로세스를 띄울 수 있는 사용자 경험이 필요하지만, slack 같은 메신저 앱 같은 경우에는 한 프로세스만 뜨도록 제한하는 사용자 경험이 더 자연스럽다.


그렇다면 electron 에서 single instance 제약을 만들어서 슬랙 같은 사용자 경험을 어떻게 만들 수 있는지 보자.

가장 먼저 electron app 에서 제공되는 single instance lock 을 요청한다. 

```javascript
import { app } from 'electron';

// single instance lock 요청
const gotTheLock = app.requestSingleInstanceLock();

if (!gotTheLock) {
  app.quit();
} else {
  // 성공적으로 single instance lock이 요청됨
}

```

그리고나서 'second-instance' 이벤트 핸들러를 구현하여 처리해 주면 된다.

second-instance 이벤트의 상세 명세는 다음 문서를 확인할 수 있다. ( https://www.electronjs.org/docs/api/app#event-second-instance )


```javascript
import { app } from 'electron';

// single instance lock 요청
const gotTheLock = app.requestSingleInstanceLock();

if (!gotTheLock) {
  app.quit();
} else {
  app.on('second-instance', () => {
    // Someone tried to run a second instance, we should focus our window.
    if (mainWindow) {
      if (mainWindow.isMinimized() || !mainWindow.isVisible()) {
        mainWindow.show();
      }
      mainWindow.focus();
    }
  });

}

```
나 같은 경우 second-instance 가 요청 되었을 경우에 main window 객체가 있는지 확인하고 가려져있으면 보여지게 하고 focus 되도록 하였다.

## 주의사항 - instance 의 기준?

이러한 feature를 구현하다가 알게 된 사실 중 유의하여야 하는 것이 있었다. electron 에서 어떤 앱의 single instance lock 을 걸 때 앱을 구분하는 기준은 package.json 에 적어놓은 `name` 프로퍼티이다. 

나의 경우 `electron-builder`[3] 를 이용하고 있었고, 환경변수를 통해 하나의 electron 프로젝트로 여러 다른 환경의 app 을 빌드해서 사용하고 있었다. 그런데 다른 환경의 app 을 빌드하면 다른 앱처럼 보여야 하기 때문에 각각 single instance 가 적용될거라고 생각했는데 package.json 의 `name` 프로퍼티가 같은 경우에는 installer도 다르고 설치경로도 다르고 설치된 executable 파일도 전부 다른데도 같은 앱으로 인식되어 두 앱을 동시에 켤 수가 없었다. 

[1]: https://atom.io/
[2]: https://hyper.is/
[3]: https://www.electron.build/