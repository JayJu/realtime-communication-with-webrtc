# 7. 메시지 교환을 위한 시그널링 서비스 설정하기

## 여기서 배우게 될 것

* ```npm```을 사용하여 package.json에 지정된대로 프로젝트 의존관계들을 설치하는 법
* 노드 서버를 실행하고 node-static을 사용하여 정적 파일을 제공하는 법
* socket.io를 사용하여 노드에 메시징 서비스 설정하기
* 이를 사용하여 '방'을 개설하고 메시지를 교환하는 법

전체 소스는 **step-04** 디렉토리에 있다.


## 개념잡기
WebRTC 호출을 설정하고 유지하려면 WebRTC 클라이언트(피어)가 메타 데이터를 교환해야 한다.
* 후보자(candidate) 네트워크 정보.
* 해상도 및 코덱과 같은 미디어에 대한 정보를 제공하는 **제공(offer)** 과 **응답(answer)** 메시지.

즉, 오디오, 비디오 또는 데이터의 피어 투 피어 스트리밍을 하기위해 메타 데이터 교환이 필요하며 이 프로세스를 **시그널링(signaling)** 이라고 한다.

이전 장에서는 보낸 사람과 받는 사람 RTCPeerConnection 객체가 같은 페이지에 있으므로 '시그널링'은 단순히 개체간에 메타 데이터를 전달하는 문제였다.

실제 응용 프로그램에서는 보낸 사람과 받는 사람 RTCPeerConnections가 서로 다른 장치의 웹 페이지에서 실행되므로 메타 데이터를 전달할 수있는 방법이 필요하다.

이를 위해 우리는 WebRTC 클라이언트(피어)간에 메시지를 전달할 수 있는 서버인 **시그널링 서버(signaling server)** 를 사용합니다. 실제 메시지는 일반 텍스트(문자열로 된 JavaScript 객체) 이다.

## 어플리케이션에 대해
이번 장에서는 Socket.IO 노드 모듈과 메시징용 JavaScript 라이브러리를 사용하여 간단한 Node.js 시그널링 서버를 구축한다. Node.js 및 Socket.IO에 대한 경험은 유용하지만 이번 장을 이해하는 데 중요하지는 않다. 메시징 구성 요소는 매우 간단하다.

