---
layout: single
title: "검증은 누가 들고 있어야 하는가"
date: 2026-05-14 23:00:00 +0900
categories: [design]
tags: [TDD, DDD, VO, OOP]
toc: true
toc_sticky: true
toc_label: "목차"
---

# 검증은 누가 들고 있어야 하는가

> **TL;DR**: 책임을 *기능*이 아니라 *바뀔 이유*로 보면, 코드의 자리는 다시 정해진다.

회원가입, 내 정보 조회, 비밀번호 변경 기능을 구현하기 위한 필드 다섯 개짜리 회원 모델로 며칠을 썼다.
코드가 어려워서가 아니라, 한 질문에 계속 걸렸기 때문이다.

> **검증은 누가 들고 있어야 하는가.**

`User`의 `loginId`, `password`, `email`, `name`, `birthDate` 다섯 개 필드는 각자 조건을 가지고 있다.

- 비밀번호는 8~16자의 영문 대소문자·숫자·특수문자만 허용
- 비밀번호에 생년월일이 포함되면 안 됨
- 이메일은 형식이 있음
- `loginId`도 자기 규칙이 있음

그럼 이 검증을 어디서 하고, 누가 책임지는가.

**단순한 기능이 단순한 답을 주지 않았다.**

TDD로 시작했으니 유저 엔티티를 만들기 전에 테스트부터 짰다.
그런데 막상 써보니, 유저 생성 테스트인데 유저는 거의 안 보이고 필드를 검증하는 테스트만 쌓였다.

**배보다 배꼽이 더 큰 셈이었다.**

---

## 회원가입 — 검증은 어디에 있어야 하는가

### 책임을 무엇으로 볼 것인가

유저 생성에서 검증해야 하는 케이스는 다음과 같다.

1. 비밀번호는 8~16자의 영문 대소문자·숫자·특수문자만 가능하다.
2. 비밀번호에 생년월일이 포함되면 안 된다.
3. 이름, 이메일, `loginId`는 각 포맷이 맞아야 한다.

한편으로는, *유저 생성이라는 책임 안에 필드 조건 검사가 들어가는 게 당연하지 않나?* 라는 생각도 들었다.
그래서 다시 고민해봤다.

> **책임을 *기능*으로 보아야 할까, 아니면 *바뀔 이유*로 보아야 할까.**

책임을 *바뀔 이유*로 보니 방향을 결정할 수 있었다.
유저 객체가 비밀번호 정책을 직접 검증하면, 비밀번호 정책이 바뀔 때마다 유저 객체가 수정되어야 한다.
이메일 형식이 바뀌어도 유저가 바뀐다.

**유저가 서로 다른 이유로 자꾸 변경의 여지를 갖게 되는 거다.**

그러니까 각 필드를 VO로 떼어내는 진짜 이유는 *재사용*이나 *책임의 개수*가 아니라 **변경의 축을 분리하는 것**이었다.

이렇게 되니 유저는 생성할 때 이미 검증이 끝난 값만 파라미터로 받게 된다.
유저 객체에는 "생성한다"는 책임 하나만 남는다.

```java
// Before — User가 모든 규칙을 안다
public class User {
    public User(String loginId, String password, ...) {
        if (!password.matches("^[A-Za-z0-9...]{8,16}$")) throw ...;
        if (password.contains(birthDate)) throw ...;
        if (!email.matches(...)) throw ...;
        // 비밀번호 정책 바뀌면 User가 바뀐다
        // 이메일 형식 바뀌면 User가 바뀐다
    }
}

// After — User는 검증된 값만 받는다
public class User {
    public User(LoginId loginId, Password password, Email email, ...) {
        // User에는 "조립한다"는 책임만 남는다
    }
}
```

### VO를 어디까지 활용할 것인가

또 한 번 의문이 들었다.

> 유저는 유효한 값만 받으면 되니, 유저 서비스의 회원가입 메서드에서 각 필드 조건을 검사하고 유저 객체를 생성해도 되는 거 아닌가?  
> 굳이 VO를 만들어서 클래스 수를 늘릴 필요가 있을까?

이 의문은 "도메인 규칙이 어디에 머무르는가"라는 질문으로 다시 바라보니 풀렸다.
검증 책임을 서비스에 두면 회원 생성의 도메인 규칙이 서비스 계층에 종속된다.
그러면 다른 어딘가에서 유저를 생성할 때 같은 검증을 또 반복해야 한다.

**도메인 규칙이 *도메인 바깥*에 머무르는 상태인 거다.**

그래서 유저 생성 시 이미 검증이 끝난 값만 넘어오게 하려고 VO를 썼다.

그런데 VO를 단순히 "검증 책임을 옮기는 자리"로만 쓰고 끝낼 일은 아니라고 생각했다.
`Password` 객체는 8~16자에 생년월일이 안 들어간 상태로만 **존재할 수 있다**.
생성 시점에 그 조건이 통과되지 않으면 객체 자체가 만들어지지 않는다.
이게 **자가 검증**이 하는 일이고, 단순한 만큼 효과는 분명하다.

