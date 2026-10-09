가능합니다. 지금 구조라면 핵심은 **`from` 파라미터를 Controller에서 받아서 `originalLink`로 조회**하는 것입니다.

 예를 들어:

 - `/sub-writes/create-form?from=akmu`
- `/sub-writes/create-form?from=bts`

 로 들어오면 Controller가 `from` 값을 받아서 각각 `akmu-blooming`, `bts-...` 같은 `originalLink`를 기준으로 댓글을 조회하도록 만들 수 있습니다.

 ### 1. Repository

```
package groomi.groomgroom.SubWrites;

import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;

public interface SubWritesRepository extends JpaRepository<SubWritesEntity, Integer> {

    List<SubWritesEntity> findByOriginalLinkOrderByWriteDateDesc(String originalLink);
}
```

 ### 2\. Controller

 `from` 값과 실제 `originalLink`가 동일하다면 가장 간단합니다.

```
@Controller
@RequestMapping("/sub-writes")
@RequiredArgsConstructor
public class SubWritesController {

    private final SubWritesRepository subWritesRepository;

    @GetMapping("/create-form")
    public String createForm(
            @RequestParam("from") String from,
            Model model
    ) {
        List<SubWritesEntity> subWrites =
                subWritesRepository.findByOriginalLinkOrderByWriteDateDesc(from);

        model.addAttribute("subWrites", subWrites);
        model.addAttribute("originalLink", from);

        return "sub-writes/create-form";
    }
}
```

 그러면 URL을

```
/sub-writes/create-form?from=akmu-blooming
```

 으로 호출했을 때 DB에서

```
originalLink = "akmu-blooming"
```

 인 댓글만 가져옵니다.

---

 ### 3\. 그런데 현재 HTML의 `from=akmu`를 유지하고 싶다면

 질문에서 원하는 구조가

```
<a href="/sub-writes/create-form?from=akmu">
    댓글 목록 보러 가기
</a>
```

 이고 DB에는

```
originalLink = "akmu-blooming"
```

 처럼 저장되어 있다면 **`akmu` → `akmu-blooming` 변환 과정**이 필요합니다.

 예를 들어 Controller에서:

```
@GetMapping("/create-form")
public String createForm(
        @RequestParam("from") String from,
        Model model
) {
    String originalLink = from + "-blooming";

    List<SubWritesEntity> subWrites =
            subWritesRepository.findByOriginalLinkOrderByWriteDateDesc(originalLink);

    model.addAttribute("subWrites", subWrites);
    model.addAttribute("originalLink", originalLink);

    return "sub-writes/create-form";
}
```

 그러면

```
from=akmu
```

 →

```
akmu-blooming
```

 → DB에서 해당 `originalLink`를 가진 댓글 조회

 가 됩니다.

 다만 **`-blooming`을 Controller에 하드코딩하는 것은 추천하지 않습니다.**

---

 ## 4\. 더 좋은 방법: `from` 자체를 originalLink로 전달

 개인적으로는 이 방식이 가장 깔끔합니다.

 `akmu.html`:

```
<a href="/sub-writes/create-form?from=akmu-blooming">
    댓글 목록 보러 가기
</a>
```

 `bts.html`:

```
<a href="/sub-writes/create-form?from=bts">
    댓글 목록 보러 가기
</a>
```

 그리고 Controller:

```
@GetMapping("/create-form")
public String createForm(
        @RequestParam String from,
        Model model
) {
    List<SubWritesEntity> subWrites =
            subWritesRepository.findByOriginalLinkOrderByWriteDateDesc(from);

    model.addAttribute("subWrites", subWrites);
    model.addAttribute("originalLink", from);

    return "sub-writes/create-form";
}
```

 이렇게 하면 `from`의 의미가 사실상 **"어떤 게시물의 댓글인가?"를 나타내는 originalLink**가 됩니다.

---

 ## 5\. 댓글 작성까지 생각한다면

 댓글 작성 폼에서도 `originalLink`를 hidden input으로 넘기면 편합니다.

```
<form action="/sub-writes/create" method="post">

    <input type="hidden"
           name="originalLink"
           th:value="${originalLink}">

    <textarea name="content"></textarea>

    <button type="submit">댓글 작성</button>
</form>
```

 그리고 작성 Controller:

```
@PostMapping("/create")
public String create(
        @RequestParam String originalLink,
        @RequestParam String content,
        @AuthenticationPrincipal LoginUsers user
) {
    SubWritesEntity entity = new SubWritesEntity();

    entity.setContent(content);
    entity.setOriginalLink(originalLink);
    entity.setAuthor(user);
    entity.setWriteDate(LocalDateTime.now());

    subWritesRepository.save(entity);

    return "redirect:/sub-writes/create-form?from=" + originalLink;
}
```

 전체 흐름은 이렇게 됩니다.

```
akmu.html
    ↓
/sub-writes/create-form?from=akmu-blooming
    ↓
Controller
    ↓
findByOriginalLinkOrderByWriteDateDesc("akmu-blooming")
    ↓
댓글 목록
    ↓
댓글 작성
    ↓
originalLink = "akmu-blooming"으로 저장
```

 **중요한 점은 `originalLink`가 실제 URL 자체라면 `String` 하나로 관리하는 것보다, 나중에는 `GroupMainEntity` 같은 부모 게시물 Entity를 만들고 `@ManyToOne`으로 연결하는 구조가 더 좋습니다.** 지금 프로젝트 구조가 `akmu-blooming`, `bts-...`처럼 여러 그룹/게시물로 확장될 예정이라면 그 구조까지 포함해서 Controller + Service + Repository + Thymeleaf를 같이 설계하는 편이 좋습니다.
