# Supabase 로컬 운영 계획 (동시 2개 반 대응)

## 1. 배경 및 결정 사항

- Supabase 무료 플랜은 **계정 전체 기준으로 활성 프로젝트 2개**까지만 허용된다 (조직을 여러 개 만들어도 총합은 동일, 일시정지된 프로젝트는 한도에서 제외).
- 현재 개인용 포트폴리오 프로젝트로 이미 1개 슬롯을 사용 중이라, 반 2개를 동시에 클라우드 프로젝트로 운영하기엔 슬롯이 1개 부족하다.
- 검토한 대안과 결론:
  | 방식 | 문제점 | 채택 여부 |
  |---|---|---|
  | 반마다 클라우드 프로젝트 생성 | 무료 슬롯 부족 | ❌ |
  | 대시보드에서 수동 일시정지/재개 토글 | 재개 1~3분 소요(늦으면 수업 중 대기), 개인 포트폴리오가 그 시간 동안 다운됨, 매번 수동 조작 필요 | ❌ |
  | Supabase Pro 업그레이드 | 매달 비용 발생 | 보류 (필요시 재검토) |
  | **로컬 Supabase(Docker + Supabase CLI)** | 반마다 Docker 리소스 필요 | ✅ **채택** |

- **결론: 반별 데모/실습용 DB는 클라우드가 아니라 각 반 프로젝트 폴더 안에서 로컬 Supabase(Docker)로 운영한다.** 클라우드 무료 슬롯 2개(개인용 1개 + 여유 1개)는 건드리지 않는다.
- 학생들은 기존 방침대로 각자 본인 명의 Supabase 계정을 사용하므로 이 계획과 무관하다. 이 문서는 **강사 데모/실습 환경**에 대한 것이다.

## 2. 적용 범위 및 폴더 원칙

- `my-ai-web` 폴더 전체가 "한 반"의 강의 자료 단위다. 두 번째 반은 이 폴더 내부가 아니라 **완전히 별도의 최상위 프로젝트 폴더**(예: 다른 위치의 `my-ai-web` 사본)로 존재한다.
- 따라서 **이 폴더 안에 class-a / class-b 같은 하위 폴더를 새로 만들지 않는다.** 로컬 Supabase 설정은 각 반의 최상위 폴더(`my-ai-web` 자체, 또는 그 사본) 안에 1세트씩 둔다.
- 같은 컴퓨터에서 두 반을 동시에 켤 것이므로, **포트가 겹치지 않도록 반별로 다른 포트를 배정**하는 것이 핵심이다.

## 3. 사전 준비물 (최초 1회)

- [ ] Docker Desktop 설치 및 실행 확인 (Windows는 WSL2 백엔드 권장)
- [ ] Supabase CLI 설치 확인 (`supabase --version`)
- [ ] 반별 포트 배정표 확정 (아래 4번 참고)
- [ ] Docker Desktop에 할당된 메모리 확인 — 스택 2개를 동시에 켤 것이므로 여유 있게 설정 (권장: Docker Desktop 설정에서 최소 6~8GB 이상)

## 4. 반별 포트 배정표

`supabase init` 실행 후 생성되는 `supabase/config.toml`에는 `[api] port`, `[db] port`, `[studio] port`, `[inbucket] port`, `[db] shadow_port` 등 여러 포트 항목이 있다. **반별로 이 값 전체를 겹치지 않게 바꿔야 한다.** 가장 쉬운 방법은 반마다 고정 오프셋을 더하는 것이다.

| 반 | 오프셋 예시 | 비고 |
|---|---|---|
| A반 (1반) | 기본값 그대로 사용 | `config.toml` 기본 포트 유지 |
| B반 (2반) | 기본값 + 10000 | 예: api 54321 → 64321 방식으로 config.toml의 **모든** 포트 항목에 동일하게 적용 |
| C반 (향후 확장 시) | 기본값 + 20000 | 반이 늘어나면 이 표에 행만 추가 |

- `project_id`도 반마다 다르게 지정한다 (예: `class-a`, `class-b`) — Docker 컨테이너/볼륨 이름이 자동으로 분리되어 데이터가 섞이지 않는다.
- 실제 포트 번호는 `supabase init` 직후 생성된 `config.toml`을 열어 확인하고, 이 표에 반별 최종 확정값을 채워 넣는다.

## 5. 반별 최초 셋업 절차 (반 개설 시 1회)

1. 해당 반의 최상위 프로젝트 폴더에서 `supabase init` 실행 → `supabase/` 폴더 생성.
2. `supabase/config.toml`에서
   - `project_id`를 반 이름으로 변경
   - 포트 항목 전체를 4번 배정표에 맞게 수정
