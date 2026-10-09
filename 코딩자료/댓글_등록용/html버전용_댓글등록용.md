네. 지금 코드의

```
return "redirect:/song-main/" + SongMainStringId + "#subwrite_" + savedSubWrite.getId();
```

 는 **Thymeleaf HTML 파일을 직접 지정하는 방식이 아니라, 브라우저에게 `/song-main/{id}` URL로 다시 요청하라고 하는 방식**입니다.

 만약 `SongMainStringId`에 따라 **서로 다른 HTML 파일로 직접 redirect**하고 싶다면 방법이 몇 가지 있습니다.

 ## 1\. ID에 따라 URL을 다르게 redirect하는 방법 — 추천

 예를 들어 HTML 구조가 다음과 같다고 하겠습니다.

```
templates/
├── SongMain/
│   ├── song1.html
│   ├── song2.html
│   └── song3.html
```

 ID가 `song1`, `song2`, `song3`이라면:

```
return "redirect:/SongMain/" + SongMainStringId + ".html";
```

 그리고 실제 URL을 받을 Controller가 필요합니다.

```
@GetMapping("/SongMain/{SongMainStringId}.html")
public String songMainPage(
        @PathVariable("SongMainStringId") String SongMainStringId,
        Model model) {

    SongMainEntity songMainEntity =
            this.songMainService.getSongMainEntity(SongMainStringId);

    model.addAttribute("songMainEntity", songMainEntity);

    return "SongMain/" + SongMainStringId;
}
```

 그러면:

```
SongMainStringId = song1
```

 일 때

```
redirect:/SongMain/song1.html
```

 → Controller가

```
return "SongMain/song1";
```

 →

```
templates/SongMain/song1.html
```

 을 렌더링합니다.

---

 ## 2\. 그런데 현재 구조라면 이 방법이 더 좋습니다

 사실 **ID마다 HTML 파일을 따로 만드는 것보다는**, 지금처럼 하나의 Thymeleaf HTML에서 ID에 맞는 데이터를 보여주는 방식을 추천합니다.

 예를 들어:

```
templates/
└── SongMain/
    └── SongMain.html
```

 Controller:

```
@GetMapping("/song-main/{SongMainStringId}")
public String songMain(
        @PathVariable("SongMainStringId") String SongMainStringId,
        Model model) {

    SongMainEntity songMainEntity =
            this.songMainService.getSongMainEntity(SongMainStringId);

    model.addAttribute("songMainEntity", songMainEntity);

    return "SongMain/SongMain";
}
```

 그러면 URL은

```
/song-main/abc
/song-main/def
/song-main/xyz
```

 처럼 각각 달라도 실제 HTML은 하나입니다.

 Thymeleaf가:

```
<h1 th:text="${songMainEntity.songMainTitle}"></h1>
```

 처럼 해당 ID의 데이터를 출력합니다.

 그리고 댓글 등록 후에는 지금 코드 그대로:

```
return "redirect:/song-main/"
        + SongMainStringId
        + "#subwrite_"
        + savedSubWrite.getId();
```

 하면 됩니다.

 즉,

```
댓글 등록
   ↓
POST /create/song-main/abc
   ↓
댓글 DB 저장
   ↓
redirect:/song-main/abc#subwrite_15
   ↓
GET /song-main/abc
   ↓
SongMain/SongMain.html
   ↓
#subwrite_15 위치로 이동
```

 이 구조가 가장 일반적인 Spring MVC 방식입니다.

---

 ## 3\. "Thymeleaf가 아닌 각각의 HTML 파일"을 원한다면

 만약 정말로 다음처럼 **HTML 파일 자체가 각각 완전히 다른 페이지**라면:

```
templates/
├── song/
│   ├── 100.html
│   ├── 101.html
│   └── 102.html
```

 Spring Controller에서 다음처럼 처리할 수 있습니다.

```
@GetMapping("/song/{id}")
public String song(@PathVariable String id) {

    return "song/" + id;
}
```

 단, 이 경우에도 `return "song/" + id;` 자체는 **Thymeleaf View 이름**입니다.

 즉:

```
return "song/100";
```

 은

```
templates/song/100.html
```

 을 렌더링합니다.

 ### HTML을 정말 정적 파일로 제공하고 싶다면

 `src/main/resources/static`에 파일을 둡니다.

```
src/main/resources/
└── static/
    └── song/
        ├── 100.html
        ├── 101.html
        └── 102.html
```

 그러면 브라우저에서:

```
/song/100.html
```

 로 접근할 수 있습니다.

 Controller에서:

```
return "redirect:/song/" + SongMainStringId + ".html";
```

 처럼 사용할 수도 있습니다.

 다만 이 방식에서는 **Thymeleaf 처리가 되지 않습니다.**

---

 ### 현재 코드에서 가장 중요한 차이

 현재:

```
return "redirect:/song-main/" + SongMainStringId
        + "#subwrite_" + savedSubWrite.getId();
```

 이것은 **URL로 redirect**입니다.

 반면:

```
return "SongMain/SongMain";
```

 은 **Thymeleaf 템플릿을 렌더링**하는 것입니다.

 그리고:

```
return "redirect:/song/" + SongMainStringId + ".html";
```

 은 **정적 HTML URL로 redirect**하는 것입니다.

 따라서 질문하신 의도가 **`SongMainStringId`마다 `100.html`, `101.html`처럼 실제 별도 HTML 파일로 이동**하는 것이라면 `static` 폴더를 사용하는 구조가 적합합니다.

 원하시면 현재 `createSubWritesSongMain()` 코드 기준으로 **`templates` 방식 / `static HTML` 방식 각각 전체 Controller + 폴더 구조 + 댓글 앵커(`#subwrite_id`)까지 작동하는 형태**로 바꿔드릴 수 있습니다.
