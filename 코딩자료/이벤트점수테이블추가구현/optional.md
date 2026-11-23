Spring Boot에서 `findById()` 호출 시 `id is not null` 에러가 발생한다면, 가장 먼저 확인해야 할 부분은 `findById()`에 전달되는 ID 값입니다.

현재 작성하신 코드에서 문제가 될 가능성이 높은 부분은 다음 코드입니다.

java

```java
String userId = principal.getName();
boolean result = eventBooleanService.registration(userId, eventBooleanDTO.getOriginalLink());

if (result) {
    ScoreEntity scoreEntity = this.scoreRepository.findById(userId);
    return ResponseEntity.ok("참여해 주셔서 감사합니다.");
}
```

## 1. findById()의 반환 타입 확인

Spring Data JPA에서 `findById()`는 일반적으로 다음과 같은 형태입니다.

java

```java
Optional<ScoreEntity> findById(ID id);
```

따라서 다음과 같이 사용하는 것이 좋습니다.

java

```java
Optional<ScoreEntity> scoreEntity = scoreRepository.findById(userId);
```

또는 실제 엔티티를 가져오려면 다음과 같이 작성할 수 있습니다.

java

```java
ScoreEntity scoreEntity = scoreRepository.findById(userId)
        .orElseThrow(() -> new RuntimeException("점수 정보를 찾을 수 없습니다."));
```

이때 `ScoreEntity`의 ID 타입과 `userId`의 타입이 일치해야 합니다.

## 2. ScoreEntity의 @Id 확인

`ScoreEntity` 클래스에서 기본키가 어떻게 설정되어 있는지 확인해 보세요.

예를 들어 다음과 같이 되어 있다면,

java

```java
@Entity
public class ScoreEntity {

    @Id
    private String userId;

    // 기타 필드
}
```

`findById(userId)`를 사용할 수 있습니다.

하지만 다음과 같이 되어 있다면,

java

```java
@Entity
public class ScoreEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String userId;
}
```

`findById()`에는 `String userId`가 아니라 `Long id`를 전달해야 합니다.

이 경우 사용자 ID로 검색하려면 Repository에 별도의 메서드를 만들어야 합니다.

java

```java
public interface ScoreRepository extends JpaRepository<ScoreEntity, Long> {

    Optional<ScoreEntity> findByUserId(String userId);
}
```

사용할 때는 다음과 같습니다.

java

```java
ScoreEntity scoreEntity = scoreRepository.findByUserId(userId)
        .orElseThrow(() -> new RuntimeException("사용자 점수 정보가 없습니다."));
```

## 3. null 값이 전달되는지 확인

에러 메시지가 정확히 `The given id must not be null`이라면 `findById()`에 null이 전달되었다는 의미입니다.

다음 코드를 추가해서 확인해 보세요.

java

```java
String userId = principal.getName();

System.out.println("로그인 사용자 ID: " + userId);

ScoreEntity scoreEntity = scoreRepository.findById(userId)
        .orElse(null);
```

만약 `userId`가 null이라면 인증 과정이나 `Principal`에서 사용자 정보를 가져오는 부분을 확인해야 합니다.

## 4. 현재 Controller에서 추천하는 수정 코드

`ScoreEntity`의 기본키가 String 타입의 사용자 ID라는 가정하에 다음과 같이 수정할 수 있습니다.

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

        return ResponseEntity.ok("참여해 주셔서 감사합니다.");

    } else {
        return ResponseEntity.status(409).body("이미 참여하셨습니다.");
    }
}
```

## 정확하게 원인을 찾으려면

다음 두 가지 코드를 추가로 보여주시면 원인을 정확하게 찾아드릴 수 있습니다.

1. `ScoreEntity` 클래스 전체 코드
2. `ScoreRepository` 인터페이스 코드

특히 ScoreEntity의 @Id가 어떤 타입으로 설정되어 있는지가 가장 중요합니다.
