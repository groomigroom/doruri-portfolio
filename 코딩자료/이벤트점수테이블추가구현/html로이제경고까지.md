가능해! 지금 구조에서는 Service에서 중복 여부를 확인하고, Controller에서 결과에 따라 메시지를 반환한 다음, HTML JavaScript에서 `alert()`로 출력하면 돼.

현재 Service는 중복 데이터가 있으면 아무 작업도 하지 않고 종료하기 때문에, Controller에서는 성공했는지 실패했는지 알 수 없어. 이 부분을 수정하자.

## 1. Service 수정

기존 `void` 반환형을 `boolean`으로 변경해 줘.

```
public boolean registration(String booleanUserid, String originalLink) {

    LoginUsers loginUser =
            this.loginUserService.getLoginUser(booleanUserid);

    boolean exist = eventBooleanRepository
            .existsByBooleanUseridAndOriginalLink(loginUser, originalLink);

    // 이미 등록된 데이터라면 false 반환
    if (exist) {
        return false;
    }

    EventBooleanEntity eventBooleanEntity = new EventBooleanEntity();
    eventBooleanEntity.setBooleanUserid(loginUser);
    eventBooleanEntity.setOriginalLink(originalLink);

    eventBooleanRepository.save(eventBooleanEntity);

    // 정상 저장되면 true 반환
    return true;
}
```

이제 반환값은 다음과 같아.

| 반환값     | 의미            |
| ------- | ------------- |
| `true`  | 데이터 저장 성공     |
| `false` | 이미 등록된 데이터 존재 |

## 2. Controller 수정

Service에서 반환한 boolean 값을 받아서 응답 메시지를 다르게 보내면 돼.

```
@PostMapping("/registration")
public ResponseEntity<?> postEventBoolean(
        Principal principal,
        @Valid @RequestBody EventBooleanDTO eventBooleanDTO,
        BindingResult bindingResult) {

    if (bindingResult.hasErrors()) {
        Map<String, String> errors = new HashMap<>();

        bindingResult.getFieldErrors().forEach(error ->
                errors.put(error.getField(), error.getDefaultMessage())
        );

        return ResponseEntity.badRequest().body(errors);
    }

    if (principal == null) {
        return ResponseEntity.status(401).body("로그인이 필요합니다.");
    }

    String userId = principal.getName();

    boolean result = eventBooleanService.registration(
            userId,
            eventBooleanDTO.getOriginalLink()
    );

    if (result) {
        return ResponseEntity.ok("등록이 완료되었습니다.");
    } else {
        return ResponseEntity.status(409).body("이미 등록된 항목입니다.");
    }
}
```

여기서 `409 Conflict`는 이미 존재하는 데이터와 충돌한다는 의미야.

## 3. 순수 HTML JavaScript 수정

이제 서버에서 보내주는 응답을 받아서 `alert()`로 출력하면 돼.

```
async function init() {

    try {
        const response = await fetch("/event-boolean/registration", {
            method: "POST",
            credentials: "same-origin",
            headers: {
                "Content-Type": "application/json",
                "X-XSRF-TOKEN": getCookie("XSRF-TOKEN")
            },
            body: JSON.stringify({
                originalLink: "tripleS"
            })
        });

        const result = await response.text();

        if (response.ok) {
            alert(result);
        } else {
            alert(result);
        }

    } catch (error) {
        console.error("네트워크 에러:", error);
        alert("서버와 통신하는 중 오류가 발생했습니다.");
    }
}
```

## 4. 실제 실행 결과

첫 번째 등록 요청

200 OK

등록이 완료되었습니다.

확인

두 번째 동일 데이터 등록 요청

409 Conflict

이미 등록된 항목입니다.

확인

## 5. 추가로 알아두면 좋은 점

현재처럼 단순한 성공/실패 메시지를 전달할 때는 `ResponseEntity`의 HTTP 상태 코드와 응답 본문을 함께 활용하는 것이 좋아.

앞으로 다음과 같은 기능도 동일한 패턴으로 구현할 수 있어.

- 좋아요 중복 방지
- 즐겨찾기 중복 등록 방지
- 댓글 중복 등록 방지
- 이벤트 참여 중복 방지
- 게시물 등록 성공 및 실패 알림

핵심은 Service에서 결과를 반환하고, Controller에서 HTTP 응답을 결정하며, JavaScript에서 사용자에게 보여주는 구조야. 이렇게 역할을 나누면 유지보수하기도 편해져.
