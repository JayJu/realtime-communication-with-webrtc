# 5. RTCPeerConnection을 이용한 동영상 스트리밍

## 여기서 배우게 될 것

* [adapter.js](https://github.com/webrtc/adapter)로 브라우저간 WebRTC 차이 메꾸기
* RTCPeerConneciton API를 이용해 동영상 스트리밍 하기
* 미디어 캡쳐와 스트리밍 제어하기

## RTCPeerConnection 이란?

RTCPeerConnection은 동영상과 음성, 그리고 데이터를 주고 받을 수 있도록 WebRTC를 구성하는 API 이다. 다음은 동일 페이지에서 두 개의 RTCPeerConnection 객체\(peer로 불리는\)끼리 연결을 맺는 예제이다. 
효율적인 방식은 아니지만 RTCPeerConnection이 어떻게 동작하는지 이해하는 데 좋다.

## video요소와 컨트롤 버튼 추가하기

**index.html** 에 있던 하나의 video 요소를 두개로 늘리고 버튼 3개를 추가한다.

```html
<video id="localVideo" autoplay></video>
<video id="remoteVideo" autoplay></video>

<div>
 <button id="startButton">Start</button>
 <button id="callButton">Call</button>
 <button id="hangupButton">Hang Up</button>
</div>

```

한 video 요소는 `getUserMedia()`로 부터 동영상 스트림을 출력하고 다른 하나는 RTCPeerConnection을 통해 동일 동영상을 출력한다. \(현실에서는 한쪽은 로컬, 다른 한쪽은 리모트에서 스트리밍된다\)

## adapter.js 끼워넣기

**main.js** 에 **adapter.js** 링크를 추가한다.

```javascript
<script src="js/lib/adapter.js"></script>
```

> **adapter.js**는 스펙의 변경이나 접두어의 차이로부터 앱을 보호하기 위한 일종의 쐐기이다.
> 
> 사실, WebRTC 구현체에서 사용하는 표준과 규약들은 매우 안정적이며 접두어가 붙은 이름들도 많지 않다.
> 
> 이 장에서는 **adapter.js**의 최신버전의 로컬사본을 연결하였다. - 연습용으로는 괜찮으나 운영서버에 적용하는것은 적절치 않다. [GitHub 레파지토리의 adapter.js ](https://github.com/webrtc/adapter)는 항상 최신버전의 기능에 대해서만 기술한다.
> 
> WebRTC의 상호운영성에\(interop\) 대한 모든 정보는 [webrtc.org\/web-apis\/interop](https://webrtc.org/web-apis/interop/)에서 확인 할 수 있다.

**index.html**은 아래와 같은 모습이 될 것이다.

```javascript
<!DOCTYPE html><html>

<head>

 <title>Realtime communication with WebRTC</title>

 <link rel="stylesheet" href="css/main.css" />

</head>

<body>

 <h1>Realtime communication with WebRTC</h1>

 <video id="localVideo" autoplay></video> 
 <video id="remoteVideo" autoplay></video>

 <div>
   <button id="startButton">Start</button> 
   <button id="callButton">Call</button> 
   <button id="hangupButton">Hang Up</button> 
 </div>

 <script src="js/lib/adapter.js"></script>
 <script src="js/main.js"></script>

</body>

</html>

```

## RTCPeerConnection 코드 설치하기

**main.js** 파일을 **step-02** 폴더의 버전으로 변경한다.

> 코드랩의 많은 코드들을 잘라내서 붙여넣기 하는것이 효율적인 방법은 아니지만 RTCPeerConnection을 실행하기 위해선 딱히 대안이 없다.
> 
> 이제 이 코드들이 어떻게 동작하는지 알아보자.

## 요청\(call\) 만들기

**index.html** 을 띄우고 웹캠에서 영상을 가져오기 위해 **start** 버튼을 클릭한다. 그런 다음 peer 커넥션을 생성하기 위해 **Call** 버튼을 클릭한다. 두개의 video에 동일한 영상이 보일 것이다. 브라우져 콘솔을 열어 WebRTC 로그들을 확인하자.

## 어떻게 동작할까

이번장은 꽤 많은 것들에 대해 알아야 한다.

> 이 부분을 건너 뛰고 싶다면, 그렇게 해도 상관 없다.
> 코드랩을 계속 하는 데 지장은 없다.

WebRTC는 **peer**라 부르는 WebRTC 클라이언트들 끼리 영상을 스트리밍 하기 위해 RTCPeerConnection 이라는 API를 사용하여 커넥션을 맺는다.

이 예제에서는 같은 페이지에 두 개의 RTCPeerConnection 객체\(`pc1`과`pc2`\)가 존재한다. 실용적인 방법은 아니지만 API가 어떻게 동작하는지 살펴보기엔 적당하다.

WebRTC peer들 간 call을 설정하기 위해선 3단계의 작업이 필요하다.

* RTCPeerConnection을 생성한 후 `getUserMedia()`로 가져온 로컬 스트림을 추가한다.
* 네크워크 정보\([ICE](https://en.wikipedia.org/wiki/Interactive_Connectivity_Establishment) 후보라 부르는 연결 가능한 단말\)를 가져와 공유한다. 
* 로컬과 리모트 정보\([SDP](https://en.wikipedia.org/wiki/Session_Description_Protocol) 포맷의 로컬 미디어 메타정보\)들을 가져와 공유한다: 

앨리스와 밥이 화상채팅을 하기 위해 RTCPeerConnection을 사용한다고 가정하자.

처음에 할 일은 서로의 네트워크 정보를 교환하는 것이다. '후보들을 찾는다' 라는 표현은 [ICE](https://en.wikipedia.org/wiki/Interactive_Connectivity_Establishment) 프레임워크를 사용하여 네트워크 인터페이스와 포트들을 찾는 절차를 의미한다.

1. 앨리스는 RTCPeerConnection 객체를 생성하고 `onicecandidate` 이벤트 핸들러를 등록한다. **main.js**의 아래 코드블럭에 해당한다.

    ```javascript
    pc1 = new RTCPeerConnection(servers); 
    trace('Created local peer connection object pc1'); 
    pc1.onicecandidate = function(e) {   
      onIceCandidate(pc1, e); 
};

    ```

    > 이번 예제에서는 RTCPeerConnection 의 `servers` 인자를 사용하지 않았다.
    > 
    > 이 부분은 STUN 과 TURN 서버들을 지정할 때 사용한다.
    > 
    > WebRTC는 유저들이 최단경로로 연결될 수 있도록 peer-to-peer 기반으로 설계되었지만 또한 현실 네트워킹\([NAT게이트웨이](https://en.wikipedia.org/wiki/NAT_traversal)나 방화벽 등\)에 잘 대응할 수 있도록 구현되어 있다.
    > 
    > 이 절차의 한 부분으로 WebRTC API들은 당신 컴퓨터의 IP주소를 가져오기 위해 STUN 서버를 사용하고 peer-to-peer 연결이 실패할 경우 를 대비해 릴레이 서버 역할을 할 수 있도록 TURN 서버를 사용한다. 상세한 정보는 [WebRTC in the real world](https://www.html5rocks.com/en/tutorials/webrtc/infrastructure)애서 확인 할 수 있다.

2. 앨리스는 `getUserMedia()`를 호출하고 전달받은 스트림을 다음과 같이 추가한다.

    ```javascript
    pc1.addStream(localStream);
    ```

3. 1단계의 `onicecandiate` 핸들러는 네트워크 후보들이 연결 가능한 상태가 될 때 실행된다.

4. 앨리스는 직렬화된 후보 데이터를 밥에게 보낸다. 실제 어플리케이션에서 이 절차\(**시그널링**으로 알려진\)는 메시징 서비스\(다음 단계에서 알아보도록 한다.\)를 통해 이루어진다. - 이 예제에서는RTCPeerConnection 객체들이 같은 페이지에 있기 때문에 외부 메시지는 필요하지 않다)

5. 밥은 앨리스로부터 후보 데이터를 받은 후 `addIceCandiate()`를 실행하여 후보 데이터를 리모트 peer 의 명세서에 추가한다.

    ``` javascript
    function onIceCandidate(pc, event) { 
      if (event.candidate) { 
        getOtherPc(pc).addIceCandidate( 
          new RTCIceCandidate(event.candidate) 
        ).then( 
          function() { 
            onAddIceCandidateSuccess(pc); 
          }, 
          function(err) { 
            onAddIceCandidateError(pc, err); 
          } 
        ); 

        trace(getName(pc) + ' ICE candidate: \n' 
      + event.candidate.candidate); 
      }    
    }

    ```
또한 WebRTC는 로컬과 리모트의 음성/영상 미디어 정보(해상도나 코덱정보 같은)를 확인하여 상호 교환해야 한다. 미디어 구성정보를 주고받는 시그널링은 **offer** 와 **answer** 로 불리는 blob으로 구성된 메타데이터들의 상호교환 절차이며 이 메타데이터들은 [SDP](https://en.wikipedia.org/wiki/Session_Description_Protocol)로 약칭되는 Session Description Protocol 포맷을 사용한다. 

    1. 앨리스는 RTCPeerConnection의 ```createOffer()``` 메서드를 실행한다. 반환된 promise는 RTCSessionDescription을 제공한다 : Alice의 로컬 세션 명세 :
    ``` javascript
    pc1.createOffer(
        offerOptions
        ).then(
        onCreateOfferSuccess,
        onCreateSessionDescriptionError
      );
    ```
    2. 성공하면 앨리스는 ```setLocalDescription()``` 을 사용하여 로컬 명세를 설정한 다음 이 세션 명세를 시그널링 채널을 통해 밥에게 전달한다.

    3. 밥은 앨리스가 보낸 명세를 ```setRemoteDescription()``` 을 사용하여 리모트 명세로 설정한다.

    4. 밥은 RTCPeerConnection의 ```createAnswer()``` 메소드를 실행하여 앨리스에게 받은 리모트 명세를 전달하여 로컬 세션을 생성할 수 있다. ```createAnswer()``` promise는 RTCSessionDescription을 전달한다. 밥은 이를 로컬 명세로 설정하고 Alice에게 보낸다.

    5. 앨리스가 밥의 세션 명세를 받게 되면 ```setRemoteDescription()``` 을 사용하여 리모트 명세로 설정한다.

    ``` javascript
    function onCreateOfferSuccess(desc) {
          pc1.setLocalDescription(desc).then(
            function() {
              onSetLocalSuccess(pc1);
            },
            onSetSessionDescriptionError
          );
          pc2.setRemoteDescription(desc).then(
             function() {
              onSetRemoteSuccess(pc2);
            },
            onSetSessionDescriptionError
          );
          // Since the 'remote' side has no media stream we need
          // to pass in the right constraints in order for it to
          // accept the incoming offer of audio and video.
          pc2.createAnswer().then(
              onCreateAnswerSuccess,
              onCreateSessionDescriptionError
          );
        }
    function onCreateAnswerSuccess(desc) {
        pc2.setLocalDescription(desc).then(
            function() {
                onSetLocalSuccess(pc2);
            },
            onSetSessionDescriptionError
          );
        pc1.setRemoteDescription(desc).then(
            function() {
                onSetRemoteSuccess(pc1);
            },
            onSetSessionDescriptionError
        );
    }
    ```
    6. Ping!

## 보너스 점수
1. **chrome://webrtc-internals** 를 확인해 보자. 이 메뉴는 WebRTC 통계와 디버깅 데이터들을 제공한다. (크롬의 전체 URL 목록은 **chrome://about** 에서 확인할 수 있다)

2. CSS로 페이지를 꾸며보자: 
  * 비디오를 나란히 배치 해 보자
  * 더 큰 텍스트와 함께 버튼을 동일한 너비로 만들어 보자
  * 모바일에서 레이아웃이 작동하는지 확인 해 보자

3. 크롬 개발자 도구에서 ```localStream```, ```pc1```, ```pc2``` 를 확인해 보자.

4. 콘솔에서 ```pc1.localDescription``` 을 확인해 보자. SDP 포맷이 어떻게 생겼는지 확인해 보자.

## 지금까지 배운 것들

이번 장에서는

* WebRTC용 쐐기(shim)인 adapter.js를 이용하여 브라우저간의 차이점을 추상화하고
* RTCPeerConnection API를 사용하여 비디오를 스트리밍하고
* 미디어 캡처 및 스트리밍을 제어하고
* WebRTC 호출을 활성화하여 peer간에 미디어 및 네트워크 정보를 공유하는 방법

에 대해 배웠다. 전체 소스는 **step-02** 디렉토리에 있다.

## 팁
* 이 장에서는 배워야 할 내용이 많았다! RTCPeerConnection에 대해 자세히 설명한 추가자료가 필요하다면 [webrtc.org/start](https://webrtc.org/start/ "webrtc.org/start")를 살펴보자. 이 페이지에는 JavaScript 프레임워크에 대한 제안들이 포함되어 있다(WebRTC를 사용하고 싶지만 API에 대한 부담이 있는 경우를 위해)
* [GitHub 저장소](https://github.com/webrtc/adapter)에서 adapter.js 대해 자세히 알아보자.
* 세계 최고의 비디오 채팅 앱이 어떻게 생겼는지보고 싶다면 WebRTC 프로젝트의 WebRTC 호출용 표준 응용프로그램인 AppRTC([데모](https://appr.tc/), [코드](https://github.com/webrtc/apprtc))를 살펴보자. 통화를 위한 준비시간이 500ms 미만이다.