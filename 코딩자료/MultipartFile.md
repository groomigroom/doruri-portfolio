kimgugui
꺼로 업로드함

---=-=-=-=

지금 코드를 전체적으로 보니까 원인이 명확해졌어. 👍\
 **`fileName`이 DB에 저장되지 않는 가장 큰 이유는 HTML의 파일 업로드 방식이 잘못되어 있기 때문이야.**

 현재 구조에서는 크게 **2가지가 잘못되어 있어.**

---

 # 1\. `<form>`에 `enctype="multipart/form-data"`가 없음

 현재:

```
<form th:action="@{/login/signup}"
      th:object="${loginUserCreateForm}"
      method="post">
```

 파일을 업로드하려면 반드시:

```
enctype="multipart/form-data"
```

 가 있어야 해.

 따라서:

```
<form th:action="@{/login/signup}"
      th:object="${loginUserCreateForm}"
      method="post"
      enctype="multipart/form-data">
```

 로 바꿔야 해.

 이게 없으면 브라우저가 파일 자체를 multipart 데이터로 전송하지 않아.

---

 # 2\. `fileName`을 `String`으로 받고 있는데 `<input type="file">`에 `th:field`를 사용하고 있음

 현재 DTO:

```
private String fileName;
```

 그리고 HTML:

```
<input type="file" th:field="*{fileName}">
```

 이렇게 되어 있지?

 여기가 중요한 문제야.

 `fileName`은 **String**이고, 실제 업로드되는 파일은 **MultipartFile**이야.

 즉,

```
fileName
    ↓
String
```

 과

```
실제 업로드 파일
    ↓
MultipartFile
```

 은 서로 다른 거야.

 네 Service도 이미 이렇게 만들어 놨잖아.

```
public LoginUsers create(
        String userid,
        String username,
        String password,
        MultipartFile imgFile)
```

 즉 네가 설계한 구조에서는 **파일을 Controller에서 `MultipartFile`로 받고 → Service에 전달 → Service에서 파일명을 만들어 DB에 저장**하는 방식이야.

 그렇다면 `LoginUserCreateForm`의 `fileName`은 사실 필요 없어.

---

 # 3\. 지금 구조에 맞게 수정하면

 ## `LoginUserCreateForm`

 이렇게:

```
@Getter
@Setter
public class LoginUserCreateForm {

    @Size(max = 50)
    @NotEmpty(message = "사용자ID는 필수항목입니다.")
    private String userid;

    @NotEmpty(message = "이름은 필수항목입니다.")
    private String username;

    @NotEmpty(message = "비밀번호는 필수항목입니다.")
    private String password1;

    @NotEmpty(message = "비밀번호 확인은 필수항목입니다.")
    private String password2;
}
```

 **`fileName` 삭제해도 돼.**

 왜냐하면 파일 이름은 사용자가 직접 입력하는 값이 아니라 Service에서 생성하기 때문이야.

---

 # 4\. HTML을 이렇게 변경

 현재:

```
<input type="file" th:field="*{fileName}">
```

 이걸:

```
<input type="file" name="imgFile">
```

 로 변경해.

 그리고 form에는 반드시:

```
enctype="multipart/form-data"
```

 를 추가.

 전체적으로:

```
<form th:action="@{/login/signup}"
      th:object="${loginUserCreateForm}"
      method="post"
      enctype="multipart/form-data">

    <div class="mb-3">
        <label for="userid" class="form-label">사용자ID</label>
        <input type="text" th:field="*{userid}" class="form-control">
    </div>

    <div class="mb-3">
        <label for="username" class="form-label">이메일</label>
        <input type="text" th:field="*{username}" class="form-control">
    </div>

    <div class="mb-3">
        <label for="password1" class="form-label">비밀번호</label>
        <input type="password" th:field="*{password1}" class="form-control">
    </div>

    <div class="mb-3">
        <label for="password2" class="form-label">비밀번호 확인</label>
        <input type="password" th:field="*{password2}" class="form-control">
    </div>

    <div class="mb-3">
        <label for="imgFile" class="form-label">프로필 이미지</label>
        <input type="file" name="imgFile" id="imgFile" class="form-control">
    </div>

    <button type="submit" class="btn btn-primary">회원가입</button>
</form>
```

 여기서 핵심은:

```
name="imgFile"
```

 이야.

 Controller의:

```
MultipartFile imgFile
```

 과 이름을 맞춰주는 거야.

---

 # 5\. Controller도 조금 명확하게 해주는 걸 추천

 현재:

```
@PostMapping("/signup")
public String loginSignup(
        @Valid LoginUserCreateForm loginUserCreateForm,
        BindingResult bindingResult,
        MultipartFile imgFile) {
```

 이것도 동작할 수 있지만, 명시적으로:

```
@PostMapping("/signup")
public String loginSignup(
        @Valid LoginUserCreateForm loginUserCreateForm,
        BindingResult bindingResult,
        @RequestParam(value = "imgFile", required = false) MultipartFile imgFile) {
```

 라고 하는 걸 추천해.

 그러면:

```
name="imgFile"
```

 ↓

```
@RequestParam("imgFile")
```

 ↓

```
MultipartFile imgFile
```

 이라는 관계가 명확해져.

---

 # 6\. Service는 지금 거의 잘 되어 있음

 네 Service:

```
public LoginUsers create(
        String userid,
        String username,
        String password,
        MultipartFile imgFile) throws Exception {

    String imgFilePath =
            System.getProperty("user.dir")
            + "\\src\\main\\resources\\static\\LoginImgs\\";

    File uploadDir = new File(imgFilePath);

    if (!uploadDir.exists()) {
        uploadDir.mkdirs();
    }

    String fileName = "";

    if (imgFile != null && !imgFile.isEmpty()) {

        UUID uuid = UUID.randomUUID();

        fileName = uuid + "_" + imgFile.getOriginalFilename();

        File saveImgFile = new File(imgFilePath, fileName);

        imgFile.transferTo(saveImgFile);
    }

    LoginUsers loginUser = new LoginUsers();

    loginUser.setUserid(userid);
    loginUser.setUsername(username);
    loginUser.setPassword(passwordEncoder.encode(password));
    loginUser.setFileName(fileName);

    this.loginUserRepository.save(loginUser);

    return loginUser;
}
```

 이 부분은 **현재 목적에 맞게 잘 되어 있어.**

 파일이 정상적으로 전달되면:

```
fileName = uuid + "_" + imgFile.getOriginalFilename();
```

 여기서 파일명이 만들어지고,

```
loginUser.setFileName(fileName);
```

 여기서 Entity에 들어가고,

```
loginUserRepository.save(loginUser);
```

 여기서 DB에 저장돼.

---

 # 7\. 현재 문제가 발생하는 흐름

 지금은 사실 이런 상태야.

```
HTML
  ↓
<input type="file" th:field="*{fileName}">
  ↓
fileName은 String
  ↓
MultipartFile imgFile로 제대로 전달되지 않음
  ↓
Service
  ↓
imgFile == null 또는 empty
  ↓
if (imgFile != null && !imgFile.isEmpty())
    실행 안 됨
  ↓
String fileName = "";
  ↓
loginUser.setFileName("");
  ↓
DB에 빈 문자열 저장
```

 그래서 `fileName`이 저장되지 않는 것처럼 보이는 거야.

---

 # 최종적으로 이렇게 생각하면 쉬워

 ### DTO

 사용자가 입력하는 회원정보:

```
LoginUserCreateForm
 ├── userid
 ├── username
 ├── password1
 └── password2
```

 ### MultipartFile

 업로드하는 파일:

```
imgFile
   ↓
MultipartFile
```

 ### Entity

 DB에 저장할 정보:

```
LoginUsers
 ├── userid
 ├── username
 ├── password
 └── fileName
```

 ### 전체 흐름

```
[signup_form.html]

userid ──────────────┐
username ────────────┤
password1 ───────────┤
password2 ───────────┤
                     ↓
          LoginUserCreateForm

imgFile ─────────────→ MultipartFile
                           ↓
                    Controller
                           ↓
                 LoginUserService
                           ↓
              UUID + 실제파일명 생성
                           ↓
                     fileName
                           ↓
                      LoginUsers
                           ↓
                        DB 저장
```

---

 ## 그래서 딱 3개만 고치면 돼

 ### ① form

```
<form th:action="@{/login/signup}"
      th:object="${loginUserCreateForm}"
      method="post"
      enctype="multipart/form-data">
```

 ### ② file input

```
<input type="file" name="imgFile">
```

 ### ③ Controller

```
@PostMapping("/signup")
public String loginSignup(
        @Valid LoginUserCreateForm loginUserCreateForm,
        BindingResult bindingResult,
        @RequestParam(value = "imgFile", required = false) MultipartFile imgFile) {
```

 그리고 `LoginUserCreateForm`에서는:

```
private String fileName;
```

 을 제거하면 돼.

---

 **추가로 하나 더:** 지금 `password` DB 컬럼이 `@Column(nullable = false)`인데 길이 지정이 없어서 JPA가 보통 충분한 길이로 잡지만, `@Column(length = 50)`로 되어 있는 실제 DB 테이블이라면 `BCrypt` 같은 인코딩된 비밀번호는 **50자를 넘을 수 있어.** 그러면 나중에 회원가입할 때 `password` 길이 때문에 DB 오류가 날 수 있으니 비밀번호 컬럼은 최소 `VARCHAR(100)` 정도로 잡는 걸 권장해.

 지금 파일 업로드 문제는 **`multipart/form-data` \+ `name="imgFile"`** 이 두 가지가 핵심이야.


--------------------------------------------------------

아, 이건 **앞에서 발생한 `fileName` 타입 문제와는 다른 문제**야.

 지금 화면의

 > **HTTP ERROR 413**

 은 보통 **업로드하려는 요청의 크기가 서버에서 허용한 크기보다 커서 거부된 것**이야.

 즉 지금은 오히려 **파일 업로드 요청 자체가 서버까지 제대로 들어가고 있다는 단계**까지 간 거야. 👍

 ### 1\. 가장 먼저 `application.properties` 확인

 `src/main/resources/application.properties`에 다음을 추가해봐.

```
# 파일 하나의 최대 크기
spring.servlet.multipart.max-file-size=10MB

# 전체 요청의 최대 크기
spring.servlet.multipart.max-request-size=10MB
```

 예를 들어 이미지가 15MB 정도라면:

```
spring.servlet.multipart.max-file-size=20MB
spring.servlet.multipart.max-request-size=20MB
```

 정도로 설정하면 돼.

 그리고 **Spring Boot를 완전히 재실행**해.

---

 ### 2\. 지금 HTML은 이런 형태인지 확인

```
<form th:action="@{/login/signup}"
      th:object="${loginUserCreateForm}"
      method="post"
      enctype="multipart/form-data">

    <div class="mb-3">
        <label for="userid" class="form-label">사용자ID</label>
        <input type="text" th:field="*{userid}" class="form-control">
    </div>

    <div class="mb-3">
        <label for="username" class="form-label">이메일</label>
        <input type="text" th:field="*{username}" class="form-control">
    </div>

    <div class="mb-3">
        <label for="password1" class="form-label">비밀번호</label>
        <input type="password" th:field="*{password1}" class="form-control">
    </div>

    <div class="mb-3">
        <label for="password2" class="form-label">비밀번호 확인</label>
        <input type="password" th:field="*{password2}" class="form-control">
    </div>

    <div class="mb-3">
        <label for="imgFile" class="form-label">프로필 이미지</label>
        <input type="file" name="imgFile" id="imgFile" class="form-control">
    </div>

    <button type="submit" class="btn btn-primary">회원가입</button>
</form>
```

 특히 이 부분:

```
enctype="multipart/form-data"
```

 가 중요하고,

```
<input type="file" name="imgFile">
```

 도 중요해.

---

 ### 3\. Controller도 이렇게

```
@PostMapping("/signup")
public String loginSignup(
        @Valid LoginUserCreateForm loginUserCreateForm,
        BindingResult bindingResult,
        @RequestParam(value = "imgFile", required = false) MultipartFile imgFile) {

    if (bindingResult.hasErrors()) {
        return "Login/signup_form";
    }

    if (!loginUserCreateForm.getPassword1()
            .equals(loginUserCreateForm.getPassword2())) {

        bindingResult.rejectValue(
                "password2",
                "passwordInCorrect",
                "2개의 패스워드가 일치하지 않습니다."
        );

        return "Login/signup_form";
    }

    try {
        loginUserService.create(
                loginUserCreateForm.getUserid(),
                loginUserCreateForm.getUsername(),
                loginUserCreateForm.getPassword1(),
                imgFile
        );

    } catch (DataIntegrityViolationException e) {
        e.printStackTrace();
        bindingResult.reject(
                "signupFailed",
                "이미 등록된 사용자입니다."
        );
        return "Login/signup_form";

    } catch (Exception e) {
        e.printStackTrace();
        bindingResult.reject(
                "signupFailed",
                e.getMessage()
        );
        return "Login/signup_form";
    }

    return "redirect:/";
}
```

---

 ## 4\. 그런데 중요한 포인트가 하나 있어

 **파일을 선택하지 않아도 413이 뜨는지 확인해봐.**

 ### 테스트 A

 파일을 아예 선택하지 않고:

```
회원가입 버튼
     ↓
정상적으로 회원가입
```

 되는지 확인.

 ### 테스트 B

 아주 작은 이미지 하나를 선택:

```
100KB 정도 JPG
     ↓
회원가입
```

 ### 테스트 C

 큰 이미지를 선택:

```
10MB 이상 이미지
     ↓
413
```

 이렇게 된다면 거의 확실하게 **파일 크기 제한 문제**야.

---

 ## 5\. `application.yml`을 쓰고 있다면

 `application.properties`가 아니라 `application.yml`을 쓰고 있다면:

```
spring:
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 10MB
```

 이렇게 해야 해.

 **둘 중 하나만** 사용하면 돼.

---

 ## 6\. 그리고 네 경우에는 `fileName`을 DTO에서 빼는 게 맞아

 현재 구조라면:

```
public class LoginUserCreateForm {
    private String userid;
    private String username;
    private String password1;
    private String password2;
}
```

 그리고 파일은 별도로:

```
MultipartFile imgFile
```

 로 받는 구조가 가장 깔끔해.

 Service에서:

```
String fileName = uuid + "_" + imgFile.getOriginalFilename();
```

 만들고,

```
loginUser.setFileName(fileName);
```

 으로 DB에 넣는 거지.

---

 ### 결론

 지금 **413은 코드의 `th:field` 문제라기보다는 업로드 용량 제한 문제**로 보는 게 맞아.

 우선 `application.properties`에 이것부터 넣어봐:

```
spring.servlet.multipart.max-file-size=10MB
spring.servlet.multipart.max-request-size=10MB
```

 그리고 **Spring Boot 재실행 → 작은 이미지로 다시 회원가입**해봐.

 만약 **파일 크기가 몇 KB밖에 안 되는데도 413이 뜬다면**, 그때는 `application.properties` 설정 문제가 아니라 다른 제한이 있는 것이므로 **Spring Boot 콘솔에 찍힌 에러 로그**를 보여줘. 그 로그를 보면 정확히 어느 레벨에서 413을 발생시키는지 바로 알 수 있어.
