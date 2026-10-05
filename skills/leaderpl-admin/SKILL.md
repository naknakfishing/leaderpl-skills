---
name: leaderpl-admin
description: HTML로 만든 작은 사이트(Vercel 배포)에 문의 폼 저장(/api 함수 + Supabase)과 비밀번호로 잠긴 /admin 관리자 페이지를 만든다. 문의 목록·상태(새 문의 → 연락함 → 완료)·48시간 알림·30일 추이·CSV 같은 관리 기능도 이 기준으로 추가한다. 사용자가 "문의 폼", "Supabase에 저장", "관리자 페이지", "/admin", "문의함"을 말하면 쓴다.
---

# 리더플 관리자 페이지 스킬

리더플AI(이신영)가 「리더플 AI 스쿨 실습실」 수업용으로 만든 스킬이다.
코딩을 모르는 사람이 만든 사이트에 **문의를 저장하고, 폰으로 확인하고, 처리 상태를 관리하는 화면**을 붙인다.

이 스킬이 하는 일: 사이트 폴더 안에 파일을 만들고 고친다(`api/`, `admin/`, 문의 폼). 그리고 사용자가 직접 할 일(SQL 실행, 환경변수 입력)을 안내한다.
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
- 개인정보: 문의 폼에 **수집·이용 동의(필수)**, 관리자 페이지는 **검색 노출 금지(noindex)**.
- 설명은 쉬운 한국어로. 끝나면 "내가 할 일"을 번호로 알려 준다.

## 2. 문의 저장

### 2-1. 표 만들기 SQL — 사용자가 Supabase **SQL Editor**에 붙여 넣고 Run

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

1. (처음 한 번) Vercel → 내 프로젝트 → **Settings → Environment Variables** → Key `ADMIN_PASSWORD`, Value 내가 정한 비밀번호 → Save
2. **Deployments** → 맨 위 배포 → **Redeploy** (환경변수는 다시 배포해야 적용돼요)
3. 폰에서 `내사이트/admin` → 비밀번호 → 테스트 문의 확인 → 상태 바꿔 보기
4. 비밀번호는 클로드 대화창·채팅·화면공유에 쓰지 않기

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

## 5. 회원가입·로그인을 물으면

"Supabase **Auth**로 만들 수 있어요"라고 답하고, 만들기 전에 필요한 것(개인정보 처리방침, 동의 문구, Supabase Authentication 설정, 브라우저에는 공개 키만)을 먼저 정리한다. 처리방침 내용은 지어내지 않고 자리만 만든다.

## 6. 막혔을 때 확인 순서

1. Vercel → Storage에 Supabase가 이 프로젝트에 **Connected**인가
2. Supabase **Table Editor**에 `inquiries` 표가 있는가 (없으면 SQL Editor에서 2-1 실행)
3. 환경변수를 넣은 뒤 **Redeploy** 했는가
4. 브라우저 개발자 도구가 아니라 **사용자가 본 화면 문구**를 캡처로 받아 원인을 쉬운 말로 설명한다