> 시스템 안쪽 어디서든 `Password` 타입의 값을 받았다면, 그 값이 유효한지 다시 의심할 필요가 없다.

의심하지 않아도 되는 만큼 코드는 단순해진다.

- 유저 객체는 자기 필드의 유효성을 확인하지 않는다
- 회원가입 서비스도 검증을 다시 돌리지 않는다
- 다른 어디서 `Password`를 받든 같은 검증 로직을 또 쓰지 않는다

검증 책임을 **값 안쪽**으로 모으니, 도메인 규칙은 그 값을 만드는 자리 한 곳에서만 일어났다.
그 자리만 통과하면 시스템 안쪽 어디서든 유효함을 전제할 수 있다.
`Password`를 받은 코드는 다시 의심하지 않고, 유저 객체도 자기 필드를 다시 확인하지 않는다.

---

다만 `name` 차례에서는 한참 망설였다.
`name`은 빈 값과 공백을 막고, 한글과 영문 외의 문자가 들어오지 못하게 하는 정도.
다른 필드들에 비해 규칙이 가벼웠다.
이 정도로 VO라고 부를 수 있는 건지, 그냥 `String`으로 둬도 되는 게 아닌지가 걸렸다.

결국 만들기로 했다.

> **VO는 규칙의 *무게*로 결정할 게 아니라, 그 규칙이 그 값 말고 다른 데 있을 이유가 없는가로 결정할 일이었다.**

`name`만 알고 있는 규칙이 있는 이상, 그걸 다른 데로 흘려보낼 이유는 없었다.

---

## 비밀번호 변경 — 검증과 상태 변경은 누가 들고 있을 것인가

회원가입에서 책임의 위치를 정하는 기준은 **변경의 축**이었다.
비밀번호 변경을 구현할 때도 비슷한 결의 고민이 따라왔다.

> 상태를 바꾸는 일과 그 앞에 붙는 검증이 한 흐름 안에 있을 때, 그 흐름 자체는 누가 들고 있어야 하는가.

회원가입까지는 비교적 단순했다.
유효한 VO가 들어오면 조립해서 `User`를 만든다. `signUp` 메서드는 이 결로 짜여 있었다.

```java
@Transactional
public void changePassword(Long userId, String currentRawPassword, Password newPassword) {
    UserModel user = userRepository.findById(userId)...;
    if (!passwordEncoder.matches(currentRawPassword, user.getPassword().getValue())) {
        throw new CoreException(ErrorType.UNAUTHORIZED);
    }
    if (currentRawPassword.equals(newPassword.getValue())) {
        throw new CoreException(ErrorType.BAD_REQUEST, "현재 비밀번호와 동일한 비밀번호로 변경할 수 없습니다.");
    }
    newPassword.requireNotContaining(user.getBirthDate());
    Password encodedNewPassword = Password.encoded(passwordEncoder.encode(newPassword.getValue()));
    user.changePassword(encodedNewPassword);
}
```

비밀번호 변경은 결이 조금 달랐다.
**상태를 바꾸는 일**과 **그 앞에 붙는 여러 검증**이 한 흐름 안에 있었기 때문이다.
이 일을 누가 들고 있을 것인가를 두고 한 번 더 멈췄다.

### 첫 번째 안 — User에게 책임을 위임

먼저 떠오른 건 `User`에게 책임을 위임하는 안이었다.
서비스는 사용자를 찾아오기만 하고, 검증과 교체는 `User`가 자기 안에서 처리한다.

```java
@Transactional
public void changePassword(Long userId, String currentRawPassword, Password newPassword) {
    UserModel user = userRepository.findById(userId)...;
    user.changePassword(currentRawPassword, newPassword, passwordEncoder);
}

// UserModel
public void changePassword(String currentRawPassword, Password newPassword, PasswordEncoder passwordEncoder) {
    // 비밀번호 검증 — 현재 == 요청 현재, 현재 != 새, 새 비밀번호에 생년월일 포함 X
    // 비밀번호 교체
}
```

![user-responsibility-diagram.png](/assets/images/user-responsibility-diagram.png)  
"첫 번째 안. User가 점점 무거워진다 — Encoder까지 알아야 한다."

이렇게 두면 서비스는 "사용자를 찾고 있으면 비밀번호를 변경한다"는 한 줄이 된다.
깔끔해 보였다.

### User가 PasswordEncoder를 알아야 하나

그런데 메서드 시그니처에 다시 고민을 하기 시작했다.
`UserModel.changePassword`의 세 번째 파라미터, `PasswordEncoder`다.

처음엔 "협력자를 받는 것"의 관점으로 봤다.
`User`가 자기 일을 하기 위해 필요한 협력자를 메서드 파라미터로 받는다. (자연스러운 모양이라고 생각했다.)

그런데 `PasswordEncoder`가 *어떤 협력자인지*를 다시 보니 생각이 변했다.

