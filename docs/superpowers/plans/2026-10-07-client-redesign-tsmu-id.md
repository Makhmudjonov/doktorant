# Klient redesign + TSMU ID — Implementation Plan

> **For agentic workers:** Use superpowers:executing-plans. Steps use `- [ ]`.

**Goal:** Klient qismini to'liq responsive redesign qilish va TSMU ID (docrest + InsightFace + SMS) identifikatsiyasini qo'shish.
**Architecture:** Backend (`project4`, Django/DRF) va frontend (`phd-admin`, Next 15 + antd 5 + Tailwind 4) alohida repolar, alohida branchlar; ular faqat quyidagi API shartnomasi orqali bog'lanadi.
**Spec:** `docs/superpowers/specs/2026-10-07-client-redesign-tsmu-id-design.md` (to'liq talablar shu yerda — har bir task uni o'qiydi).

## Global Constraints

- Branchlar: `project4` → `feature/tsmu-id`; `phd-admin` → `feature/client-redesign-tsmu-id`. Har task alohida commit. **Push qilinmaydi.**
- Admin panel (`app/admin-panel`, `app/(admin-panel)`, `components/admin`) va Payme kodi o'zgartirilmaydi (admin vizualini saqlash uchun zarur minimal CSS ko'chirishdan tashqari).
- Backend docker image Python 3.9 — `X | Y` type union va `match` ishlatilmaydi.
- Mavjud javob konventsiyasi: muvaffaqiyat `{"data": {...}, "message": "..."}`; xato `{"code": "UPPER_SNAKE", "message": "<o'zbekcha>"}` + mos HTTP status.
- `FACE_MATCH_THRESHOLD_PERCENT = 70`, kalibrlash nuqtalari `[(0.20, 0), (0.45, 70), (0.75, 100)]`, yaw farqi ≥ 15°, max 3 urinish, sessiya 30 daqiqa.
- UI matnlari o'zbek (lotin) tilida.

## API shartnomasi (`/api/v1/auth/tsmu-id/`)

```
POST lookup/            {pinfl:"14 raqam", birth_date:"YYYY-MM-DD"}
  200 {data:{verification_id:uuid, masked_name:"M********* M.", expires_at:iso}}
  404 DOCREST_NOT_FOUND | 503 DOCREST_UNAVAILABLE | 409 PINFL_TAKEN | 429 RATE_LIMITED | 400 VALIDATION_ERROR
POST face/              multipart: verification_id, frames (3 marta takrorlangan fayl maydoni, image/jpeg)
  200 {data:{passed:bool, similarity_pct:number, attempts_left:int, reason:null|"NO_FACE"|"MULTIPLE_FACES"|"LOW_SIMILARITY"|"LIVENESS_FAILED"}}
  409 INVALID_STATE | 410 SESSION_EXPIRED | 429 ATTEMPTS_EXHAUSTED
POST phone/             {verification_id, phone_number:"+998901234567"}
  200 {data:{sent:true, expires_in:600}} | 409 PHONE_TAKEN | 409 INVALID_STATE
POST phone/verify/      {verification_id, otp_code:"123456"}
  200 {data:{verified:true}} | 400 INVALID_OTP
POST complete/          {verification_id, password, confirm_password}
  201 {data:{tokens:{access,refresh}, user:User}}
POST attach/  (Bearer)  {verification_id}
  200 {data:{user:User}} | 403 FORBIDDEN | 409 INVALID_STATE
```

`User` (shuningdek `GET /auth/me/` va login javobida): mavjud maydonlar + `is_verified:bool`, `identity_verified_at:iso|null`, `photo_url:string|null`, `gender`, `nationality`, `citizenship`, `birth_place`.

