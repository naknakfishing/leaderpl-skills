---
name: leaderpl-admin
description: HTML로 만든 작은 사이트(Vercel 배포)에 문의 폼 저장(/api 함수 + Supabase)과 비밀번호로 잠긴 /admin 관리자 페이지를 만든다. 문의 목록·상태(새 문의 → 연락함 → 완료)·48시간 알림·30일 추이·CSV 같은 관리 기능도 이 기준으로 추가한다. 사용자가 원하면 Supabase Auth로 회원가입·로그인·회원 전용 페이지와 관리자 화면의 회원 목록도 붙인다. 사용자가 "문의 폼", "Supabase에 저장", "관리자 페이지", "/admin", "문의함", "회원가입", "로그인"을 말하면 쓴다.
---

# 리더플 관리자 페이지 스킬

리더플AI(이신영)가 「리더플 AI 스쿨 실습실」 수업용으로 만든 스킬이다.
코딩을 모르는 사람이 만든 사이트에 **문의를 저장하고, 폰으로 확인하고, 처리 상태를 관리하는 화면**을 붙인다.

이 스킬이 하는 일: 사이트 폴더 안에 파일을 만들고 고친다(`api/`, `admin/`, 문의 폼). Supabase 표도 일회용 통로로 직접 만들고 바로 지운다. 사용자가 직접 할 일은 관리자 비밀번호 입력 하나다.
이 스킬이 하지 않는 일: 비밀번호·키 값을 묻거나 파일에 적기, 데이터 지우기, 사이트 밖으로 데이터 보내기.

## 0. 먼저 확인할 것 (매번)

1. 이 사이트가 **HTML 파일 + Vercel `/api` 함수** 구조인지 본다. Next.js 같은 다른 구조면 그 구조의 방식으로 바꿔서 같은 원칙을 지킨다.
2. 이미 `api/inquiry.js`, `admin/`, inquiries 표 SQL이 있는지 본다. 있으면 **새로 만들지 말고 그 칸 이름에 맞춘다.**
3. Supabase는 **Vercel 프로젝트 → Storage에서 연결**한 것을 전제로 한다. 연결하면 Vercel 환경변수에 주소와 키가 자동으로 들어간다.

## 1. 지켜야 할 원칙 (가장 중요)

- **키와 비밀번호는 코드·대화·파일 어디에도 값으로 적지 않는다.** 환경변수 이름으로만 읽는다.
  - Supabase 주소: `SUPABASE_URL` 없으면 `NEXT_PUBLIC_SUPABASE_URL`
  - Supabase 서버 키: `SUPABASE_SERVICE_ROLE_KEY` 없으면 `SUPABASE_SECRET_KEY`
  - 관리자 비밀번호: `ADMIN_PASSWORD` — 값은 **사용자가 Vercel 화면에서 직접** 넣는다
- **브라우저 쪽 코드(HTML·JS)에는 Supabase 키가 하나도 없다.** 저장·조회·수정은 전부 `/api` 함수가 한다.
- 표에는 **RLS(행 보안)를 켜고 정책은 만들지 않는다** → 공개 키로는 아무도 읽고 쓸 수 없고, 서버 키를 가진 `/api`만 접근한다.
- 패키지 설치 없이 동작하게 한다(`fetch`로 Supabase REST API 호출). `package.json`이 없어도 된다.
  `package.json`에 `"type": "module"`이 있으면 `/api` 파일을 `module.exports` 대신 `export default`로 쓴다.
- `.gitignore`에 `.env*`와 `.vercel`을 넣는다. 커밋 전에 `git diff --cached`로 키처럼 보이는 값이 없는지 본다. 이미 올라간 키는 지우기 전에 "Vercel/Supabase에서 키를 새로 바꿔야 해요"라고 먼저 알린다.
- 개인정보: 문의 폼에 **수집·이용 동의(필수)**, 관리자 페이지는 **검색 노출 금지(noindex)**.
- 설명은 쉬운 한국어로. 끝나면 "내가 할 일"을 번호로 알려 준다.

