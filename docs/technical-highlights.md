# 기술 구현 하이라이트

로즈앤 어시스트를 만들고 원내 도입을 준비하면서 풀었던 문제 중, 설계 판단이 담긴 것들을 **문제 → 원인 → 해결 → 검증** 순서로 정리했습니다. 코드는 실제 저장소에서 발췌했고, 설명에 필요한 부분만 남겼습니다(`…`는 생략).

1. [익명 예진의 조용한 데이터 유실](#1-익명-예진의-조용한-데이터-유실)
2. [엣지 함수 인증 계층 — PHI 유출 경로 차단](#2-엣지-함수-인증-계층--phi-유출-경로-차단)
3. [AI는 초안까지만 — 사람 확정 구조](#3-ai는-초안까지만--사람-확정-구조)
4. [칸반 현황판과 접수 경과시간 긴급도](#4-칸반-현황판과-접수-경과시간-긴급도)
5. [방문유형 오분류를 읽기 모델에서 바로잡기](#5-방문유형-오분류를-읽기-모델에서-바로잡기)
6. [PostgREST 임베드 필터 함정](#6-postgrest-임베드-필터-함정)
7. [데이터 모델 — 환자·방문 분리와 규약](#7-데이터-모델--환자방문-분리와-규약)
8. [보안 위생과 배포 전환](#8-보안-위생과-배포-전환)
9. [사후 설계 문서화](#9-사후-설계-문서화)

---

## 1. 익명 예진의 조용한 데이터 유실

**문제.** 환자는 로그인 없이 태블릿에서 예진을 제출합니다. 그런데 실제로는 이름·나이만 저장되고 **동의·서명·증상·예진·갱년기 점수가 전부 유실**되고 있었습니다. 화면에는 "제출 완료"가 떴기 때문에 아무도 몰랐습니다.

**원인.** 두 가지가 겹쳤습니다.
- 자식 테이블(증상·예진·건강정보 등)의 RLS가 로그인 사용자만 쓰기를 허용해 익명(`anon`) 삽입이 거부됨
- 방문 행 UPDATE는 RLS 정책이 있어도 익명에게 **0행 갱신으로 조용히 통과**(에러 없음)
- 제출 로직은 실패를 `console.error`로만 남기고 `success: true`를 반환

**해결.** "아무 방문이나"가 아니라 **방금 생성된, 아직 배정되지 않은 방문**에만 익명 쓰기를 허용하는 가드를 만들고, 방문 필드 갱신은 같은 가드를 다시 검사하는 `SECURITY DEFINER` RPC로 옮겼습니다.

```sql
-- 방금 생성된(2시간 이내) 미배정 방문인지 — 익명 쓰기 허용 범위의 기준
CREATE OR REPLACE FUNCTION public.is_fresh_unassigned_visit(p_visit_id uuid)
RETURNS boolean LANGUAGE sql STABLE SECURITY DEFINER SET search_path = public AS $$
  SELECT EXISTS (
    SELECT 1 FROM public.visits v
    WHERE v.id = p_visit_id
      AND v.assigned_doctor_id IS NULL
      AND v.deleted_at IS NULL
      AND v.created_at > now() - interval '2 hours'
  )
$$;

-- 자식 테이블은 같은 가드로 익명 INSERT 허용
CREATE POLICY "visit_symptoms_insert_anon_intake"
ON public.visit_symptoms FOR INSERT TO anon
WITH CHECK (public.is_fresh_unassigned_visit(visit_id));

-- 방문 필드는 RLS UPDATE 대신 가드를 재검사하는 RPC로
CREATE OR REPLACE FUNCTION public.set_intake_visit_fields(p_visit_id uuid, p_consent_given boolean, …)
RETURNS void LANGUAGE plpgsql SECURITY DEFINER SET search_path = public AS $$
BEGIN
  IF NOT public.is_fresh_unassigned_visit(p_visit_id) THEN
    RAISE EXCEPTION …;   -- 오래됐거나 이미 배정된 방문은 거부
  END IF;
  UPDATE public.visits SET consent_given = p_consent_given, … WHERE id = p_visit_id;
END $$;
```

제출 쪽은 실패를 모아 **성공으로 위장하지 않도록** 바꿨습니다.

```ts
const failures: string[] = [];
…
if (visitErr) { console.error("set_intake_visit_fields error:", visitErr); failures.push("동의·서명"); }
…
if (failures.length > 0) {
  return {
    success: false,
    visitId,
    error: `일부 정보 저장에 실패했습니다 (${failures.join(", ")}). 다시 시도하거나 직원에게 알려 주세요.`,
  };
}
```

**검증.** 순수 익명 세션으로 제출 → 동의(true)·서명·증상·갱년기 점수·건강정보가 모두 저장됨을 확인. 이 가드 덕분에 익명 사용자는 **배정이 끝났거나 시간이 지난 방문**은 건드릴 수 없습니다.

> **배운 점.** RLS의 UPDATE 정책은 조건이 안 맞으면 에러가 아니라 "0행"을 돌려줍니다. 권한 경계가 있는 쓰기는 RLS에만 기대지 말고, 가드를 스스로 검사하는 RPC로 만드는 편이 안전하고 관찰 가능합니다.

---

## 2. 엣지 함수 인증 계층 — PHI 유출 경로 차단

**문제.** REST API 설계 점검 중 발견했습니다. AI 기능을 담당하는 엣지 함수 5개가 모두 게이트웨이 JWT 검증(`verify_jwt`)을 끈 상태였는데, 함수 안에서 인증을 확인하는 건 1개뿐이었습니다. 특히 2개는 **service role 권한으로 방문 ID 하나만 받아 진료 메모·음성 요약을 반환** — URL만 알면 민감 의료정보를 꺼낼 수 있는 경로였습니다.

| 함수 | 본문 인증 | 위험 |
| --- | --- | --- |
| 환자 안내문 생성 | 없음 (service role) | **PHI 유출** |
| 진료요약 생성 | 없음 (service role) | **PHI 유출 + 무단 쓰기** |
| AI 브리핑·SOAP | 없음 | AI 키 비용 도용 |
| 음성 전사·요약 | 없음 | OpenAI·Anthropic 키 도용 |

**해결.** 각 함수 진입부에 의료진 역할 검증 가드를 넣었습니다. 대시보드 배포 방식(파일 단위 붙여넣기) 때문에 공유 모듈 대신 자체 완결형으로 인라인했습니다.

```ts
// verify_jwt=false 게이트웨이 보완 — 본문에서 토큰+역할을 직접 검증.
// 활성 의료진(staff/doctor/superadmin)만 허용. 통과 시 null, 거부 시 Response 반환.
async function requireMedicalStaff(req: Request): Promise<Response | null> {
  const authHeader = req.headers.get("Authorization");
  if (!authHeader) return json({ error: "Unauthorized" }, 401);
  const token = authHeader.replace("Bearer ", "");
  const userClient = createClient(SUPABASE_URL, ANON_KEY, {
    global: { headers: { Authorization: `Bearer ${token}` } },
  });
  const { data: { user }, error } = await userClient.auth.getUser();
  if (error || !user) return json({ error: "Unauthorized" }, 401);
  const admin = createClient(SUPABASE_URL, SERVICE_KEY);
  const { data: roleData } = await admin.rpc("get_user_role", { _user_id: user.id });
  const role = (roleData as { role?: string } | null)?.role;
  if (!role || !["staff", "doctor", "superadmin"].includes(role)) {
    return json({ error: "Forbidden" }, 403);
  }
  return null;
}

Deno.serve(async (req) => {
  if (req.method === "OPTIONS") return new Response("ok", { headers: corsHeaders });
  const denied = await requireMedicalStaff(req);
  if (denied) return denied;
  …
});
```

**검증 (배포 후 실측).**

| 호출 | 결과 |
| --- | --- |
| 인증 없음 → 4개 함수 | `401 Unauthorized` (배포 전에는 400 = 무방비) |
| 익명 키로 안내문 함수 호출 | `401` 차단 |
| 스태프 토큰 → 4개 함수 | 가드 통과 → 입력 검증 단계(400)까지 도달 |

"막을 건 막고, 정상 사용자는 통과"를 양방향으로 확인했습니다. 이 과정에서 엣지 함수 배포를 대시보드 붙여넣기에서 **Supabase CLI**로 전환했습니다.

---

## 3. AI는 초안까지만 — 사람 확정 구조

진료 내용을 환자가 이해하지 못한 채 귀가하는 문제를 풀기 위해, 진료 메모와 음성 요약을 **쉬운 말 안내문**으로 바꿔 인쇄해 드리는 기능을 만들었습니다. 환자에게 직접 나가는 문서라 설계 원칙을 명확히 했습니다.

- **AI는 초안까지만.** AI 출력은 저장하지 않습니다. 의사가 확인·수정해 **확정한 내용만** 저장·인쇄됩니다.
- **근거가 없으면 만들지 않음.** 메모도 음성 요약도 없으면 `422 no_source`로 초안 생성 자체를 거부합니다.
- **지어내지 않음.** 프롬프트에 환각 금지 제약을 명시했습니다.

```text
[작성 규칙]
- 따뜻하고 정중한 한국어 존댓말. 초등학교 고학년도 이해할 수준의 쉬운 말.
- 의학 용어는 반드시 풀어서 설명(예: "낭종" → "물혹", "경과 관찰" → "지켜보기").
- 입력에 있는 내용만 사용. 진단·약·수치를 절대 지어내지 않는다. 근거가 없으면 그 섹션은 생략.
- 음성 요약에는 잡음·오인식이 있을 수 있으니, 불확실하거나 앞뒤가 안 맞는 내용은 넣지 않는다.
```

같은 진료 기록이라도 **화면(사용자)에 따라 AI의 역할을 나눴습니다.**

| 기능 | 대상 | 페르소나 | 사람 검증 |
| --- | --- | --- | --- |
| AI 브리핑·SOAP | 원장 | 의료 전문 (임상 표기) | 참고 자료 |
| 음성 진료 요약 | 원장 | 의료 전문 | 참고 자료 |
| 환자 안내문 | 환자 | 환자 교육 (쉬운 말) | **의사 확정 필수** |
| 진료요약 발송문 | 환자 | 환자 교육 | 승인 워크플로 |

그 밖의 안전장치: AI 생성은 **자동이 아닌 수동 트리거**(비용·오남용 방지), AI 녹음은 **환자 동의 시에만** 가능, 음성 원본은 전사 후 **저장하지 않음**.

→ 자세한 내용은 [AI 명세서](ai-spec.md)

---

## 4. 칸반 현황판과 접수 경과시간 긴급도

**문제.** 기존 스태프 화면은 "미배정 목록 + 원장별 박스" 구조였는데, 박스마다 내부 스크롤바가 생겨 환자가 가려지고 한눈에 안 들어왔습니다. 오래 기다린 환자를 알아채기도 어려웠습니다.

**해결.**
- 미배정 | 원장별 **가로 칸반**으로 재구성해 중첩 스크롤 제거
- 카드 정렬: **진료중 → 대기중 → 완료**, 완료는 회색 처리
- 배정은 드롭다운 하나로, 상태는 카드의 상태 태그를 눌러 변경, 원장 간 이관 지원
- **접수 후 경과시간 배지**: 10분 단위로 색이 짙어지고 60분 이상은 긴급(⚠️)

```ts
function elapsedStyle(min: number) {
  if (min >= 60) return { bg: "#c0392b", color: "#ffffff", weight: 800, urgent: true };
  if (min >= 50) return { bg: "#ffcdd2", color: "#b71c1c", weight: 700, urgent: false };
  if (min >= 40) return { bg: "#ffccbc", color: "#bf360c", weight: 700, urgent: false };
  if (min >= 30) return { bg: "#ffe0b2", color: "#e65100", weight: 600, urgent: false };
  if (min >= 20) return { bg: "#fff3cd", color: "#f57f17", weight: 600, urgent: false };
  if (min >= 10) return { bg: "#e8f5e9", color: "#2e7d32", weight: 500, urgent: false };
  return { bg: "var(--bg-section)", color: "var(--text-muted)", weight: 500, urgent: false };
}
```

현황판은 10초 주기로 갱신되므로 별도 타이머 없이 색이 따라 바뀝니다. 완료 카드는 이미 처리됐으므로 배지를 숨깁니다.

---

## 5. 방문유형 오분류를 읽기 모델에서 바로잡기

**문제.** 포트폴리오 스크린샷을 찍다가 발견했습니다. 산부인과나 성형만 고른 환자에게도 **"검진" 태그**가 붙고, 문진표 원본의 방문목적도 `부인과 일반, 정기검진`으로 표시됐습니다.

**원인.** 제출 로직이 성형은 방문목적을 확인해 넘겼지만, 검진은 확인 없이 넘겼습니다. 초기값 `{ selectedItems: [] }`에 키가 있어 "비어 있지 않으면 저장" 검사를 통과했고, **모든 방문에 빈 검진 행이 쌓였습니다.** 방문유형은 이 행들에서 유도되므로 모든 환자가 검진으로 분류됐습니다.

**해결.** 두 곳을 고쳤습니다.

```ts
// ① 원천 — 검진도 선택했을 때만 전달 (새 제출부터 빈 행이 생기지 않음)
screening: hasScr ? ({ ...screening } as ExaminationData) : ({} as ExaminationData),
```

```ts
// ② 읽기 모델 — 내용 없는 행은 방문유형·섹션 데이터에서 제외
// null·빈 문자열·빈 배열만 든 데이터는 고르지 않은 섹션으로 본다.
// false·0은 환자가 고른 답(아니오·0회)이므로 내용으로 친다.
function examHasContent(data: unknown): boolean {
  if (!data || typeof data !== "object") return false;
  return Object.values(data as Record<string, unknown>).some((val) =>
    Array.isArray(val) ? val.length > 0 : val !== null && val !== undefined && val !== "",
  );
}

function flattenVisit(v: RawVisit): FlatVisit {
  const exams = (v.examinations || []).filter((e) => examHasContent(e.data));
  …
}
```

**판단.** 과거 데이터를 지우는 마이그레이션 대신 **읽기 모델에서 걸러내는 방식**을 택했습니다. 운영 데이터를 건드리지 않고, 이미 쌓인 과거 방문까지 즉시 바로잡힙니다. `false`와 `0`을 "내용 있음"으로 둔 건, "아니오"라는 답과 "고르지 않음"을 구분하기 위해서입니다.

**검증.** 데모 환자 7명으로 확인 — 검진을 고르지 않은 5명의 태그가 사라지고, 검진을 고른 2명만 HPV·액상세포가 표시됨.

---

## 6. PostgREST 임베드 필터 함정

**문제.** 예진 데이터에서 환자 이름으로 검색하면, 이름이 비어 있고 나이가 `?세`인 행이 대량으로 쏟아졌습니다.

**원인.** 이름 필터를 left join된 임베드(`patient:patients(…)`)에 걸면, PostgREST는 부모(방문) 행을 걸러내지 않고 **임베드만 null로** 만듭니다.

**해결.** 이름 검색일 때만 inner join 임베드로 바꿔 부모 행 자체를 필터합니다.

```ts
const selectStr = opts.patientNameLike
  ? VISIT_SELECT.replace("patient:patients(", "patient:patients!inner(")
  : VISIT_SELECT;
```

---

## 7. 데이터 모델 — 환자·방문 분리와 규약

초기에는 제출 한 건을 통째로 담는 `form_submissions` 테이블이었는데, 재방문 환자의 이력이 누적되지 않는 문제가 있어 **환자(`patients`)와 방문(`visits`)을 분리**하고 섹션별 하위 테이블로 정규화했습니다.

```
patients ─┬─ visits ─┬─ visit_examinations   (산부인과·검진·성형·갱년기 — 섹션당 1행)
          │          ├─ visit_symptoms
          │          ├─ visit_health_info / visit_medical_conditions
          │          ├─ visit_guardians       (미성년자 보호자)
          │          ├─ visit_ai_results      (브리핑·SOAP·전사문·사용 모델)
          │          └─ visit_consultations   (진료 메모·음성 요약·안내문 확정본)
          └─ (재방문 시 같은 환자에 방문이 누적)
```

- 방문 생성은 `create_visit` RPC가 이름·생년월일·연락처로 **기존 환자를 찾거나 새로 만듭니다.**
- DB enum은 한국어(`대기중`·`여성` 등), UI는 영어 키 — 변환은 한 파일에 모았습니다.
- 삭제는 모두 soft delete(`deleted_at`) — **퇴사한 원장을 비활성화해도 그가 진료한 기록과 이름은 보존**됩니다.
- 대시보드·브리핑·태그가 공통으로 쓰는 **평탄화 읽기 모델**(`flattenVisit`)을 하나 두어, 5번 같은 보정을 한 곳에서 할 수 있게 했습니다.

---

## 8. 보안 위생과 배포 전환

- **`.env` 히스토리 제거** — 저장소에 `.env`가 커밋돼 있던 것을 발견. 노출된 값은 공개용(anon) 키뿐이라 실제 위험은 낮았지만, 백업 번들을 만든 뒤 `git filter-repo`로 404개 커밋을 재작성해 히스토리에서 완전히 제거했습니다.
- **키 경계** — 서비스 롤 키와 AI API 키는 엣지 함수 환경변수에만 두고 클라이언트 번들에는 공개용 키만 둡니다.
- **DB 함수 `search_path` 고정**, 모든 테이블 RLS.
- **배포** — 엣지 함수를 대시보드 수동 배포에서 Supabase CLI로 전환. 프런트엔드는 Cloudflare Workers(SSR), 콜드스타트 완화를 위한 주기 핑을 GitHub Actions로 운영.

---

## 9. 사후 설계 문서화

이 시스템은 요구사항 정의나 설계 없이 빠르게 만들어졌습니다. 그래서 새 기능을 넘길 때마다 범위와 제약을 매번 다시 설명해야 했습니다. 코드·DB 스키마·개발 로그(130여 건)를 근거로 **역공학해 기준선 문서를 만들었습니다.**

- [요구사항 정의서](requirements.md) — 기능 요구 50여 항목마다 실제 구현 상태(구현 / 부분 / 미구현) 표기
- [AI 명세서](ai-spec.md) — AI 5개 파이프라인 명세 + **설계 원칙 대비 격차표**

AI 명세서를 쓰면서 배포된 함수를 직접 호출해 실측했고, 이 과정에서 **고급 모델 경로가 은퇴한 모델 ID를 지정하고 있어 실패하던 결함**을 찾아냈습니다. 에러가 전부 같은 메시지로 뭉개져 있어 사용자 화면에서는 원인을 알 수 없던 문제였습니다.