`GET /applicant/profile/` javobiga ham `is_verified`, `identity_verified_at`, `photo_url` va `locked_fields: string[]` (tasdiqlangan bo'lsa read-only maydon nomlari) qo'shiladi.

`POST /applicant/submissions/create/` tasdiqlanmagan user uchun → `403 {code:"IDENTITY_NOT_VERIFIED", message}`. Submission detail javobida `applicant_snapshot` (object|null).

## Review Focus

1. JSHSHIR'ni bilgan begona odam `lookup/` orqali rasm/to'liq ism ololmasligi — javobda faqat masked_name.
2. `face/` ga `verification_id` boshqa user/IP dan qayta yuborilishi — holat mashinasi va urinishlar limiti ishlashi.
3. Kamera ruxsati rad etilgan / kamera yo'q / iOS Safari — wizard tushunarli xabar bilan to'xtashi.
4. 375px kenglikda uzun F.I.Sh, uzun ariza nomlari — gorizontal scroll yo'q, matn kesiladi/wrap bo'ladi.
5. Telefon orqali ro'yxatdan o'tgan eski user ariza formasini ochganda — forma emas, tasdiqlash gate'i.

---

## Backend (`project4`, branch `feature/tsmu-id`)

### Task B1: Test infratuzilmasi + requirements UTF-8
- Create `config/test_settings.py` (settings'ni import qilib, sqlite in-memory, `SECURE_SSL_REDIRECT=False`, debug_toolbar'siz). Run: `python manage.py test --settings=config.test_settings`.
- `requirements.txt` UTF-16 → UTF-8; dublikat `paytechuz` olib tashlanadi; `insightface==0.7.3`, `onnxruntime==1.19.2`, `opencv-python-headless==4.10.0.84`, `numpy<2` qo'shiladi.
- Commit: `chore: test settings and utf-8 requirements`

### Task B2: docrest client
- Create `src/infrastructure/docrest/{__init__,client}.py`, test `src/infrastructure/docrest/tests.py` (yoki `test/`).
- `PersonData` dataclass, `fetch_person(pinfl, birth_date) -> PersonData`, `DocrestNotFound`, `DocrestUnavailable` (spec §1.1). Settings: `DOCREST_API_URL`, `DOCREST_API_TOKEN`, `.env.example`.
- Testlar: agregat javob, to'g'ridan-to'g'ri javob, `result!="1"`, timeout → Unavailable, pasport `AE7515765` → `AE`/`7515765`, `sex` 1/2 → MALE/FEMALE, photo base64 decode.
- Commit: `feat(docrest): person lookup client`

### Task B3: Face matcher
- Create `src/infrastructure/face/{__init__,matcher}.py` + tests.
- `similarity_percent(cosine) -> float` (bo'lakli-chiziqli, clamp 0..100), `FaceMatcher` lazy singleton (`insightface` import funksiya ichida), `verify(reference_jpeg: bytes, frames: list) -> FaceVerdict(passed, similarity_pct, reason)`.
- Testlar `_analyze` (yoki `FaceAnalysis`) mock bilan: no face, multi face, low similarity, liveness fail (yaw farqi < 15), pass; `similarity_percent(0.45)==70`, `(0.2)==0`, `(0.9)==100`.
- Commit: `feat(face): insightface matcher with liveness`

### Task B4: Model + migratsiyalar
- `IdentityVerification` (spec §1.1 jadvali), `CustomUser`ga `identity_verified_at`, `nationality`, `citizenship`, `birth_place` (yo'q bo'lsa), `OTP.purpose`ga `TSMU_REGISTRATION`, `ApplicationSubmission.applicant_snapshot`. `makemigrations core`.
- Commit: `feat(core): identity verification model`

### Task B5: TSMU ID endpointlar
- Create `src/apps/auth/views/tsmu_id.py`, `src/apps/auth/serializers/tsmu_id.py`, `src/apps/auth/services/tsmu_id.py` (holat mashinasi, user yaratish/attach, snapshot uchun umumiy `apply_person_to_user(user, person_data)`), URL'lar `src/apps/auth/urls.py` ga.
- Rate-limit: Django cache asosida (`lookup` IP 10/soat, PINFL 5/soat).
- Testlar: to'liq oqim, INVALID_STATE, SESSION_EXPIRED, ATTEMPTS_EXHAUSTED, PINFL_TAKEN, PHONE_TAKEN, attach boshqa user → 403, lookup javobida `photo`/to'liq ism yo'qligi.
- Commit: `feat(auth): tsmu id verification endpoints`

### Task B6: Register OTP tuzatish + User serializer maydonlari
- `CompleteRegistrationSerializer`: so'nggi 30 daqiqada `is_used/verified` REGISTRATION OTP talab. User serializer(lar)iga `is_verified`, `identity_verified_at`, `photo_url`, yangi maydonlar.
- Testlar: OTP'siz complete → 400.
- Commit: `fix(auth): require verified otp before completing registration`

### Task B7: Profil read-only + ariza gate + snapshot
- Applicant profile serializer: `locked_fields`, tasdiqlangan userda read-only. `submissions/create/` gate (403 IDENTITY_NOT_VERIFIED) + `applicant_snapshot`. Submission serializerlarida `applicant_snapshot`.
- Testlar: 403, snapshot yozilgan, read-only maydon o'zgarmaydi.
- Commit: `feat(applicant): require verified identity and snapshot applicant data`

### Task B8: Docker
- `Dockerfile`: `apt-get install -y libgl1 libglib2.0-0`, build'da `python -c "from insightface.app import FaceAnalysis; FaceAnalysis(name='buffalo_l', providers=['CPUExecutionProvider']).prepare(ctx_id=-1)"`. `.env.example` yangilanadi.
- Commit: `chore(docker): insightface runtime deps and model preload`

---

## Frontend (`phd-admin`, branch `feature/client-redesign-tsmu-id`)

### Task F1: Dizayn tokenlari + AntdProvider
- `lib/theme/tokens.ts`, `app/globals.css` (`@theme` + `.dark`), `components/AntdProvider.tsx` tokenlarni oladi, Inter antd'ga ulanadi. Global `td,th` override admin layout'ga ko'chiriladi.
- Verify: `npm run build`. Commit: `feat(ui): design tokens`

### Task F2: Responsive shell
- `components/applicant/shell/{AppShell,Sidebar,Topbar,BottomNav}.tsx`, `components/applicant/ui/{PageHeader,Card,StatusBadge,ResponsiveTable,Sheet,VerifiedBadge}.tsx`; `app/(applicant)/layout.tsx` qayta yoziladi (spec §2.2). Eski `components/applicant/{Sidebar,Header}.tsx` o'chiriladi.
- Commit: `feat(ui): responsive applicant shell`

### Task F3: Public sahifalar
- Landing, `/login`, `/register` (tanlov + `?method=phone` eski oqim), `/forgot-password` redesign. Mantiq/endpointlar o'zgarmaydi.
- Commit: `feat(ui): redesign public auth pages`

### Task F4: TSMU ID API + wizard
- `lib/api/tsmuId.ts`, `types/index.ts` yangilanadi, `components/applicant/tsmu-id/{StepLookup,StepFace,StepPhone,StepPassword,StepDone,WizardProgress,useHeadPoseCapture}.tsx`, `app/(applicant)/register/tsmu-id/page.tsx`, `app/(applicant)/verify-identity/page.tsx`. `@mediapipe/tasks-vision` o'rnatiladi, dynamic import.
- Commit: `feat(tsmu-id): registration and verification wizard`

### Task F5: Profil (dashboard)
- Verified karta / banner, bo'limlar, `locked_fields` read-only, tahrirlash sheet faqat email/tashkilot/manzil.
- Commit: `feat(ui): redesign profile page`

### Task F6: Arizalar ro'yxati + ariza formasi bo'linishi + gate
- `app/(applicant)/applications/page.tsx` kartalar; `applications/[id]/page.tsx` → `components/applicant/application-form/{ApplicantSummaryCard,SpecialitySection,EducationFormSection,DynamicFieldsSection,DynamicField,SubmitBar,VerifyIdentityGate}.tsx`. Yuborish mantiqi (FormData shakli) o'zgarmaydi.
- Commit: `feat(ui): redesign application list and form with identity gate`

### Task F7: Mening arizalarim
- Ro'yxat (`ResponsiveTable`) va detail (timeline, snapshot, hujjatlar, amallar).
- Commit: `feat(ui): redesign my submissions`

### Task F8: Tekshiruv
- `npm run lint`, `npm run build`; Playwright 375/768/1280 skrinshotlar (backend'siz — API mock yoki faqat public sahifalar + mocked localStorage), scrollWidth tekshiruvi.