## 2. 문의 저장

### 2-0. 표는 클로드가 만든다 (기본 방법)

사용자에게 SQL Editor를 시키지 않는다. 아래 순서로 클로드가 직접 만든다.

1. `api/setup-db.js`(일회용)를 만들고, `pg` 패키지를 잠깐 쓸 수 있게 한다.
   - `package.json`이 **없으면** `"pg": "^8.13.0"` 하나만 든 `package.json`을 새로 만든다.
   - `package.json`이 **이미 있으면** 덮어쓰지 말고 dependencies에 `"pg": "^8.13.0"` 한 줄만 더한다. 원래 내용을 기억해 둔다.
   `api/setup-db.js` 내용은 2-1의 SQL을 실행하는 함수:
   ```js
   // 한 번만 쓰는 표 만들기 통로 — 표가 생기면 이 파일은 지운다. 데이터는 읽지도 보내지도 않는다.
   const { Client } = require('pg');
   const SQL = `/* 2-1의 SQL 전체 */`;
   module.exports = async (req, res) => {
     if (req.method !== 'POST') return res.status(405).json({ error: 'POST만 가능해요' });
     const url = process.env.POSTGRES_URL_NON_POOLING || process.env.POSTGRES_URL;
     if (!url) return res.status(500).json({ error: '데이터베이스 연결 정보가 없어요. Vercel Storage에서 Supabase를 연결해 주세요.' });
     const client = new Client({ connectionString: url.replace(/[?&]sslmode=[^&]*/, ''), ssl: { rejectUnauthorized: false } });
     try { await client.connect(); await client.query(SQL); return res.status(200).json({ ok: true }); }
     catch (e) { return res.status(500).json({ error: '표를 만들지 못했어요', detail: String(e.message).slice(0, 200) }); }
     finally { await client.end().catch(() => {}); }
   };
   ```
2. 커밋 → 푸시 → Vercel 자동 배포를 1~2분 기다린다(`GET 사이트주소/api/setup-db`가 405를 돌려주면 배포 완료).
3. 클로드가 직접 `POST 사이트주소/api/setup-db` 를 한 번 보낸다 → `{"ok":true}`.
4. **바로 `api/setup-db.js`를 지우고**, `package.json`은 1에서 새로 만들었으면 지우고, 원래 있던 파일이면 `pg` 한 줄만 빼서 원래대로 되돌린 뒤 커밋 → 푸시. (`/api/setup-db`가 404가 되면 끝)
5. 사용자에게 "Supabase → Table Editor → inquiries 에서 표를 볼 수 있어요"라고 알려 준다.

안 될 때만(예: `POSTGRES_URL`이 없음) 아래 2-1 SQL을 사용자에게 주고 Supabase **SQL Editor**에서 Run 하게 한다.
2026-10-06 새싹 공방 데모에서 이 순서로 표 생성 → 문의 저장 → /admin 조회까지 확인했다.

### 2-1. 표 SQL (2-0이 실행하는 내용 · 막힐 때만 SQL Editor에 붙여 넣기)

폼에 받을 칸이 다르면 `message` 자리에 칸을 바꾸거나 더한다. 상태 값 세 개는 바꾸지 않는다.

```sql
create table if not exists public.inquiries (
  id bigint generated always as identity primary key,
  created_at timestamptz not null default now(),
  name text not null,
  contact text not null,
  message text,
  consent boolean not null default false,
  status text not null default '새 문의' check (status in ('새 문의', '연락함', '완료')),
  updated_at timestamptz
);
alter table public.inquiries enable row level security;
revoke all on public.inquiries from anon, authenticated;
-- 정책은 만들지 않는다: 브라우저(공개 키)로는 접근 불가, 서버(/api)만 접근
```

### 2-2. `api/inquiry.js` — 문의 받기

