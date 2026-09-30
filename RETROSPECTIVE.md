# 작업 회고록: Claude Code 한국어 문서 번역 프로젝트

> **프로젝트**: https://code.claude.com/docs/ 전체 문서 한국어 번역
> **결과물**: https://jkf87kc.github.io/claude-code-docs-ko/
> **저장소**: [jkf87kc/claude-code-docs-ko](https://github.com/jkf87kc/claude-code-docs-ko)
> **기간**: 2026-04-14 ~ 2026-04-15 (약 24시간)
> **작성일**: 2026-04-15

---

## 1. 프로젝트 개요

Anthropic의 Claude Code 공식 문서(112페이지)를 한국어로 완역하고, VitePress 기반 정적 사이트로 GitHub Pages에 배포하는 프로젝트.

### 최종 성과
- ✅ **112/112 문서 번역 완료** (100%)
- ✅ **VitePress 빌드 성공** (gh-pages에 358 files 배포)
- ✅ **라이브 사이트 정상 동작** (HTTP 200 확인)

---

## 2. 타임라인

| 시간 | 단계 | 내용 |
|---|---|---|
| 4/14 오전 | 프로젝트 초기 설정 | VitePress 프로젝트 생성, `package.json`에 `"type": "module"` 설정 (ESM 필수), `config.ts` 기본 구성 |
| 4/14 오전 | 원문 수집 | `curl`로 https://code.claude.com/docs/ 에서 107개 .md 파일 다운로드 (`source/` 디렉토리) |
| 4/14 오전~오후 | ❌ 서브에이전트 방식 실패 | `sessions_spawn`으로 batch1-8 워커 스폰 시도 → 전부 실패/타임아웃 |
| 4/14 오후 | Claude Code `--print` 전환 | `coding-agent` 스킬 활용, `claude --print --permission-mode bypassPermissions`로 백그라운드 실행 |
| 4/14 오후 | 첫 번째 커밋 | batch1으로 43/112 완료 → 커밋 `6bdf198` |
| 4/14 오후~저녁 | SIGTERM 반복 | batch2, batch3이 OS에 의해 강제 종료 (메모리/타임아웃 문제로 추정) |
| 4/14 저녁 | ohmyclaw 계획 수립 | Planner(GLM-5.1)가 T1-T6 계획 수립 |
| 4/14 밤~4/15 새벽 | 나머지 21개 번역 | 3개 Claude Code 프로세스 병렬 실행 → 108/112 달성 |
| 4/15 오전 | 최종 4개 직접 번역 | env-vars(214줄), features-overview(310줄) 직접 작성, python(3234줄), changelog(2991줄)은 Claude Code로 처리 |
| 4/15 오전 | 빌드 에러 해결 | `${{ secrets }}` Vue 보간 에러, `<Update>` 태그 에러 수정 |
| 4/15 오전 | 배포 완료 | gh-pages에 358 files 푸시, 최종 커밋 `fe521e2` |

---

## 3. 시도한 접근법과 결과

### ❌ 방법 1: OpenClaw 서브에이전트 (spawn)
- **방식**: `sessions_spawn`으로 워커 에이전트 생성, batch1-8 매니페스트 분할
- **결과**: 전부 실패
- **원인**: 서브에이전트 샌드박스가 파일 시스템 접근을 격리하여 워크스페이스 파일을 읽고 쓸 수 없음
- **교훈**: 파일 I/O가 필요한 작업에는 `sessions_spawn` 부적합

### ⚠️ 방법 2: Claude Code `--print` 모드 (부분 성공)
- **방식**: `claude --print --permission-mode bypassPermissions`로 백그라운드 exec
- **결과**: batch1 성공, batch2-3 SIGTERM
- **장점**: 워크스페이스 파일 직접 읽고 쓰기 가능
- **문제**: 3000줄+ 대형 파일은 10분+ 실행 시 OS가 SIGTERM으로 강제 종료
- **교훈**: 대형 파일은 `nohup`으로 실행하거나 청크 분할 필요

### ✅ 방법 3: 직접 번역 (AI가 직접 작성)
- **방식**: `read`로 원문 읽고 `write`로 번역본 작성
- **결과**: env-vars(214줄), features-overview(310줄) 성공
- **장점**: 프로세스 종료 리스크 없음, 정밀한 품질 관리 가능
- **단점**: 대형 파일(3000줄+)은 컨텍스트 윈도우 제약으로 어려움

### ✅ 방법 4: Claude Code + nohup (대형 파일)
- **방식**: `nohup claude --print ...`으로 OS 시그널 방지
- **결과**: python.md(3234줄), changelog.md(2991줄) 모두 성공
- **교훈**: 장시간 Claude Code 실행 시 `nohup` 필수

---

## 4. 해결한 기술 이슈

### 4.1 VitePress `type: "module"` 필수
- **증상**: `esbuild` 빌드 충돌
- **원인**: VitePress 1.6.4가 ESM 전용인데 `package.json`에 `"type": "module"` 누락
- **해결**: `package.json`에 `"type": "module"` 추가

### 4.2 `cleanUrls: false` 필수
- **증상**: GitHub Pages에서 404
- **원인**: `cleanUrls: true`가 `index.html` 생성을 생략
- **해결**: `config.ts`에 `cleanUrls: false` 설정

### 4.3 `${{ secrets }}` Vue 보간 에러
- **증상**: `TypeError: Cannot read properties of undefined (reading 'ANTHROPIC_API_KEY')`
- **원인**: VitePress가 마크다운을 Vue SFC로 컴파일할 때 `${{ ... }}`를 Vue 템플릿 보간으로 해석. 코드블록 내부도 예외 없음
- **시도한 것** (전부 실패):
  - `\${{` 백슬래시 이스케이프 → VitePress가 `\`을 strip
  - `&dollar;{{` HTML 엔티티 → VitePress가 엔티티를 원래 문자로 복원
- **해결**: `${ '{' secrets.ANTHROPIC_API_KEY '}' }` — `${`와 `{{` 사이에 공백과 문자열 분리

### 4.4 `<Update>` 커스텀 태그 에러
- **증상**: `Element is missing end tag` 빌드 에러 (changelog.md)
- **원인**: Claude Code가 `<Update label="..." description="...">` 태그를 그대로 번역에 유지
- **해결**: Python 스크립트로 `<Update>` → `<details><summary>` 변환

### 4.5 YAML frontmatter 콜론 에러
- **증상**: `incomplete explicit mapping pair` VitePress 빌드 에러
- **원인**: description에 `:` 포함 시 YAML 파서 오류
- **해결**: description 값을 따옴표로 감싸기

### 4.6 Python SSL 인증 에러
- **증상**: `python3 urllib`로 파일 다운로드 시 `[SSL: CERTIFICATE_VERIFY_FAILED]`
- **해결**: `curl`로 우회

### 4.7 gh-pages 배포 방법
- **시도**: orphan 브랜치 + `git checkout` (같은 worktree에서 불가)
- **해결**: `dist/` → `/tmp/` 복사 후 독립 `git init` → `git push -f origin gh-pages`

---

## 5. 번역 규칙 (최종 확정)

| 항목 | 규칙 |
|---|---|
| frontmatter `title` | 한국어 |
| frontmatter `description` | 한국어 (콜론/특수문자 시 따옴표 필수) |
| 기술 용어 | 영어 유지 (Hooks, Plugins, Agent SDK, Ultraplan, Bedrock, Vertex AI, Foundry 등) |
| 코드 블록 | 원본 보존 |
| 내부 링크 | `/en/xxx` → `/xxx` |
| Footer | "Was this page helpful?" 및 "© Anthropic" 제거 |
| 커스텀 컴포넌트 | `<Note>/<Tip>/<Info>/<Warning>` → `::: tip/info/warning` |
| 문체 | 자연스러운 한국어 설명문 스타일 |

---

## 6. 아키텍처 최종 구성

```
claude-code-docs-ko/
├── docs/                    # VitePress 소스 (112개 한국어 .md)
│   ├── .vitepress/
│   │   ├── config.ts        # VitePress 설정 (cleanUrls:false, base:/claude-code-docs-ko/)
│   │   └── dist/            # 빌드 출력
│   ├── agent-sdk/           # Agent SDK 하위 28개 문서
│   ├── whats-new/           # 주간 업데이트 3개
│   └── *.md                 # 최상위 문서 81개
├── source/                  # 원문 107개 (다운로드한 영어 원본)
├── manifests/               # batch 분할 매니페스트 (batch1-8.tsv)
├── package.json             # type: "module" 필수
└── RETROSPECTIVE.md         # 이 파일
```

### VitePress 설정 요약
```typescript
{
  cleanUrls: false,           // GitHub Pages 호환
  ignoreDeadLinks: true,      // 미번역 링크 허용
  base: '/claude-code-docs-ko/',
  lang: 'ko-KR'
}
```

### 배포 파이프라인
```
npx vitepress build docs
  → cp -r docs/.vitepress/dist /tmp/gh-pages-ccd
  → cd /tmp/gh-pages-ccd && git init && git add -A && git commit
  → git push -f origin gh-pages
```

---

## 7. 커밋 이력

| 커밋 | 내용 | 파일 수 |
|---|---|---|
| `79992e9` | Claude Code What's New 한국어 번역 (Week 13-15) | 초기 |
| `6bdf198` | 번역 진행: agent-sdk 전체 28개 + 기타 15개 (43/112) | 169 |
| `2983a1c` | 번역 111/112: env-vars, features-overview, python 포함 | 76 |
| `fe521e2` | 번역 완료 112/112 + 빌드/배포 수정 | 9 |

---

## 8. 남은 정리 작업 (TODO)

- [ ] **Sidebar/Nav 점검**: `config.ts`의 사이드바가 모든 112개 문서를 정확히 반영하는지 확인
- [ ] **내부 링크 검증**: 번역된 문서 간 상호 링크가 올바른지 점검
- [ ] **용어 일관성 감사**: Hooks/Hook, Skills/Skill, 서브에이전트 등 번역 용어 통일
- [ ] **QA**: 전체 페이지 렌더링 품질 확인 (표 깨짐, 마크다운 렌더링 이슈 등)
- [ ] **changelog.md 품질**: Claude Code가 자동 번역한 changelog의 용어 일관성 점검

---

## 9. 배운 점

1. **서브에이전트 ≠ 파일 작업 워커**: OpenClaw의 `sessions_spawn`은 샌드박스 환경이라 로컬 파일 I/O에 부적합. 파일이 필요한 작업은 `exec`로 Claude Code CLI를 직접 실행해야 함.

2. **VitePress는 마크다운 전체를 Vue 템플릿으로 컴파일**: 코드블록 안의 `${{ }}`도 Vue 보간으로 해석됨. GitHub Actions 문서에서 반드시 주의 필요.

3. **장시간 Claude Code 실행에는 nohup 필수**: `--print` 모드가 10분 이상 실행되면 macOS가 SIGTERM으로 강제 종료. `nohup`으로 해결.

4. **`cleanUrls: false`는 GitHub Pages 필수**: Vercel/Netlify에서는 `true`가 편하지만, GitHub Pages에서는 `index.html`이 있어야 404를 피할 수 있음.

5. **번역 규칙은 작업 전에 확정**: 중간에 규칙이 바뀌면 일관성이 깨짐. 첫 batch에서 확정하고 문서화한 게 큰 도움이 됨.

6. **빌드-배포-확인 주기**: 번역 완료 후 한 번에 빌드하면 에러 원인을 찾기 어려움. batch마다 빌드 테스트하는 게 안전함.
