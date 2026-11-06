# Supabase 일시중지(pause) 방지 설정

Supabase 무료 플랜은 **일정 기간 API 요청이 전혀 없으면 프로젝트를 일시중지**합니다.
일시중지된 뒤에는 API 요청으로 깨울 수 없고 **대시보드에서 수동 복구(Restore)** 해야 합니다.
따라서 중지되기 *전에* 주기적으로 요청을 보내는 것이 유일한 예방책입니다.

방어선은 두 겹입니다.

| | 수단 | 동작 조건 | 역할 |
|---|---|---|---|
| 1 | **GitHub Actions** (`.github/workflows/supabase-keepalive.yml`) | 앱을 열지 않아도 매일 자동 | **주 방어선** |
| 2 | 앱 내부 ping (`index.html`) | 앱을 열었을 때만 | 보조 |

---

## 1단계 — 이미 중지됐는지 먼저 확인

keepalive는 **살아 있는 프로젝트를 유지**하는 장치이고, 중지된 프로젝트를 깨우지는 못합니다.

1. https://supabase.com/dashboard 접속
2. 해당 프로젝트가 `Paused` 상태면 **Restore** 클릭 (복구에 몇 분 소요)
3. 복구가 끝난 뒤 아래 2단계를 설정

---

## 2단계 — GitHub 시크릿 2개 등록

저장소 → **Settings → Secrets and variables → Actions → New repository secret**

| 시크릿 이름 | 값 |
|---|---|
| `SUPABASE_URL` | `https://<프로젝트ID>.supabase.co` |
| `SUPABASE_ANON_KEY` | Supabase 대시보드 → Settings → API → `anon` `public` 키 |

> `anon` 키는 원래 브라우저에 노출되는 공개 키이고 RLS로 보호됩니다.
> 그래도 저장소에 평문으로 커밋하지 않도록 시크릿으로 넣습니다.
> **`service_role` 키는 절대 넣지 마세요.**

---

## 3단계 — 동작 확인

저장소 → **Actions → Supabase Keepalive → Run workflow** (수동 실행)

로그에 이렇게 나오면 정상입니다.

```
otp_monthly → HTTP 200
otp_settings → HTTP 200
✅ keepalive 성공 — 2026-11-06 18:13 UTC
```

이후에는 **매일 한국시간 03:13**에 자동 실행됩니다.

### 결과별 의미

| 로그 | 의미 | 할 일 |
|---|---|---|
| `HTTP 200` | 정상 | 없음 |
| `HTTP 404` | 테이블 없음 (요청은 도달 → 유지는 됨) | 앱 💾 → ☁️ 클라우드 탭의 SQL 실행 |
| `HTTP 401/403` | 키 오류 | `SUPABASE_ANON_KEY` 다시 등록 |
| `도달하지 못했습니다` | 이미 중지됐거나 URL 오류 | 대시보드에서 Restore / URL 확인 |

실패하면 GitHub가 등록된 메일로 알려줍니다.

---

## ⚠️ 꼭 알아둘 제약

**GitHub는 저장소에 60일간 아무 활동이 없으면 예약 워크플로를 자동으로 비활성화합니다.**
(비활성화 전에 메일로 경고가 옵니다.)

- 평소처럼 이 저장소에 커밋을 하고 있다면 신경 쓸 일 없습니다.
- 오래 손대지 않을 예정이라면, 두 달에 한 번쯤 **Actions 탭에서 Run workflow**를 눌러주거나
  아무 커밋이나 하나 올리면 카운터가 초기화됩니다.
- 메일로 경고가 왔다면 그때 **Enable workflow**를 눌러 다시 켜면 됩니다.

---

## 앱 내부 ping (보조 수단)

`index.html`이 다음 시점에 Supabase REST에 가벼운 요청을 보냅니다.

- 앱을 열었을 때
- 탭을 다시 활성화했을 때
- 탭을 오래 열어둔 경우 6시간마다

같은 기기에서는 **최소 6시간 간격**으로만 보내 불필요한 요청을 막습니다.
마지막 요청 시각과 일시중지까지 남은 일수는 앱의 **💾 → ☁️ 클라우드** 탭에서 볼 수 있고,
같은 화면의 **💓 지금 깨우기** 버튼으로 즉시 보낼 수도 있습니다.

이 ping은 **앱을 열어야만** 동작하므로, 1단계의 GitHub Actions를 반드시 함께 쓰세요.