```js
const SB_URL = process.env.SUPABASE_URL || process.env.NEXT_PUBLIC_SUPABASE_URL;
const SB_KEY = process.env.SUPABASE_SERVICE_ROLE_KEY || process.env.SUPABASE_SECRET_KEY;
const clip = (v, n) => String(v ?? '').trim().slice(0, n);
const sbHeaders = () => ({
  apikey: SB_KEY,
  ...(String(SB_KEY).startsWith('eyJ') ? { Authorization: `Bearer ${SB_KEY}` } : {}),
  'Content-Type': 'application/json',
});

module.exports = async (req, res) => {
  if (req.method !== 'POST') return res.status(405).json({ error: 'POST만 가능해요' });
  if (!SB_URL || !SB_KEY) return res.status(500).json({ error: 'Supabase 연결 정보가 없어요. Vercel Storage 연결 후 다시 배포해 주세요.' });
  let b = req.body || {};
  if (typeof b === 'string') { try { b = JSON.parse(b); } catch { b = {}; } }
  if (b.website) return res.status(200).json({ ok: true }); // 스팸 함정 칸(사람에게는 안 보임)
  const row = { name: clip(b.name, 50), contact: clip(b.contact, 100), message: clip(b.message, 1000), consent: b.consent === true };
  if (!row.name || !row.contact) return res.status(400).json({ error: '이름과 연락처를 적어 주세요' });
  if (!row.consent) return res.status(400).json({ error: '개인정보 수집·이용에 동의해 주세요' });
  const r = await fetch(`${SB_URL}/rest/v1/inquiries`, { method: 'POST', headers: { ...sbHeaders(), Prefer: 'return=minimal' }, body: JSON.stringify(row) });
  if (!r.ok) return res.status(500).json({ error: '저장하지 못했어요. 잠시 뒤 다시 시도해 주세요.' });
  return res.status(200).json({ ok: true });
};
```

### 2-3. 문의 폼 (사이트 HTML)

- 칸: 이름 · 연락처 · (PRD의 "문의 폼에서 받을 것" 하나) · 동의 체크(필수, 목적·보관 기간을 한 줄로)
- 화면에 안 보이는 `website` 칸 하나(스팸 함정, `tabindex="-1"`, `autocomplete="off"`)
- 제출: `fetch('/api/inquiry', { method: 'POST', headers: {'Content-Type':'application/json'}, body: JSON.stringify(값) })`
- 동의 값은 반드시 `consent: 동의체크박스.checked`(true/false)로 보낸다. 폼 값을 그대로 묶으면 `"on"`이 가서 체크해도 저장이 거절된다
- 완료 문구: "문의가 접수됐어요. 확인 후 연락드릴게요." — **지킬 수 없는 답변 시간은 쓰지 않는다**
- 실패 문구는 서버가 준 한국어 메시지를 그대로 보여 준다
- 질문이 많을수록 문의가 줄어든다. 칸은 최소로

## 3. 관리자 페이지 `/admin`

### 3-1. `api/admin.js` — 비밀번호 확인 + 목록 + 상태 바꾸기