3. `supabase start`로 최초 기동 확인.
4. `supabase status`로 출력되는 API URL / anon key를 확인해 데모용 React 앱의 `.env` (`VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY`)에 반영.
5. 필요한 테이블(`projects`, `guestbook` 등) 스키마와 시드 데이터를 준비한다. **아래 두 단계로 진행하며, MCP가 아니라 "파일 작성 + CLI 명령 실행" 방식이다.**
   > **⚠️ Supabase MCP로는 할 수 없다.** 이 프로젝트에 연결된 Supabase MCP(`mcp.supabase.com`)는 OAuth로 로그인한 **클라우드 계정의 프로젝트**에만 접근하는 원격 서버라, 내 컴퓨터의 `localhost` 로컬 Docker 스택에는 애초에 닿을 수 없다. 또한 MCP로 뭔가 하려면 결국 클라우드 프로젝트 2개 중 하나를 써야 하므로, 클라우드 슬롯을 아끼려는 이 계획의 목적과도 맞지 않는다.
   1. **파일 작성**: `supabase migration new <이름>` 으로 빈 SQL 파일을 만들고, 그 안에 `CREATE TABLE` 등 원하는 스키마 SQL을 직접 작성(또는 Claude Code에게 요청)한다. 초기 더미 데이터가 필요하면 `supabase/seed.sql`에도 `INSERT` 문을 작성한다. *이 단계는 DB에 아직 아무 영향도 주지 않는, 순수 파일 편집이다.*
   2. **CLI 명령 실행**: 작성한 SQL 파일을 실제 로컬 DB에 반영하려면 `supabase db reset`(전체 재구성) 또는 `supabase migration up`(추가분만 반영) 명령을 실행한다. 두 명령의 차이는 10번 표 참고.
6. `.gitignore`에 `supabase/.branches`, `supabase/.temp` 등 로컬 전용 파일이 커밋되지 않도록 확인.

## 6. 수업 당일 워크플로우 (매 수업 반복)

> **`npm run dev`는 DB와 무관하다.** `npm run dev`는 React 프론트엔드(Vite 개발 서버)만 띄우는 명령이며, 테이블 생성·데이터 반영과는 전혀 관계없다. `.env`에 적힌 URL/anon key로 "이미 떠 있는" Supabase에 접속만 시도할 뿐이라, 그 대상 DB에 테이블이 없으면 화면에 에러만 뜬다. 순서는 항상 **① `supabase start`(DB 켜기) → ② 필요시에만 `db reset`/`migration up`(스키마 반영) → ③ `npm run dev`(화면 켜기)** 이다.
>
> **매 수업 ②번이 필요한 건 아니다.** 5번(최초 셋업)에서 마이그레이션을 한 번 적용해두면 데이터는 Docker 볼륨에 계속 남아있어서, `supabase stop` 후 다음 주에 `supabase start`만 해도 이전 데이터 그대로 이어진다. ②번을 다시 실행해야 하는 경우는 **테이블 구조를 새로 바꿨을 때**(migration 파일 추가 후 `migration up`) 또는 **데모 데이터를 깨끗이 지우고 싶을 때**(`db reset`)뿐이다.

1. 수업 시작 10~15분 전, 해당 반 폴더에서 `supabase start` 실행 (반마다 별도 터미널 탭).
2. `supabase status`로 정상 기동 확인 (초록불/URL 출력 확인).
3. (스키마를 새로 바꿨거나 데이터를 초기화하고 싶을 때만) `supabase migration up` 또는 `supabase db reset` 실행 — 평소엔 생략.
4. 데모용 React 앱의 `.env`가 그 반의 로컬 URL/anon key를 가리키는지 확인 후 `npm run dev`.
5. 수업 진행 — 두 반 모두 동일한 방식으로 독립 실행되므로 서로 영향 없음.
6. 수업 종료 후 해당 폴더에서 `supabase stop` (데이터는 Docker 볼륨에 보존되어 다음 주에도 이어서 사용 가능).

## 7. 당일 트러블슈팅 체크리스트

- `supabase start` 실패 → Docker Desktop이 켜져 있는지 먼저 확인
- 포트 충돌 에러 → 4번 배정표대로 `config.toml` 포트가 실제로 겹치지 않는지 재확인
- 두 스택 동시 실행 시 PC가 느려짐 → 불필요한 프로그램 종료, Docker Desktop 리소스 할당량 확인
- `.env`에 반영한 URL/anon key가 이전 반 것과 섞이지 않았는지 매번 확인 (반별로 값이 다름)

## 8. 클라우드 무료 슬롯 사용 원칙

- 클라우드 프로젝트 2개(개인용 1개 + 여유 1개)는 이 로컬 운영 계획과 무관하게 그대로 유지한다.
- 여유 슬롯 1개는 "실제 공개 배포 결과물"을 보여줘야 하는 특별한 시연이 필요할 때만 임시로 활성화하고, 시연이 끝나면 정리(일시정지 또는 삭제)한다.

## 9. 향후 반 확장 시

- 로컬 방식은 클라우드 프로젝트 개수 제한과 무관하므로, 반이 늘어나도 포트만 새로 배정하면 무제한으로 확장 가능하다.
- 새 반이 생기면 4번 포트 배정표에 행을 추가하고, 5번 셋업 절차를 그 반 폴더에서 한 번 수행한다.

## 10. Supabase CLI 명령어 정리표

이 문서에서 사용하는 명령어는 전부 **터미널에서 해당 반의 프로젝트 폴더로 이동한 뒤** 실행한다. 명령어는 Supabase 대시보드/MCP가 아니라 컴퓨터에 설치된 CLI 프로그램이다.

