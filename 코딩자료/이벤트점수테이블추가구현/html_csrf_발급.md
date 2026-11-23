가능해! 오히려 네가 원하는 구조라면 순수 HTML + JavaScript의 fetch 방식은 그대로 유지하고, Spring Security에서 로그인 인증과 CSRF 토큰을 함께 처리하는 방식이 가장 적절해.

핵심은 다음 세 가지야.

1. 로그인한 사용자만 POST 요청 가능
2. HTML과 JavaScript는 Thymeleaf 없이 독립적으로 유지
3. Spring Security의 CSRF 보안도 유지

현재 구조를 최대한 유지하면서 수정해 보자.

## 1. SecurityConfig 수정

우선 기존 설정에서 다음 부분은 그대로 유지해.

```
.anyRequest().authenticated()
```

이 설정 덕분에 `/event-boolean/registration`은 로그인한 사용자만 접근할 수 있어.

그리고 CSRF 설정은 수정해야 해.

기존 코드:

```
.csrf((csrf) -> csrf
    .ignoringRequestMatchers("/login/signup"))
```

이 코드는 그대로 유지해. `/event-boolean/registration`을 CSRF 예외로 추가하지 않는 거야.

즉, 로그인 인증과 CSRF 검증을 모두 적용하는 거지.

## 2. 순수 HTML에서 CSRF 토큰 발급받기

여기서 중요한 부분이 있어.

Thymeleaf를 사용하지 않으므로 HTML에서 `th:content`를 사용할 수 없어.

대신 Spring Security가 제공하는 CSRF 토큰을 별도의 API로 받아오면 돼.

Spring Security 설정에 다음 코드를 추가해 보자.

```
.csrf(csrf -> csrf
    .csrfTokenRepository(
        CookieCsrfTokenRepository.withHttpOnlyFalse()
    )
    .ignoringRequestMatchers("/login/signup")
)
```

필요한 import:

```
import org.springframework.security.web.csrf.CookieCsrfTokenRepository;
```

이렇게 하면 서버가 `XSRF-TOKEN`이라는 쿠키에 CSRF 토큰을 저장해 줘.

`withHttpOnlyFalse()`는 JavaScript가 해당 쿠키를 읽을 수 있도록 하는 설정이야.

## 3. JavaScript에서 CSRF 토큰 읽기

이제 네가 작성한 `test.html`에서 다음과 같이 작성하면 돼.

```
// 쿠키에서 CSRF 토큰 가져오기
function getCookie(name) {
    const value = `; ${document.cookie}`;
    const parts = value.split(`; ${name}=`);

    if (parts.length === 2) {
        return parts.pop().split(';').shift();
    }

    return null;
}

async function init() {

    const csrfToken = getCookie("XSRF-TOKEN");

    try {
        const response = await fetch(
            "/event-boolean/registration",
            {
                method: "POST",
                credentials: "same-origin",
                headers: {
                    "Content-Type": "application/json",
                    "X-XSRF-TOKEN": csrfToken
                },
                body: JSON.stringify({
                    originalLink: "tripleS"
                })
            }
        );

        const result = await response.text();

        console.log("응답 상태:", response.status);
        console.log("서버 응답:", result);

    } catch (error) {
        console.error("네트워크 에러:", error);
    }
}
```

## 4. 전체 동작 구조

① 순수 HTML + JavaScript

fetch()로 POST 요청

② Spring Security

로그인 세션 확인 + CSRF 토큰 검증

③ EventBooleanController

Principal에서 로그인 사용자 ID 획득

④ EventBooleanService → Repository

데이터베이스 저장

## 5. 주의할 점: HTML과 서버의 주소가 다른 경우

현재 네 요청 주소는 다음과 같아.

```
http://112.76.56.127/event-boolean/registration
```

만약 `test.html`도 같은 주소와 포트에서 실행된다면 위 코드를 그대로 적용할 수 있어.

하지만 HTML이 다른 도메인이나 포트에서 실행된다면 다음 설정도 필요해.

- CORS 허용 설정
- `credentials: "include"`
- 쿠키의 SameSite 및 Secure 설정
- CSRF 쿠키를 읽을 수 있는 구조

## 6. 마지막으로 한 가지 확인하자

지금 네 `test.html`은 어디에서 실행하고 있어?

test.html 실행 위치

Spring Boot 서버와 같은 주소 (112.76.56.127)

내 컴퓨터에서 직접 실행 (파일 더블클릭, file://)

다른 웹서버 또는 다른 도메인

답변하기

이걸 확인해야 쿠키와 CSRF 토큰이 제대로 전달되도록 정확하게 설정할 수 있어. 특히 로그인 세션을 유지하면서 순수 JavaScript로 인증된 POST 요청을 보내는 것이 이번 목표야.