```js
const crypto = require('node:crypto');
const SB_URL = process.env.SUPABASE_URL || process.env.NEXT_PUBLIC_SUPABASE_URL;
const SB_KEY = process.env.SUPABASE_SERVICE_ROLE_KEY || process.env.SUPABASE_SECRET_KEY;
const sbHeaders = () => ({
  apikey: SB_KEY,
  ...(String(SB_KEY).startsWith('eyJ') ? { Authorization: `Bearer ${SB_KEY}` } : {}),
  'Content-Type': 'application/json',
});
const same = (a, b) => {
  const h = (s) => crypto.createHash('sha256').update(String(s)).digest();
  return crypto.timingSafeEqual(h(a), h(b));
};
const STATUSES = ['새 문의', '연락함', '완료'];

module.exports = async (req, res) => {
  res.setHeader('Cache-Control', 'no-store');
  res.setHeader('X-Robots-Tag', 'noindex, nofollow');
  const pass = process.env.ADMIN_PASSWORD;
  if (!pass) return res.status(500).json({ error: 'Vercel 환경변수에 ADMIN_PASSWORD를 넣고 Redeploy 해 주세요.' });
  if (!same(req.headers['x-admin-password'] || '', pass)) return res.status(401).json({ error: '비밀번호가 맞지 않아요.' });
  if (!SB_URL || !SB_KEY) return res.status(500).json({ error: 'Supabase 연결 정보가 없어요.' });

  if (req.method === 'GET') {
    const r = await fetch(`${SB_URL}/rest/v1/inquiries?select=*&order=created_at.desc&limit=500`, { headers: sbHeaders() });
    if (!r.ok) return res.status(500).json({ error: '불러오지 못했어요.' });
    return res.status(200).json({ rows: await r.json() });
  }
  if (req.method === 'PATCH') {
    let b = req.body || {};
    if (typeof b === 'string') { try { b = JSON.parse(b); } catch { b = {}; } }
    const id = Number(b.id);
    if (!Number.isInteger(id) || !STATUSES.includes(b.status)) return res.status(400).json({ error: '잘못된 요청이에요.' });
    const r = await fetch(`${SB_URL}/rest/v1/inquiries?id=eq.${id}`, {
      method: 'PATCH', headers: { ...sbHeaders(), Prefer: 'return=minimal' },
      body: JSON.stringify({ status: b.status, updated_at: new Date().toISOString() }),
    });
    if (!r.ok) return res.status(500).json({ error: '바꾸지 못했어요.' });
    return res.status(200).json({ ok: true });
  }
  return res.status(405).json({ error: '지원하지 않는 요청이에요.' });
};
```

### 3-2. `admin/index.html` — 화면 (주소: `내사이트/admin`)

- `<meta name="robots" content="noindex, nofollow">`
- CSS에 `[hidden]{display:none!important}`를 넣는다(로그인 칸에 display를 주면 숨김이 안 먹는 흔한 실수 방지)
- 처음엔 비밀번호 입력칸만. 입력하면 `/api/admin`에 `x-admin-password` 헤더로 보낸다
- 비밀번호는 **sessionStorage**(탭을 닫으면 사라짐)에만 잠깐 둔다. localStorage·쿠키·URL에 넣지 않는다
- 맨 위에 **새 문의 개수**를 크게. 그 아래 최신순 목록: 날짜 · 이름 · 연락처 · 내용 · 상태 버튼
- 상태 버튼은 `새 문의 → 연락함 → 완료` 세 개. 누르면 PATCH 후 목록 새로고침
- 연락처는 눌러서 복사할 수 있게(`navigator.clipboard.writeText`, 실패하면 선택 상태로)
- **폰 화면 우선**: 표 대신 카드 목록, 글자 16px 이상, 버튼 손가락 크기
- 사이트 디자인 색을 따르되 관리 화면은 단정하게(장식 없이)
- 비밀번호가 틀리면 "비밀번호가 맞지 않아요."만. 다른 정보는 보여 주지 않는다. 입력칸은 비우고 다시 입력할 수 있게 포커스

### 3-3. 끝나면 사용자에게 알려 줄 "내가 할 일"

1. (처음 한 번) Vercel → 내 프로젝트 → **Settings → Environment Variables** → Key `ADMIN_PASSWORD`, Value 내가 정한 비밀번호(짧거나 쉬운 것 말고) → Save
   비밀번호는 사용자가 직접 정해서 넣는다. 클로드가 대신 만들거나 파일·대화에 적지 않는다.
2. **Deployments** → 맨 위 배포 → **Redeploy** (환경변수는 다시 배포해야 적용돼요)
3. 폰에서 `내사이트/admin` → 비밀번호 → 테스트 문의 확인 → 상태 바꿔 보기
4. 비밀번호는 클로드 대화창·채팅·화면공유에 쓰지 않기