| 명령어 | 하는 일 | 사용 시점 | 데이터 영향 |
|---|---|---|---|
| `supabase --version` | CLI가 설치돼 있는지, 버전이 몇인지 확인 | 최초 설치 확인할 때 | 없음 |
| `supabase init` | 현재 폴더에 로컬 Supabase 설정(`supabase/` 폴더)을 새로 만듦 | 반 개설 시 1회 (3~5번 절차) | 없음 |
| `supabase migration new <이름>` | 비어있는 마이그레이션 SQL 파일을 생성 (`supabase/migrations/타임스탬프_이름.sql`) | 테이블을 새로 만들거나 구조를 바꿀 때, 그 안에 SQL을 채워 넣기 위해 | 파일만 생성, DB엔 아직 반영 안 됨 |
| `supabase start` | 로컬 Docker 컨테이너(DB·Auth·API·Studio 등)를 전부 켬 | **매 수업 시작 전** | 기존 데이터 유지, 없으면 새로 만듦 |
| `supabase status` | 지금 켜진 로컬 스택의 API 주소·anon key 등을 화면에 출력 | `.env`에 넣을 값을 확인할 때 | 없음 (조회만) |
| `supabase stop` | 로컬 컨테이너를 종료 | **매 수업 종료 후** | 데이터는 Docker 볼륨에 그대로 보존됨 |
| `supabase db reset` | `migrations` 폴더의 SQL + `seed.sql`을 처음부터 순서대로 다시 실행해 DB를 완전히 새로 만듦 | 스키마를 통째로 새로 깔고 싶을 때, 또는 실습 중 꼬인 데이터를 깨끗이 지우고 싶을 때 | ⚠️ **기존 데이터 전부 삭제됨** |
| `supabase migration up` | 아직 적용되지 않은 마이그레이션 파일만 순서대로 적용 | 기존 데이터는 남긴 채로 스키마 변경사항만 반영하고 싶을 때 | 기존 데이터 유지, 새 스키마만 추가 |

> **`db reset`과 `migration up`의 차이가 헷갈릴 때**: "그동안 쌓인 방명록/게시글 데이터를 지워도 되면" → `db reset`, "학생들이 입력한 데이터는 남기고 테이블 구조만 바꾸고 싶으면" → `migration up`.

## 11. GitHub Pages 배포 시 주의사항 (로컬 환경과 절대 혼동 금지)

이 문서의 로컬 Supabase는 **수업 중 강사 PC에서만 접속 가능한 환경**이다. 반면 GitHub Pages로 배포된 사이트는 전 세계 누구나 접속하므로, 로컬 주소(`localhost:포트`)를 절대 알 수 없다. **배포용 빌드에 로컬 URL이 들어가면 배포된 사이트는 100% 작동하지 않는다.**

**이미 올바르게 구성되어 있는 부분** (`my-portfolio` 등 이 과정의 프로젝트 기준, `.github/workflows/deploy.yml` 확인 완료):
- 배포 빌드(`npm run build`)는 로컬 `.env` 파일이 아니라 **GitHub repo secrets**(`VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY`)의 값을 빌드 시점에 주입받는다.
- `.env` 자체는 `.gitignore`에 포함되어 있어 저장소에 커밋되지 않으므로, 로컬 반별 포트 값이 실수로 배포에 섞여 들어갈 일은 없다.

**단, 아래는 실제로 사람이 챙겨야 한다:**

1. **repo secrets는 반드시 "실제 클라우드 프로젝트" 값이어야 한다.** GitHub Actions 러너는 강사 PC의 Docker에 접근할 수 없으므로, `VITE_SUPABASE_URL`/`VITE_SUPABASE_ANON_KEY` secrets에 로컬 `localhost` 주소를 넣는 실수를 하면 안 된다 — 8번의 클라우드 슬롯(개인용 또는 여유분) URL/key를 사용해야 한다.
2. **배포 대상 클라우드 프로젝트에도 스키마가 미리 적용되어 있어야 한다.** 로컬에서 `supabase db reset`/`migration up`으로 만든 테이블은 로컬 DB에만 존재한다. 실제로 공개 배포해서 보여줄 계획이라면, 같은 `supabase/migrations` 내용을 클라우드 프로젝트에도 반영해야 한다 (`supabase link --project-ref <ref>` 후 `supabase db push`, 또는 클라우드 프로젝트이므로 이 경우엔 Supabase MCP의 `apply_migration`도 사용 가능 — 5번과 달리 클라우드 대상이라 MCP 제약이 없음).
3. **시연이 끝나면** 8번 원칙대로 그 클라우드 슬롯을 정리(일시정지)한다.

**체크리스트**
- [ ] repo secrets(`VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY`)가 `localhost`가 아닌 실제 클라우드 프로젝트를 가리키는가
- [ ] 그 클라우드 프로젝트에 필요한 테이블/시드가 이미 반영되어 있는가
- [ ] 로컬 `.env`가 실수로 커밋되지 않았는가 (`.gitignore`에 `.env` 포함 여부 확인)
