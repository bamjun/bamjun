가능합니다. 정확히는 **Vercel 셀프호스팅이라기보다, 내 Vercel 계정에 전용 인스턴스를 배포하는 방식**입니다.

이렇게 하면 공용 `github-profile-summary-cards.vercel.app`의 공유 토큰·공유 사용량 대신 **내 Vercel 배포와 내 GitHub 토큰**을 사용하므로 지금 본 공용 API의 rate limit 문제를 크게 줄일 수 있습니다. 다만 이 프로젝트는 Vercel 배포에서도 `GITHUB_TOKEN`이 반드시 필요합니다. ([GitHub][1])

## 전체 구조

```text
GitHub README
  ↓ 이미지 요청
내 Vercel 프로젝트
  ↓ GitHub API 조회
GitHub PAT
  ↓
SVG 카드 반환
```

## 1. 원본 저장소 Fork

GitHub에서 다음 저장소를 엽니다.

```text
vn7n24fzkq/github-profile-summary-cards
```

오른쪽 위 `Fork`를 눌러 내 계정으로 복사합니다.

예상 결과:

```text
bamjun/github-profile-summary-cards
```

이 저장소는 이미 Vercel Function과 `vercel.json`을 포함하고 있으며, 현재 `package.json`은 Node.js 24를 지정합니다. 따라서 별도의 서버 코드를 만들 필요는 없습니다. ([GitHub][2])

---

## 2. 내 계정만 조회하도록 고정하기

원본 상태로 배포하면 누구든 다음처럼 접근할 수 있습니다.

```text
/api/cards/stats?username=다른사람
```

그러면 다른 사람이 내 GitHub 토큰의 API 할당량을 사용할 수 있습니다. 현재 공통 API 핸들러는 요청에서 `username`을 직접 받아 카드 생성 함수에 전달합니다. ([GitHub][3])

Fork한 저장소에서 다음 파일을 엽니다.

```text
api/utils/handle-card.ts
```

다음 부분을 찾습니다.

```ts
const {username, theme: rawTheme = 'default'} = req.query;

if (typeof rawTheme !== 'string') {
  res.status(400).send('theme must be a string');
  return;
}

if (typeof username !== 'string') {
  res.status(400).send('username must be a string');
  return;
}
```

위 코드를 아래 코드로 교체합니다.

```ts
const {
  username: requestedUsername,
  theme: rawTheme = 'default',
} = req.query;

if (typeof rawTheme !== 'string') {
  res.status(400).send('theme must be a string');
  return;
}

const fixedUsername = process.env.PROFILE_USERNAME?.trim();

if (!fixedUsername) {
  res.status(500).send('PROFILE_USERNAME is not configured');
  return;
}

// username을 전달했다면 고정된 계정과 같은지 검사
if (
  requestedUsername !== undefined &&
  (
    typeof requestedUsername !== 'string' ||
    requestedUsername.toLowerCase() !== fixedUsername.toLowerCase()
  )
) {
  res.status(403).send(
    'This deployment only supports the configured GitHub profile',
  );
  return;
}

// 실제 카드 생성에는 무조건 고정된 사용자명 사용
const username = fixedUsername;
```

커밋 메시지는 다음 정도로 하면 됩니다.

```text
feat: restrict cards to bamjun profile
```

이제 URL에서 `username`을 생략해도 되고, 다른 아이디를 넣으면 `403`이 반환됩니다.

---

## 3. GitHub PAT 만들기

Vercel 서버가 GitHub 통계를 조회하려면 PAT가 필요합니다. 공식 프로젝트는 다음 두 형태를 지원합니다. ([GitHub][1])

### 갱신이 귀찮다면 Classic PAT

GitHub에서 다음으로 이동합니다.

```text
Settings
→ Developer settings
→ Personal access tokens
→ Tokens (classic)
→ Generate new token (classic)
```

설정:

```text
Note:
github-profile-summary-cards-vercel

Expiration:
No expiration

Scopes:
✓ public_repo
✓ read:user
```

공식 프로젝트에서도 공개 프로필 조회용 Classic PAT 권한으로 `public_repo`와 `read:user`를 안내합니다. ([GitHub][1])

다만 Classic PAT는 Fine-grained PAT보다 권한 범위가 넓습니다. 반드시 Vercel 환경변수에만 저장하고, 코드나 `.env` 파일을 GitHub에 올리면 안 됩니다.

---

## 4. Vercel에 배포하기

Vercel Dashboard에서:

```text
Add New
→ Project
→ Import Git Repository
```

방금 Fork한 저장소를 선택합니다.

```text
bamjun/github-profile-summary-cards
```

프로젝트 설정은 다음처럼 둡니다.

```text
Framework Preset:
Other

Root Directory:
./

Build Command:
기본값 유지

Output Directory:
기본값 유지

Install Command:
기본값 유지
```

프로젝트의 `vercel.json`이 출력 디렉터리, API rewrite, Function 메모리와 실행시간을 이미 정의하고 있으므로 일반적으로 별도 빌드 설정은 필요 없습니다. ([GitHub][2])

### Environment Variables

배포 화면의 `Environment Variables`에 두 개를 추가합니다.

```text
GITHUB_TOKEN
```

값:

```text
방금 만든 GitHub Classic PAT
```

그리고:

```text
PROFILE_USERNAME
```

값:

```text
bamjun
```

환경은 전부 체크합니다.

```text
✓ Production
✓ Preview
✓ Development
```