### 3-4. 끝나기 전 클로드가 스스로 점검 (결과를 다섯 줄로 보고)

1. 비밀번호 없이 `GET 내사이트/api/admin` → 401
2. 브라우저 쪽 파일(HTML·JS) 어디에도 서버 키·비밀번호 값이 없다
3. Supabase에서 `inquiries` 표의 RLS가 켜져 있다
4. 깃 기록에 `.env` 파일이 없다 (`git log --all --name-only | grep -c "^\.env"` → 0)
5. 테스트로 넣은 문의는 목록으로 알려 주고 지우지 않는다(지울지는 사용자가 정한다)
확인하지 못한 항목은 못 했다고 쓴다.

## 4. 관리 기능 더하기 (사용자가 하나를 고르면)

기존 목록과 상태 바꾸기는 그대로 둔다. 서버 키는 계속 `/api`에만.

| 기능 | 만드는 법 |
|---|---|
| **48시간 알림** | `status === '새 문의'` 이고 `created_at`이 48시간보다 오래된 문의를 빨간 테두리로, 맨 위에 "답장 늦은 문의 N건" |
| **30일 추이** | 최근 30일을 날짜별로 세어 막대그래프(라이브러리 없이 CSS 막대). 오늘이 오른쪽 끝 |
| **CSV 내려받기** | 화면의 목록을 CSV로. 엑셀 한글 깨짐 방지로 맨 앞에 BOM(`﻿`). 파일명 `inquiries_날짜.csv`. 버튼 옆에 "개인정보가 들어 있어요. 보관에 주의하세요." |
| **상태별 보기** | 새 문의 / 연락함 / 완료 / 전체 필터 버튼 |
| **메모 칸** | 표에 `memo text` 칸 추가 SQL을 먼저 안내 → 관리자 화면에서 문의마다 한 줄 메모 저장(PATCH) |

더 큰 기능(콘텐츠 직접 수정, 완료 후 자동 삭제, 알림 메일)은 표와 `/api`가 더 필요하다. 바로 만들지 말고 **필요한 것과 순서를 먼저 설명**한다.

## 5. 회원가입·로그인 (사용자가 원할 때만)

관리자 페이지를 만들 때나 만든 뒤에 사용자가 "회원가입도", "로그인도", "회원 전용 페이지"를 말하면 이 절대로 더한다. 묻지 않았으면 먼저 만들지 않는다.
Supabase **Auth**(이메일 + 비밀번호)를 쓴다. 문의함(`inquiries`)과 관리자 비밀번호(`ADMIN_PASSWORD`)는 그대로 두고 **따로** 붙인다. 손님 회원 로그인과 사장님 관리자 로그인은 다른 문이다.

### 5-0. 만들기 전에 사용자에게 먼저 말할 것 (짧게, 번호로)

1. 회원 정보(이메일)를 받으면 **개인정보 처리방침**이 필요하다. 내용은 지어내지 않고 `privacy.html` 자리만 만든다. 실제 내용은 사용자가 채운다.
2. Supabase 화면에서 사용자가 직접 할 설정 두 가지(5-5). 메뉴 이름은 바뀔 수 있으니 "제 기준으로는"을 붙인다.
3. 받는 것은 **이메일·비밀번호·동의 체크**만. 이름·전화번호 같은 칸은 사용자가 원할 때만 더한다.

### 5-1. 원칙

- **브라우저에는 공개 키(anon/publishable)만** 쓴다. 공개 키는 로그인 창구용이라 노출돼도 되지만, 서버 키(`SUPABASE_SERVICE_ROLE_KEY`·`SUPABASE_SECRET_KEY`)는 계속 `/api`에만.
- 공개 키는 HTML에 값으로 적지 않고 `/api/auth-config`가 환경변수에서 읽어 내려 준다(키를 바꿔도 코드 수정 없음).
  - 주소: `SUPABASE_URL` → `NEXT_PUBLIC_SUPABASE_URL`
  - 공개 키: `SUPABASE_ANON_KEY` → `NEXT_PUBLIC_SUPABASE_ANON_KEY` → `SUPABASE_PUBLISHABLE_KEY` → `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY`