> `PasswordEncoder`는 도메인 레이어에 인터페이스로 두었지만, 그 구현체는 인프라에 있다 (스프링 시큐리티의 `BCryptPasswordEncoder`).
> 즉 이건 **기술적 능력의 추상화**였다.

도메인 안에 인터페이스를 둔 건 DIP 때문에 그렇게 한 것일 뿐, *해싱*이라는 행위 자체는 인프라의 일이다.
`User`가 이걸 파라미터로 받는다는 건, **결국 인프라 추상화를 끌어안는 모양**이라고 생각했다.

이는 테스트 코드를 작성하면서 `User`의 도메인 순수성이 깨진다고 한번 더 느꼈다.
`User`의 단위 테스트를 작성하려고 보니 mock encoder가 필요했다.
도메인 모델의 단위 테스트는 순수하게 외부 의존 없이 일어나야 한다고 봤는데, **mock이 필요하다는 사실 자체가 "모델이 더 이상 순수하지 않다"는 신호였다.**

리팩터링 하라는 소리로 들렸다.

### 서비스가 조율

지금 코드는 다시 **서비스가 검증을 들고, `User`는 *교체*만 책임지는 형태**로 정착했다.

```java
@Transactional
public void changePassword(Long userId, String currentRawPassword, Password newPassword) {
    UserModel user = userRepository.findById(userId)...;
    if (!passwordEncoder.matches(currentRawPassword, user.getPassword().getValue())) {
        throw new CoreException(ErrorType.UNAUTHORIZED);
    }
    if (currentRawPassword.equals(newPassword.getValue())) {
        throw new CoreException(ErrorType.BAD_REQUEST, "현재 비밀번호와 동일한 비밀번호로 변경할 수 없습니다.");
    }
    newPassword.requireNotContaining(user.getBirthDate());
    Password encodedNewPassword = Password.encoded(passwordEncoder.encode(newPassword.getValue()));
    user.changePassword(encodedNewPassword);
}
```

![service-orchestration-diagram.png](/assets/images/service-orchestration-diagram.png)  
"두 번째 안. Service가 셋을 조율하고, User는 다시 가벼워진다."

여기서 서비스가 하는 일을 다시 보면 — **`User` 하나에 자연스럽게 안 붙는 도메인 로직을 조율하는 자리**다.

비밀번호 변경이라는 흐름에 등장하는 협력자가 셋이다.

- **Repository** (사용자 조회) — Aggregate 외부의 일
- **Aggregate** (상태 교체) — `User` 자기 상태
- **Domain Port** (비밀번호 해싱) — 인프라 추상화

> 세 협력자가 등장하는 워크플로우는 누구의 일도 아니다.
> **조율의 일이고, 그 자리가 도메인 서비스다.**

각 책임이 자기 자리에 있게 된 셈이다.

### 호출 안전성

이 형태에도 단점은 있다.

> 누군가 서비스의 검증을 거치지 않고 `user.changePassword`를 직접 호출하면?
> **비밀번호의 유효성이 깨진다.**

이 점이 생각보다 크리티컬해서 한참 고민했다.

이 보장을 코드 차원에서 강제하려면 다시 첫 번째 안으로 돌아가야 한다.
그러면 `User`가 *"나의 정체성을 어떻게 증명하는가"*까지 알게 된다.  
그건 `User`의 일이 아니다.

그래서 **호출 진입점을 좁히는 쪽**으로 갔다.

`UserModel.changePassword`는 `package-private`으로 선언했다.
같은 도메인 패키지 안의 서비스에서만 호출 가능하다.

> 즉 이 메서드를 "비밀번호 변경 유스케이스의 진입점"으로 두기로 했다.

다른 곳에서 `User`의 비밀번호를 바꾸는 일이 생긴다면, 그건 **새로운 유스케이스**가 추가된 것이고,
그때 그 새 유스케이스 서비스가 자기 시나리오에 맞는 검증을 갖추면 된다.

물론 이 보장은 컴파일러가 강제하는 보장은 아니다.
같은 패키지 안에서는 누구든 직접 호출할 수 있으니까.

**하지만 *아키텍처 레이어가 강제하는 보장*은 된다.**

도메인 모델을 직접 호출하는 자리는 서비스로 정해져 있고, 서비스는 각자의 유스케이스 시나리오를 가진다.
이 **레이어 규약**이 호출 안전성을 책임진다고 봤다.

완벽한 답은 아니지만, **받아들일 수 있는 trade-off**라고 봤다.

돌아보면 회원가입에서도, 비밀번호 변경에서도 결국 같은 질문을 두 번 고민했다.  
>"어떤 책임이 어느 자리에 있어야 하는가"  

코드의 자리는, 그 코드가 바뀌어야 하는 이유에 따라 정해진다.
그걸 한 번씩 다시 묻는 일이 결국 설계라는 걸, 단순한 기능을 짜면서 다시 배웠다.