공식 프로젝트 역시 Vercel에서 `GITHUB_TOKEN`을 환경변수로 등록하고 배포하도록 안내합니다. 환경변수를 나중에 변경했다면 기존 배포에 자동 반영되지 않으므로 반드시 새로 배포해야 합니다. ([GitHub][1])

이제 `Deploy`를 누릅니다.

---

## 5. 배포 주소 확인

배포가 끝나면 다음과 비슷한 주소가 나옵니다.

```text
https://bamjun-profile-summary-cards.vercel.app
```

실제 발급된 주소는 Vercel Dashboard에서 확인해야 합니다.

브라우저에서 먼저 다음 URL들을 테스트합니다.

```text
https://내주소.vercel.app/api/cards/profile-details?theme=aura
```

```text
https://내주소.vercel.app/api/cards/repos-per-language?theme=aura
```

```text
https://내주소.vercel.app/api/cards/most-commit-language?theme=aura
```

```text
https://내주소.vercel.app/api/cards/stats?theme=aura
```

```text
https://내주소.vercel.app/api/cards/productive-time?theme=aura&utcOffset=9
```

`PROFILE_USERNAME=bamjun`으로 고정했기 때문에 이제 다음 파라미터는 없어도 됩니다.

```text
username=bamjun
```

다른 사용자를 넣으면 차단되는지도 확인합니다.

```text
https://내주소.vercel.app/api/cards/stats?username=octocat&theme=aura
```

예상 결과:

```text
403
This deployment only supports the configured GitHub profile
```

---

## 6. 최종 README 코드

`YOUR_VERCEL_DOMAIN`을 실제 Vercel 주소로 바꾸세요.

```html
<div align="center">
  <a href="https://git.io/typing-svg">
    <img
      src="https://readme-typing-svg.demolab.com?font=Honk&size=35&pause=1000&random=false&width=435&lines=HI%2C+there.+I'm+bamjun.+%F0%9F%91%8B"
      alt="Typing SVG"
    />
  </a>
</div>

<br />

<div align="center">
  <img
    src="https://YOUR_VERCEL_DOMAIN/api/cards/profile-details?theme=aura"
    alt="GitHub profile details"
  />
</div>

<div align="center">
  <img
    src="https://YOUR_VERCEL_DOMAIN/api/cards/repos-per-language?theme=aura"
    alt="Repositories per language"
  />
  <img
    src="https://YOUR_VERCEL_DOMAIN/api/cards/most-commit-language?theme=aura"
    alt="Most commit language"
  />
</div>

<div align="center">
  <img
    src="https://YOUR_VERCEL_DOMAIN/api/cards/stats?theme=aura"
    alt="GitHub stats"
  />
  <img
    src="https://YOUR_VERCEL_DOMAIN/api/cards/productive-time?theme=aura&utcOffset=9"
    alt="Productive time"
  />
</div>
```

예를 들어 발급 주소가 다음이라면:

```text
bamjun-profile-summary-cards.vercel.app
```

첫 번째 이미지는 이렇게 됩니다.

```html
<img
  src="https://bamjun-profile-summary-cards.vercel.app/api/cards/profile-details?theme=aura"
  alt="GitHub profile details"
/>
```

공식 API는 `profile-details`, `repos-per-language`, `most-commit-language`, `stats`, `productive-time` 엔드포인트와 테마·UTC 오프셋 옵션을 지원합니다. ([GitHub][1])

## 7. 기존 GitHub Action 정리

Vercel URL을 README에서 직접 사용할 거라면 다음 Workflow는 더 이상 필요하지 않습니다.

```text
.github/workflows/profile-summary-cards.yml
```

다음 정적 SVG 폴더도 필요 없으면 삭제할 수 있습니다.

```text
profile-summary-card-output/
```

다만 **Vercel 카드 5개가 README에서 모두 정상 출력되는 것을 확인한 뒤** 삭제하는 게 안전합니다.

## 최종 선택

가장 편한 구성은 이겁니다.

```text
내 Vercel 배포
+ PROFILE_USERNAME=bamjun
+ Classic PAT / No expiration
+ public_repo
+ read:user
+ README에서 내 Vercel URL 직접 사용
```

이렇게 구성하면 GitHub Action으로 SVG를 다운로드하고 커밋할 필요가 없고, 공용 서버의 공유 rate limit에도 덜 휘말립니다. 다만 내 GitHub PAT 자체의 API 한도는 존재하므로, Vercel 주소를 `bamjun` 전용으로 고정하는 수정은 꼭 적용하는 게 좋습니다.

---

**Refined Prompt Suggestion**

> `github-profile-summary-cards`를 내 GitHub 계정 `bamjun` 전용으로 Vercel에 배포하고 싶어. 다른 사용자가 username 파라미터를 바꿔 내 GitHub PAT 할당량을 사용하지 못하게 차단하고, Classic PAT 무기한 설정, Vercel 환경변수, 수정할 TypeScript 코드, 배포 테스트 URL, 최종 README HTML까지 복사 가능한 형태로 작성해줘.

[1]: https://github.com/vn7n24fzkq/github-profile-summary-cards "GitHub - vn7n24fzkq/github-profile-summary-cards: A tool to generate your GitHub summary card for profile README · GitHub"
[2]: https://github.com/vn7n24fzkq/github-profile-summary-cards/blob/main/vercel.json "github-profile-summary-cards/vercel.json at main · vn7n24fzkq/github-profile-summary-cards · GitHub"
[3]: https://github.com/vn7n24fzkq/github-profile-summary-cards/blob/main/api/utils/handle-card.ts "github-profile-summary-cards/api/utils/handle-card.ts at main · vn7n24fzkq/github-profile-summary-cards · GitHub"