- 패키지 설치 없이 `fetch`로 Supabase Auth REST(`/auth/v1/...`)를 부른다.
- 로그인 표(세션)는 **sessionStorage**에 둔다(탭을 닫으면 로그아웃). 오래 유지하자고 하면 그때 localStorage로 바꾸고 이유를 말한다.
- 회원 전용 내용은 HTML에 그대로 두지 않고 `/api/members`가 **로그인 확인 후에만** 내려 준다(HTML에 두면 로그인 없이도 소스 보기로 보인다).
- 비밀번호는 **8자 이상**. 비밀번호 값은 어디에도 저장·출력하지 않는다.

### 5-2. `api/auth-config.js` — 공개 정보만 내려 주기

```js
module.exports = (req, res) => {
  const url = process.env.SUPABASE_URL || process.env.NEXT_PUBLIC_SUPABASE_URL;
  const key = process.env.SUPABASE_ANON_KEY || process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY
    || process.env.SUPABASE_PUBLISHABLE_KEY || process.env.NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY;
  if (!url || !key) return res.status(500).json({ error: 'Supabase 공개 키가 없어요. Vercel Storage 연결 후 다시 배포해 주세요.' });
  res.setHeader('Cache-Control', 'no-store');
  return res.status(200).json({ url, key }); // 공개 키만. 서버 키는 절대 여기 넣지 않는다
};
```

### 5-3. 화면 — `signup.html` · `login.html` · `members.html`

- 처음에 `/api/auth-config`로 `url`·`key`를 받는다. 모든 Auth 요청 헤더: `apikey: key`, `Content-Type: application/json`
- **회원가입**: `POST {url}/auth/v1/signup` · body `{ email, password, data: { consent: true, consent_at: 지금시각 } }`
  - 동의 체크(필수, 목적·보관 기간 한 줄 + `privacy.html` 링크)가 `.checked`일 때만 보낸다
  - 응답에 `access_token`이 있으면 바로 로그인 상태로, 없으면 "확인 메일을 보냈어요. 메일의 링크를 누른 뒤 로그인해 주세요."
- **로그인**: `POST {url}/auth/v1/token?grant_type=password` · body `{ email, password }` → `access_token`을 sessionStorage에 두고 `members.html`로
- **비밀번호 찾기**: `POST {url}/auth/v1/recover` · body `{ email }` → "메일을 확인해 주세요" (가입 여부는 알려 주지 않는다)
- **로그아웃**: `POST {url}/auth/v1/logout`(헤더에 `Authorization: Bearer 토큰`) 후 sessionStorage 비우기
- **members.html**: 토큰이 없으면 `login.html`로. 있으면 `/api/members`에 `Authorization: Bearer 토큰`으로 요청해 받은 내용만 보여 준다. 401이면 토큰 지우고 로그인으로
- 오류 문구는 쉬운 한국어로: 비밀번호 틀림·없는 계정은 똑같이 "이메일 또는 비밀번호가 맞지 않아요."
- 세 페이지와 `members.html`은 `noindex`. 첫 화면 메뉴에 "로그인" 링크 하나
- 폰 화면 우선, 사이트 디자인 색을 따른다

### 5-4. `api/members.js` — 로그인한 사람에게만 내용 주기

