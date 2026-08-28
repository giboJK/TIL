
> 2026-08-28 · AWS 처음 만져보면서 기획서 공개 파이프라인을 만든 기록.
> 목표는 하나였음: **로컬에서 고치고 커밋·푸시하면, 끝. 보는 사람은 https 짧은 주소로 항상 최신을 봄.**

## 오늘의 결과물

```
로컬에서 기획서 수정
  → git push (MVP 브랜치)
    → pre-push 훅이 변경 감지
      → S3 sync (바뀐 파일만)
        → CloudFront 캐시 무효화
          → 1~2분 뒤 https://dxxxx.cloudfront.net/ 반영
```

비용은 사실상 0원임. 아래에 왜 0원인지 나옴.

## 1. Amplify를 버린 이유 — 클라우드 과금은 "누가 일하느냐"의 문제

처음엔 Amplify(깃 연결하면 push마다 자동 배포해주는 AWS 서비스)를 쓰려고 했음.
그런데 Amplify는 **연결한 브랜치에 push될 때마다 빌드**가 돌고, 빌드가 **분당 $0.01 과금**임.

우리 문서는 바닐라 JS 정적 파일이라 빌드할 게 없음. 파일 복사가 전부임.
즉 "빌드" 자체가 우리한테는 불필요한 일임.

그래서 아래처럼 간단하게 역할을 쪼갬:

| 역할 | 담당 | 비용 |
|---|---|---|
| 파일 보관 | S3 | 몇 MB → 월 1원 미만 |
| 전송 + HTTPS | CloudFront | 월 1TB 무료 티어 |
| 배포 실행 | 내 맥 (CLI 스크립트) | 0원 |

오늘 얻은 관점: **AWS가 일하면 돈이고, 내 컴퓨터가 일하면 공짜임. 어차피 내가 기획서 쓰고 저장하니까 저장한 김에 배포까지 해버리자!인거임**
배포를 AWS(Amplify 빌드)에 시키는 대신 내 맥(스크립트)에 시키니 과금 주체가 사라짐.

## 2. IAM 생성하고 configuration 설정

- 루트 계정으로 CLI 쓰는 거 아니고, **배포 전용 IAM 사용자**를 만듦.
- 정책(권한) 두 개 연결: `AmazonS3FullAccess` + `CloudFrontFullAccess`.
- **액세스 키** 발급 = ID + Secret 한 쌍. **Secret은 발급 화면에서 딱 한 번만 보여줌.** 놓치면 재발급.
- 터미널 등록은 `aws configure` — 키 2개, region, output 물어봄.
- region 입력할 때 "아시아 태평양(서울)"이 아니라 **`ap-northeast-2`**라고 씀.
- 연결 확인은 이거:

```bash
aws sts get-caller-identity
```

"지금 내가 누구로 접속해 있는가"를 알려줌. Arn에 내 IAM 유저명 뜨면 성공.

## 3. 배포 스크립트

스크립트 뼈대는 "버킷 존재 보장(확인 후 없으면 생성) → 버킷 설정 보장(공개 정책·웹 호스팅을 매번 확인) → 동기화(로컬의 최신파일을 업로드) → 캐시 갱신 → 링크 출력"

```bash
set -euo pipefail                      # 하나라도 실패하면 즉시 중단

aws s3api head-bucket ...              # 버킷 있나? 확인만 함 (없으면 아래 create-bucket)
aws s3api create-bucket ...            # 없을 때만 실행
aws s3api put-bucket-policy ...        # 공개 읽기 정책 — 매번 다시 적용
aws s3 website ... --index-document index.html

aws s3 sync <로컬> s3://<버킷>/desk --delete --exclude ...
```

배운 것들:

- **멱등**: 같은 설정을 두 번 적용해도 결과가 같게 짜면, 몇 번을 돌려도 안전함.
  "버킷 없으면 만들고, 설정은 매번 다시 맞춘다" 패턴.
- **`sync` vs `cp`**: sync는 바뀐 파일만 올림. 두 번째 실행부터 몇 초면 끝남.

CloudFront 쪽이 재밌었는데, 배포 ID를 어디 적어두는 게 아니라 **스크립트가 매번 찾아냄**:

```bash
DIST_ID=$(aws cloudfront list-distributions \
  --query "DistributionList.Items[?Origins.Items[?contains(DomainName, '$BUCKET')]].Id | [0]" \
  --output text 2>/dev/null || true)
```

