# 리더플 스킬 (leaderpl-skills)

리더플AI(이신영)가 「리더플 AI 스쿨 실습실」 수업용으로 만든 클로드 스킬 모음입니다.
누구나 무료로 쓸 수 있어요.

| 스킬 | 하는 일 |
|---|---|
| [leaderpl-site-prd](skills/leaderpl-site-prd/SKILL.md) | 사이트를 만들기 전에 한 번에 한 질문씩 인터뷰해서 설계도(PRD) 한 장을 `docs/PRD.md`로 저장해요. 사이트 코드는 만들지 않아요. |
| [leaderpl-admin](skills/leaderpl-admin/SKILL.md) | 문의 폼 저장(Supabase)과 비밀번호로 잠긴 `/admin` 관리자 페이지를 만들어요. 키와 비밀번호는 코드에 적지 않고 환경변수로만 읽어요. |

## 설치하는 법 — 클로드에게 시키기

클로드 데스크탑 앱의 **Code** 화면에서 아래 순서대로 말하면 돼요.

1. 출처와 내용 먼저 확인
   ```
   이 저장소의 스킬 두 개(leaderpl-site-prd, leaderpl-admin)를 설치하기 전에 확인하고 싶어.
   https://github.com/naknakfishing/leaderpl-skills
   각 SKILL.md를 읽고 무엇을 하는 스킬인지 한 줄씩 알려 줘.
   파일을 지우거나 밖으로 보내는 명령, 비밀번호나 키를 달라는 부분이 있으면 그 줄을 보여 줘.
   아직 설치하지는 마.
   ```
2. 설치
   ```
   문제가 없으면 두 스킬을 내 개인 스킬 폴더(~/.claude/skills)에 설치해 줘.
   끝나면 설치된 폴더 위치를 알려 줘.
   ```
3. **새 대화**를 열고 확인
   ```
   지금 쓸 수 있는 스킬 목록을 보여 줘.
   ```

## 이 스킬들이 하지 않는 것

- 비밀번호·API 키를 묻거나 파일에 적기
- 데이터를 지우거나 사이트 밖으로 보내기
- 숫자·후기·경력을 지어내기

## 만든 사람

리더플AI 이신영 · leaderpl@naver.com · https://leaderpl.com

라이선스: MIT (LICENSE 파일)