```js
module.exports = async (req, res) => {
  res.setHeader('Cache-Control', 'no-store');
  const url = process.env.SUPABASE_URL || process.env.NEXT_PUBLIC_SUPABASE_URL;
  const key = process.env.SUPABASE_ANON_KEY || process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY
    || process.env.SUPABASE_PUBLISHABLE_KEY || process.env.NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY;
  const token = String(req.headers.authorization || '').replace(/^Bearer\s+/i, '');
  if (!url || !key) return res.status(500).json({ error: 'Supabase 연결 정보가 없어요.' });
  if (!token) return res.status(401).json({ error: '로그인이 필요해요.' });
  const r = await fetch(`${url}/auth/v1/user`, { headers: { apikey: key, Authorization: `Bearer ${token}` } });
  if (!r.ok) return res.status(401).json({ error: '다시 로그인해 주세요.' });
  const user = await r.json();
  return res.status(200).json({
    email: user.email,
    // 회원 전용 내용: 사용자가 준 내용만. 없으면 "준비 중"
    content: '회원 전용 내용은 준비 중이에요.',
  });
};
```

### 5-5. 사용자가 Supabase 화면에서 할 일 (끝나면 번호로 알려 준다)

1. Vercel → Storage → my-site-db → **Open in Supabase** → **Authentication**
2. **URL Configuration** → Site URL에 내 사이트 주소(https://…vercel.app 또는 내 도메인) → Save. 확인 메일 링크가 이 주소로 돌아온다
3. (수업·테스트용) **Sign In / Providers → Email**에서 **Confirm email**을 끄면 확인 메일 없이 바로 가입된다. 실제로 손님을 받을 때는 다시 켜는 것을 권한다
   - Supabase 기본 메일은 시간당 보낼 수 있는 수가 아주 적다. 가입이 많으면 메일 발송 설정(SMTP)이 따로 필요하다고 한 줄로 알려 준다

### 5-6. 관리자 페이지에 "회원" 보기 더하기 (사용자가 원하면)

- `api/admin.js`의 GET에 `?view=members`를 더한다. 같은 관리자 비밀번호 확인을 거친 뒤 서버 키로
  `GET {SB_URL}/auth/v1/admin/users?page=1&per_page=200` (헤더는 `sbHeaders()`) → `users`에서 **이메일 · 가입일 · 마지막 로그인**만 골라 돌려준다
- `admin/index.html`에 [문의함 | 회원] 탭. 맨 위에 회원 수
- 회원 삭제·비밀번호 바꾸기·메일 보내기 버튼은 만들지 않는다. 필요하면 Supabase 화면에서 하도록 안내한다

### 5-7. 확인 (사용자와 같이)

1. 테스트 이메일로 가입 → (Confirm email을 켰다면 메일 링크) → 로그인 → `members.html`에 내 이메일이 보인다
2. 로그아웃 후 `members.html` 주소로 바로 들어가면 로그인 화면으로 돌아간다
3. `/admin` → 회원 탭에 방금 가입한 이메일이 보인다
4. 사이트 파일 어디에도 서버 키 값이 없다(공개 키도 값으로 적혀 있지 않다)

## 6. 막혔을 때 확인 순서

1. Vercel → Storage에 Supabase가 이 프로젝트에 **Connected**인가
2. Supabase **Table Editor**에 `inquiries` 표가 있는가 (없으면 2-0을 다시, 그래도 안 되면 SQL Editor에서 2-1 실행)
3. 환경변수를 넣은 뒤 **Redeploy** 했는가
4. 브라우저 개발자 도구가 아니라 **사용자가 본 화면 문구**를 캡처로 받아 원인을 쉬운 말로 설명한다
5. Supabase를 새로 만들 때 "무료 프로젝트 한도" 안내가 뜨면 → 클로드는 기존 프로젝트를 지우거나 멈추거나 결제하지 않는다. 사용자에게 (a) 기존 데이터베이스를 같이 쓰고 표 이름 앞에 사이트 약칭을 붙이기 (b) 사용자가 직접 정리한 뒤 다시 하기 중에서 고르게 한다
6. (회원가입) 가입했는데 로그인이 안 되면 → 확인 메일을 눌렀는지, 또는 5-5의 Confirm email 설정. 메일이 안 오면 → 시간당 발송 한도. 메일 링크가 엉뚱한 주소로 가면 → 5-5의 Site URL