"원본 도메인에 내 버킷 이름이 들어간 배포"를 JMESPath 쿼리로 검색.
끝의 `2>/dev/null || true`가 포인트 — CloudFront가 없거나 권한이 없어도 죽지 않고 건너뜀.
찾았으면 캐시 비우기:

```bash
aws cloudfront create-invalidation --distribution-id "$DIST_ID" --paths "/*"
```

`/*` 통째로 비워도 **경로 1건**으로 계산돼서 월 1,000건 무료 안에 들어감.

## 4. git pre-push 훅 — "푸시가 곧 배포"

원하는 그림이 "수정 → 푸시 → 끝"이었는데, 이걸 훅으로 해결함.
pre-push(push 직전에 실행되는) 스크립트고, git이 표준입력으로
`로컬ref 로컬sha 원격ref 원격sha` 줄들을 넘겨줌. 조건 두 개를 검사함:

1. **브랜치 조건** — 푸시 대상이 작업 브랜치(refs/heads/MVP)인가
2. **경로 조건** — 푸시 범위에 문서 경로 변경이 있는가

```bash
git diff --name-only "$remote_sha" "$local_sha" |
  grep -qE '^Docs/(product/{제품명}/plan/|resources/fonts/|index\.html$)'
```

기획서 본문, 공용 폰트, 그리고 문서 목록 대문(`Docs/index.html`) 셋 중 하나라도 바뀌면 배포.
둘 다 맞을 때만 배포 스크립트 실행. 설계 원칙 세 개를 정했음:

- **배포가 실패해도 push는 막지 않는다.** (공개본만 잠깐 옛날 걸로 남을 뿐)
- **aws CLI나 자격 증명이 없는 머신에선 조용히 통과.** (훅 파일은 레포에 커밋되니까)
- **새 브랜치 첫 푸시는 비교 대상이 없으니 무조건 1회 배포.**

그리고 함정 하나 — **`.githooks/`에 파일만 두면 git은 안 봄.** 레포에 훅을 커밋해서 쓰려면 경로를 알려줘야 함:
(.githooks/ 폴더는 프로젝트 root에 위치.)

```bash
git config core.hooksPath .githooks
```

(`.git/hooks/`에 두면 이 설정이 필요 없지만, 거긴 git이 추적을 안 해서 팀 공유가 안 됨.)

### 설계를 한 번 뒤집었음

처음엔 배포 전용 브랜치(productplan)를 파서 "여기 병합하면 배포" 규칙으로 갔음.
그땐 **훅이 "어떤 파일이 바뀌었는지"까지 볼 수 있다는 걸 몰라서**, 공개 시점을 브랜치로 통제하려 한 거임.
그런데 훅 코드에 경로 필터를 넣을 수 있었음
브랜치를 지우고 "MVP + 경로 필터"로 단순화함.

## 5. 최종 비용 정산

| 항목 | 사용량 | 요금 |
|---|---|---|
| S3 저장 | 몇 MB | ~0원 |
| CloudFront 전송 | 월 1TB까지 | 무료 티어 |
| 캐시 무효화 | 월 1,000건까지 | 무료 |
| 배포 실행 | 내 맥 | 0원 |
| Route 53 도메인 | 안 삼 (cloudfront.net 기본 도메인) | 0원 |

무료 티어는 **계정 유형에 따라 조건이 다름.** 예전 계정은 CloudFront 1TB가 상시 무료였는데,
2025년 7월 이후 만든 새 계정은 크레딧 기반 무료 플랜으로 구조가 바뀌었음.
어느 쪽이든 이 규모에서 실제 청구는 0에 수렴하지만, 걱정되면 **Budget 알림($1)** 걸어두면 끝.

## 오늘 배운 것 요약

- 배포 스크립트는 멱등하게 — 몇 번 돌려도 같은 결과가 나오게.
- git pre-push 훅으로 "푸시 = 배포"를 만들 수 있음. 단, 훅은 push를 절대 막지 않게 짤 것.
- URL 상대경로는 루트 위로 못 올라감(클램핑). 문서를 `/desk/` 같은 **깊이 1 폴더**에 두면
  `../../../resources/fonts`가 `/resources/fonts`로 수렴해서, 문서마다 폰트를 복사할 필요가 없음.(즉 앞에 ../ 이 부분들이 무시가 되고 싸그리 사라짐)