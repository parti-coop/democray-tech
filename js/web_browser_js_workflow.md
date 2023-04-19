# 웹 브라우저에서의 JS 동작 방식

Javascript는 싱글 스레드입니다. 싱글 스레드라서 한 번에 하나의 일 밖에 처리하지 못하는 단점이 있습니다. 하지만 여러 일을 한번에 처리할 수 있는 멀티쓰레드 방식을 사용하면 데드락과 같은 동시성 문제에 대한 처리가 설계에 포함되어야 합니다. 설계가 복잡해지고 언어 구현이 어렵기 때문에 간단한 스크립트 목적에는 부합하지 않아 싱글 스레드를 채택한걸로 추측해봅니다. [Why JavaScript is a single-thread language that can be non-blocking?, geeksforgeeks.org](https://www.geeksforgeeks.org/why-javascript-is-a-single-thread-language-that-can-be-non-blocking/)

JS는 웹 브라우저에서 많이 사용되고 있습니다. JS를 사용하다보면 Ajax 통신, setTimeout 과 같은 Timer 기능과 같은 여러 비동기 기능들을 사용하곤 합니다. 싱글 스레드 방식으로는 비동기처리가 불가능한데 어떻게 가능할 수 있었던 것일까요?

정답은 웹 브라우저에서 비동기처리를 합니다. JS 싱글스레드 단점을 멀티쓰레드가 가능한 웹브라우저에서 보완하고 있습니다. Ajax 통신을 할 때 사용하는 xhr() 과 타이머 기능의 setTimeOut()과 같은 비동기처리 API 들은 모두 웹브라우저의 API인 Web API 입니다. JS 에는 이러한 API가 없습니다.

웹브라우저와 JS 엔진과 협업이 어떻게 이뤄지는지 아래 다이어그램을 보면 알 수 있습니다. 주요 등장인물로는 다음과 같습니다.

![이미지](/assets/img/web_browser_js_workflow_img_1.png)

- Web API: 웹 브라우저의 API. 비동기 기능들 포함.
- Task Queue: Web API에서 처리가 필요한 콜백함수들을 위한 대기열
- Event Loop: Task Queue 에 있는 Task를 JS의 Call Stack 에 등록하는 역할. JS 엔진의 Call Stack을 관찰하다가 비워지면, Task Queue의 Task을 Call Stack에 등록합니다.
- Call Stack: JS 엔진 스레드가 실행하는 콜스택.

동작 방식을 알기위해 하나의 예를 들어 보겠습니다. 비동기통신을 위해 xhr() 함수를 실행시키면 Web API 에서 네트워크 통신을 시작합니다. 이 처리는 웹브라우저의 멀티쓰레드 중 한 쓰레드가 담당합니다. JS 스레드는 비동기함수 말고 콜스택에 쌓인 함수를 처리합니다. Web API 의 xhr 통신이 끝나면 xhr 에 등록한 콜백함수를 Task Queue에 등록합니다. xhr 콜백함수는 큐에서 대기합니다. Event Loop 에서 Call Stack 이 드디어 비워졌다는 걸 알고 큐에 있던 xhr 콜백함수를 Call Stack 에 등록합니다. 그리고 JS 엔진의 스레드가 콜스택에 쌓인 xhr 콜백함수를 처리합니다. [어쨌든 이벤트 루프는 무엇입니까? | Philip Roberts | JSConf EU](http://latentflip.com/loupe/?code=JC5vbignYnV0dG9uJywgJ2NsaWNrJywgZnVuY3Rpb24gb25DbGljaygpIHsKICAgIHNldFRpbWVvdXQoZnVuY3Rpb24gdGltZXIoKSB7CiAgICAgICAgY29uc29sZS5sb2coJ1lvdSBjbGlja2VkIHRoZSBidXR0b24hJyk7ICAgIAogICAgfSwgMjAwMCk7Cn0pOwoKY29uc29sZS5sb2coIkhpISIpOwoKc2V0VGltZW91dChmdW5jdGlvbiB0aW1lb3V0KCkgewogICAgY29uc29sZS5sb2coIkNsaWNrIHRoZSBidXR0b24hIik7Cn0sIDUwMDApOwoKY29uc29sZS5sb2coIldlbGNvbWUgdG8gbG91cGUuIik7!!!PGJ1dHRvbj5DbGljayBtZSE8L2J1dHRvbj4%3D)

이런 원리를 알고 있다면, 처리시간이 오래 걸리는 특정 로직을 setTimeout 콜백으로 해두고 시간 0으로 설정해서 비동기처리 되도록하면 다른 로직들을 먼저 처리하고 맨 나중에 처리시간이 오래걸리는 콜백 로직을 실행할 수 있습니다.
