올려준 HTML을 확인해 보니, 이제 어디를 집중적으로 확인해야 할지 알겠어.

현재 구조는 다음과 같아.

- `/login/login-data`에서 로그인 사용자 정보를 가져옴.
- `login_user`에 사용자 ID를 저장함.
- `XSRF-TOKEN` 쿠키를 읽음.
- `X-XSRF-TOKEN` 헤더에 CSRF 토큰을 넣어 POST 요청을 보냄.

그런데 여기서 중요한 문제가 하나 있어.

`getUserInfo()`를 호출했다고 해서 CSRF 토큰 쿠키가 자동으로 생성되는 것은 아니야.

또한 Spring Security 6에서는 CSRF 토큰을 지연 생성하고, 기본적으로 BREACH 보호를 위한 토큰 처리 방식도 적용할 수 있기 때문에 쿠키 저장소와 요청 헤더 설정을 정확히 맞춰야 해.

차근차근 확인해 보자.

## 1. SecurityConfig를 먼저 수정해 보자

다음과 같이 CSRF 설정을 명시적으로 작성해 줘.

```
import org.springframework.security.web.csrf.CookieCsrfTokenRepository;
import org.springframework.security.web.csrf.CsrfTokenRequestAttributeHandler;
```

그리고 `filterChain()` 안에 다음 설정을 적용해.

```
CsrfTokenRequestAttributeHandler requestHandler =
        new CsrfTokenRequestAttributeHandler();

requestHandler.setCsrfRequestAttributeName(null);

http
    .csrf(csrf -> csrf
        .csrfTokenRepository(
            CookieCsrfTokenRepository.withHttpOnlyFalse()
        )
        .csrfTokenRequestHandler(requestHandler)
        .ignoringRequestMatchers("/login/signup")
    );
```

여기서 중요한 부분은 다음이야.

- `CookieCsrfTokenRepository`: CSRF 토큰을 쿠키로 관리
- `withHttpOnlyFalse()`: JavaScript에서 쿠키 접근 허용
- `CsrfTokenRequestAttributeHandler`: 쿠키에 저장된 원본 토큰과 요청 헤더의 토큰을 동일한 방식으로 검증

기존 `.csrf()` 설정을 위 코드로 교체하면 돼.

## 2. CSRF 토큰을 발급하는 API 추가

쿠키가 실제로 생성되도록 별도의 GET API를 만들자.

기존 `EventBooleanController`에 다음 메서드를 추가해도 돼.

```
import org.springframework.security.web.csrf.CsrfToken;
import org.springframework.web.bind.annotation.GetMapping;

@GetMapping("/csrf-token")
public CsrfToken getCsrfToken(CsrfToken csrfToken) {
    return csrfToken;
}
```

이 API는 CSRF 토큰을 발급하고 JSON으로 반환하는 역할이야.

예상 응답:

```
{
    "headerName": "X-XSRF-TOKEN",
    "parameterName": "_csrf",
    "token": "발급된_토큰"
}
```

이 API는 로그인하지 않은 사용자도 접근할 수 있어야 해. 현재 `.anyRequest().authenticated()`를 사용하고 있으므로 다음 경로를 허용 목록에 추가해 줘.

```
.requestMatchers("/event-boolean/csrf-token").permitAll()
```

주의할 점은 CSRF 토큰 발급 API만 공개하는 것이지, 실제 등록 API를 공개하는 것이 아니라는 거야.

## 3. HTML JavaScript 수정

이제 기존의 `getCookie()` 함수는 필요 없어.

CSRF 토큰 발급 API에서 토큰을 직접 받아오도록 수정하면 돼.

```
async function init() {

    try {
        // 1. CSRF 토큰 발급
        const csrfResponse = await fetch("/event-boolean/csrf-token", {
            method: "GET",
            credentials: "same-origin"
        });

        if (!csrfResponse.ok) {
            throw new Error("CSRF 토큰 발급 실패");
        }

        const csrfData = await csrfResponse.json();

        console.log("CSRF 토큰 발급 완료");

        // 2. 로그인 사용자 확인 및 POST 요청
        const response = await fetch("/event-boolean/registration", {
            method: "POST",
            credentials: "same-origin",
            headers: {
                "Content-Type": "application/json",
                [csrfData.headerName]: csrfData.token
            },
            body: JSON.stringify({
                originalLink: "tripleS"
            })
        });

        const result = await response.text();

        console.log("응답 상태:", response.status);
        console.log("서버 응답:", result);

    } catch (error) {
        console.error("네트워크 에러:", error);
    }
}
```

이제 `getCookie()` 함수 없이도 토큰을 전달할 수 있어.

## 4. 최종 요청 흐름

1. HTML에서 버튼 클릭

   `init()` 실행
2. GET /event-boolean/csrf-token

   CSRF 토큰 발급 및 쿠키 저장
3. POST /event-boolean/registration

   로그인 세션 쿠키와 CSRF 헤더 전달
4. Spring Security 인증

   로그인 여부 및 CSRF 토큰 검증
5. EventBooleanController 실행

   `Principal.getName()`으로 사용자 ID 획득
6. Service → Repository

   데이터베이스 저장

## 5. 테스트할 때 확인할 사항

F12 → Network에서 두 요청을 확인해 줘.

| 요청                                 | 예상 결과 |
| ---------------------------------- | ----- |
| GET `/event-boolean/csrf-token`    | 200   |
| POST `/event-boolean/registration` | 200   |

만약 POST가 여전히 403이라면 다음을 확인해야 해.

- GET 응답에 `Set-Cookie: XSRF-TOKEN`이 있는지
- POST 요청 헤더에 `X-XSRF-TOKEN`이 있는지
- POST 요청에 로그인 세션 쿠키 `JSESSIONID`가 포함되는지

추가로 중요한 점: `formLogin`을 사용하면 비로그인 상태의 POST 요청이 401 대신 로그인 페이지로 리다이렉트될 수도 있어. 따라서 실제 로그인 여부는 서버 응답과 세션 쿠키도 함께 확인해야 해.

내가 추천하는 방식은 지금처럼 순수 HTML + JavaScript + Spring Security 세션 인증 + CSRF 토큰 방식이야.

Thymeleaf를 사용하지 않아도 되고, 로그인 사용자만 등록 API를 이용하도록 유지할 수 있어. 현재 컨트롤러의 `Principal` 사용 방식도 그대로 유지하면 돼.