> **올바른 시그널링 서버 선택하기**
>
> 이 코드 랩은 시그널링 서버에 [Socket.IO](http://socket.io/)를 사용한다.
>
> Socket.io는 메시지를 교환하는 서비스를 직관적으로 만들 수 있도록 설계되었고 , 기본적으로 '방(rooms)'이라는 개념이 있기 때문에 WebRTC 시그널링에 적합하다.
>
> 운영 서비스의 경우 더 나은 대안이 있을 수 있다. 다음 [WebRTC 프로젝트를 위한 시그널링 프로토콜을 선택하는 방법](https://bloggeek.me/siganling-protocol-webrtc/)을 참고하자.

이 예제에서 서버 (Node 응용 프로그램)는 index.js에 구현되고 실행되는 클라이언트 (웹 응용 프로그램)는 index.html에 구현된다.

이 단계의 노드 응용 프로그램에는 두 가지 작업이 있다.

첫째, 메시지 릴레이 역할을 한다.

``` javascript
socket.on('message', function (message) {
  log('Got message: ', message);
  socket.broadcast.emit('message', message);
});
```

둘째, WebRTC 화상 채팅 '방(rooms)'을 관리한다.

``` javascript
if (numClients === 1) {
  socket.join(room);
  socket.emit('created', room, socket.id);
} else if (numClients === 2) {
  socket.join(room);
  socket.emit('joined', room, socket.id);
  io.sockets.in(room).emit('ready');
} else { // max two clients
  socket.emit('full', room);
}
```
우리의 간단한 WebRTC 응용 프로그램은 최대 두 명의 피어가 방을 공유하도록 허용한다.

## HTML & Javascript
**index.html** 다음과 같이 수정한다.

``` javascript
<!DOCTYPE html>
<html>

<head>

  <title>Realtime communication with WebRTC</title>

  <link rel="stylesheet" href="css/main.css" />

</head>

<body>

  <h1>Realtime communication with WebRTC</h1>

  <script src="/socket.io/socket.io.js"></script>
  <script src="js/main.js"></script>
  
</body>

</html>

```

이 페이지에서는 아무 것도 볼 수 없으며 모든 로깅은 브라우저 콘솔에서 수행된다. (Chrome에서 콘솔을 보려면 Ctrl-Shift-J 또는 Mac의 경우 Command-Option-J 를 누른다.)

js/main.js를 다음으로 대체한다.

``` javascript
'use strict';

var isInitiator;

window.room = prompt("Enter room name:");

var socket = io.connect();

if (room !== "") {
  console.log('Message from client: Asking to join room ' + room);
  socket.emit('create or join', room);
}

socket.on('created', function(room, clientId) {
  isInitiator = true;
});

socket.on('full', function(room) {
  console.log('Message from client: Room ' + room + ' is full :^(');
});

socket.on('ipaddr', function(ipaddr) {
  console.log('Message from client: Server IP address is ' + ipaddr);
});

socket.on('joined', function(room, clientId) {
  isInitiator = false;
});

socket.on('log', function(array) {
  console.log.apply(console, array);
});
```

## Socket.IO 설정 후 Node 실행

이 단계와 다음 단계에서는 노드에서 Socket.IO를 실행한다.

**작업** 디렉토리의 최상위 레벨에서 다음 내용으로 **package.json** 파일을 작성하자.

``` javascript
{
  "name": "webrtc-codelab",
  "version": "0.0.1",
  "description": "WebRTC codelab",
  "dependencies": {
    "node-static": "0.7.7",
    "socket.io": "1.2.0"
  }
}
```
이것은 노드 패키지 관리자(```npm```)에게 설치할 프로젝트 종속성을 알려주는 응용 프로그램 목록이다.

종속성을 설치하려면 작업 디렉토리의 명령 행 터미널에서 다음을 실행한다.

``` shellscript
npm install
```

다음과 같은 설치 로그가 나타난다.
![](/img/ch701.png)

위와 같이 npm은 **package.json** 에 정의 된 종속성을 설치한다.

경고가 표시 될 수 있지만 빨간색 에러가 나타난다면 도움을 요청하자!

**작업** 디렉토리의 최상위 레벨 (**js** 디렉토리 아님)에 **index.js** 라는 새 파일을 작성하고 다음 코드를 추가한다.

``` javascript
'use strict';

var os = require('os');
var nodeStatic = require('node-static');
var http = require('http');
var socketIO = require('socket.io');

var fileServer = new(nodeStatic.Server)();
var app = http.createServer(function(req, res) {
  fileServer.serve(req, res);
}).listen(8080);

var io = socketIO.listen(app);
io.sockets.on('connection', function(socket) {

  // convenience function to log server messages on the client
  function log() {
    var array = ['Message from server:'];
    array.push.apply(array, arguments);
    socket.emit('log', array);
  }

  socket.on('message', function(message) {
    log('Client said: ', message);
    // for a real app, would be room-only (not broadcast)
    socket.broadcast.emit('message', message);
  });

  socket.on('create or join', function(room) {
    log('Received request to create or join room ' + room);

    var numClients = io.sockets.sockets.length;
    log('Room ' + room + ' now has ' + numClients + ' client(s)');

    if (numClients === 1) {
      socket.join(room);
      log('Client ID ' + socket.id + ' created room ' + room);
      socket.emit('created', room, socket.id);

    } else if (numClients === 2) {
      log('Client ID ' + socket.id + ' joined room ' + room);
      io.sockets.in(room).emit('join', room);
      socket.join(room);
      socket.emit('joined', room, socket.id);
      io.sockets.in(room).emit('ready');
    } else { // max two clients
      socket.emit('full', room);
    }
  });

  socket.on('ipaddr', function() {
    var ifaces = os.networkInterfaces();
    for (var dev in ifaces) {
      ifaces[dev].forEach(function(details) {
        if (details.family === 'IPv4' && details.address !== '127.0.0.1') {
          socket.emit('ipaddr', details.address);
        }
      });
    }
  });

});
```

명령 행 터미널에서 **작업** 디렉토리에서 다음 명령을 실행한다.

``` shellscript
node index.js
```

브라우저에서 **localhost:8080** 에 접속한다.

이 URL에 접속 할 때마다 회의실 이름을 입력하라는 메시지가 표시된다. 같은 방에 가려면 'foo'와 같은 방 이름을 매번 선택해야 한다.

새 탭 페이지를 열고 **localhost:8080** 에 다시 접속한다. 같은 방 이름을 선택한다.

세 번째 탭이나 창에서 **localhost:8080** 에 접속한다. 같은 방 이름을 다시 선택한다.

각 탭에서 콘솔을 확인한다. 위의 JavaScript에서 로깅을 확인해야 한다.

## 보너스 점수
1. 대안이 될 만한 메시징 메커니즘은 무엇일까? '순수한' WebSocket을 사용할 때 어떤 문제가 발생할 수 있는가?

2. 이 응용 프로그램을 확장할(scaling) 경우 발생 할 수 있는 문제는 무엇인가? 수천 또는 수백만 개의 동시 방(room) 요청을 테스트 할 수 있는 방법을 개발할 수 있을까?

3. 이 앱은 JavaScript 프롬프트로 회의실명을 입력받는다. URL에서 회의실 이름을 가져 오는 방법을 찾아보십시오. 예를 들어 _localhost:8080/foo_ 는 ```foo```라는 회의실 이름을 부여한다.

## 지금까지 배운 것들

이번 장에서는

* npm을 사용하여 package.json에 지정된대로 프로젝트 종속성을 설치하고
* 서버쪽 정적 파일들로 노드 서버를 실행하고
* socket.io를 사용하여 노드에 메시징 서비스를 설정하고
* 이를 사용하여 '회의실'을 만들고 메시지를 교환하는 방법

에 대해 배웠다. 전체 소스는 **step-04** 디렉토리에 있다.

## 좀 더 찾아보기
* [Socket.io 채팅 예제](https://github.com/rauchg/chat-example)
* [실제 세계의 WebRTC: STUN, TURN 및 시그널링](https://www.html5rocks.com/en/tutorials/webrtc/infrastructure/)
* [WebRTC에서 '시그널링' 이라는 용어](https://www.webrtc-experiment.com/docs/WebRTC-Signaling-Concepts.html)

## 다음단계
시그널링을 사용하여 두 명의 사용자가 피어연결을 수행하는 방법에 대해 알아보자.