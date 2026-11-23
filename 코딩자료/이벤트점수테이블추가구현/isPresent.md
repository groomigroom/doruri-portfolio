현재 `setScore()`가 안 되는 이유는 `findById()`의 반환 타입이 `ScoreEntity`가 아니라 `Optional<ScoreEntity>`이기 때문입니다.

즉, `Optional` 안에 들어 있는 실제 `ScoreEntity` 객체를 꺼내야 `setScore()`를 사용할 수 있습니다.

## 1. 가장 간단한 해결 방법

현재 코드의 다음 부분을 수정하세요.

기존 코드:

java

```java
Optional<ScoreEntity> scoreEntity = this.scoreRepository.findById(userId);
scoreEntity.setScore
```

수정 코드:

java

```java
Optional<ScoreEntity> scoreEntity = scoreRepository.findById(userId);

if (scoreEntity.isPresent()) {
    ScoreEntity score = scoreEntity.get();
    score.setScore(score.getScore() + 10);
}
```

이렇게 하면 기존 점수에 10점을 추가할 수 있습니다.

## 2. Controller 전체 수정 예시

java

```java
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
        return ResponseEntity.status(401).body("로그인 필요합니다.");
    }

    String userId = principal.getName();

    boolean result = eventBooleanService.registration(
            userId,
            eventBooleanDTO.getOriginalLink()
    );

    if (result) {

        Optional<ScoreEntity> scoreEntity =
                scoreRepository.findById(userId);

        if (scoreEntity.isPresent()) {
            ScoreEntity score = scoreEntity.get();

            // 기존 점수에 10점 추가
            score.setScore(score.getScore() + 10);
        }

        return ResponseEntity.ok("참여해 주셔서 감사합니다.");

    } else {
        return ResponseEntity.status(409).body("이미 참여하셨습니다.");
    }
}
```

## 3. 그런데 한 가지 중요한 점이 있습니다\!

현재 `ScoreEntity`는 다음과 같이 선언되어 있습니다.

java

```java
@OneToOne
@MapsId
@JoinColumn(name = "userid")
private LoginUsers loginUsers;
```

`@MapsId`를 사용하고 있으므로 `ScoreEntity`의 기본키인 `userid`는 `LoginUsers`의 기본키와 연결됩니다.

따라서 다음 두 가지를 확인해야 합니다.

- `principal.getName()`이 실제 `LoginUsers`의 기본키 값과 일치하는가?
- 해당 사용자의 `ScoreEntity`가 데이터베이스에 실제로 존재하는가?

또한 JPA의 트랜잭션 안에서 조회한 엔티티의 값을 변경하면 변경 감지(Dirty Checking)를 통해 데이터베이스에 반영됩니다.

따라서 점수 변경 작업은 가급적 Service에서 `@Transactional`을 사용해 처리하는 것을 추천합니다.

개인적으로는 점수 변경 로직을 Controller가 아닌 Service로 옮기는 것을 추천합니다. Controller는 요청과 응답을 담당하고, 점수 계산과 저장은 Service가 담당하도록 분리하는 것이 좋습니다.
