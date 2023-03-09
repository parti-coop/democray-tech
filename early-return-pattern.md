# early-return-pattern

함수를 작성할 때 조건이 붙으면 if/else 패턴을 사용해야하는 경우가 흔히 있습니다.
if/else를 중첩하다보면 코드의 depth가 자동으로 생겨나고 유지보수를 힘들게 할 수 있는 이유가되기도 하는데요.
이때 조건에 따라 빠르게 return을 사용함으로써 가독성과 코드 흐름을 빠르게 볼 수 있는 효과가 있는 패턴입니다.

예를들어 어떤 이벤트를 클릭했을 때 token과 id가 있다면 저장하는 함수를 만든다고 가정해보겠습니다.

짧은 코드지만 if의 depth가 3번이나 생기기 때문에 혼잡하다는 느낌이 듭니다.

```javascript
// 중첩된 if
function handleClick(event) {
  if (event.target.matches('.save-data')) {
    let id = event.target.getAttribute('data-id');
    if (id) {
      let token = localStorage.getItem('token');
      if (token) {
        localStorage.setItem(`${token}_${id}`, true);
      }
    }
  }
}
```

빠른 리턴을 사용하면 코드의 흐름을 자연스럽게 읽을 수 있습니다.

```javascript
// 빠른 return
function handleClick(event) {
  if (!event.target.matches('.save-data')) return;

  let id = event.target.getAttribute('data-id');
  if (!id) return;

  let token = localStorage.getItem('token');
  if (!token) return;

  localStorage.setItem(`${token}_${id}`, true);
}
```

하지만, 단점도 있는데요.
Single Entry, Single Exit (SESE)의 개념처럼
많은 양의 코드가 작성되어야 하는 함수일 경우 너무 많은 return은 오히려 독이 될수도 있다고합니다.
정답은 없기 때문에 코드리뷰시의 동료들과 합리적인 선에서의 if/else의 depth를 합의하고 컨트롤 하는 것이 필수일 것 같습니다.
