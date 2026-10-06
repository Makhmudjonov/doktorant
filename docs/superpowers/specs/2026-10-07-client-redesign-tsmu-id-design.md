# Klient qismi redesign + TSMU ID identifikatsiya — Dizayn spec

**Sana:** 2026-10-07
**Loyihalar:** `phd-admin` (Next.js frontend), `project4` (Django backend)
**Holat:** tasdiqlash kutilmoqda

## Maqsad

1. Ariza beruvchi (klient) qismini to'liq qayta chizish — zamonaviy SaaS uslubi, mobil/planshet/desktop'ga to'liq moslashgan.
2. **TSMU ID** — JSHSHIR + tug'ilgan sana bo'yicha davlat bazasidan (docrest) shaxsiy ma'lumot va rasm olish, kamera orqali yuzni solishtirish (≥70%) va liveness, so'ng telefon (SMS) + parol bilan akkaunt yaratish.
3. Ariza yuborish faqat shaxsi TSMU ID orqali tasdiqlangan foydalanuvchilarga ruxsat etiladi; ariza paytidagi shaxsiy ma'lumotlar ariza ichida muzlatiladi.

**Muvaffaqiyat mezoni:** 375px, 768px, 1280px kengliklarda barcha klient sahifalari gorizontal scroll'siz ishlaydi; foydalanuvchi JSHSHIR → yuz → SMS → parol orqali akkaunt ochib, keyin telefon+parol bilan kirib, ariza yubora oladi; tasdiqlanmagan foydalanuvchi ariza yubora olmaydi (backend 403).

## Qarorlar (foydalanuvchi bilan kelishilgan)

| Savol | Qaror |
|---|---|
| Yuz solishtirish | InsightFace (ArcFace, `buffalo_l`, onnxruntime CPU), **backend'da** |
| Liveness | Oddiy: brauzer MediaPipe bilan bosh burish ko'rsatmasi, 3 kadr; tekshiruv **serverda** (pose + bir odam) |
| Eski telefon+SMS ro'yxatdan o'tish | **Qoladi**, lekin ariza yuborish uchun TSMU ID bilan shaxsni tasdiqlash majburiy |
| Vizual yo'nalish | Zamonaviy SaaS: neytral zinc + indigo aksent, Inter |
| Admin panel | Tegilmaydi |
| Payme | Tegilmaydi (hozircha kerak emas) |

## Tashqi manba: docrest

`POST {DOCREST_API_URL}` header `Authorization: Token {DOCREST_API_TOKEN}`, body `{"pinfl": <int>, "birth_date": "YYYY-MM-DD"}`, timeout 15s.

Javob ikki ko'rinishda kelishi mumkin — client ikkalasini ham qabul qiladi:
- agregat: `{"docrest": {"result": "1", "data": [...]}, "mehnat": ..., ...}`
- to'g'ridan-to'g'ri: `{"result": "1", "data": [...]}`

`data[0]` dan ishlatiladigan maydonlar: `surnamelat/namelat/patronymlat` (+ `*cyr`, `eng*` zaxira), `birth_date`, `birthplace`, `birthcountry`, `nationality`, `citizenship`, `sex` (1=erkak, 2=ayol), `current_document`, `documents[]` (`document`, `docgiveplace`, `datebegin`, `dateend`), `photo` (**base64 JPEG**, prefikssiz).

`result != "1"` yoki bo'sh `data` → "topilmadi". Tarmoq xatosi / non-200 → "xizmat javob bermadi".

## 1. Backend (`project4`)

### 1.1 Yangi modullar

**`src/infrastructure/docrest/client.py`**
- `fetch_person(pinfl: str, birth_date: date) -> PersonData` (dataclass): `last_name, first_name, middle_name, full_name_cyr, full_name_eng, birth_date, birth_place, birth_country, nationality, citizenship, gender ('MALE'|'FEMALE'|None), passport_series, passport_number, passport_issued_by, passport_issued_date, passport_expiry_date, photo_bytes`.
- Pasport: `current_document` ni `documents[]` dan topadi; `"AE7515765"` → series `"AE"`, number `"7515765"`.
- Xatolar: `DocrestNotFound`, `DocrestUnavailable`.
- Settings: `DOCREST_API_URL`, `DOCREST_API_TOKEN` (env), `.env.example` ga qo'shiladi.

**`src/infrastructure/face/matcher.py`**
- InsightFace `FaceAnalysis(name="buffalo_l", providers=["CPUExecutionProvider"])`, `det_size=(640,640)`, lazy thread-safe singleton (birinchi so'rovda yuklanadi).
- `embed(image_bytes) -> FaceResult(embedding, yaw, det_score)`; `NoFaceDetected`, `MultipleFacesDetected` xatolari (eng katta yuz emas — bir nechta yuz bo'lsa xato).
- `similarity_percent(cosine: float) -> float` — xom cosine'ni kalibrlangan foizga aylantiradi (ArcFace'da bir odamning hujjat rasmi vs selfi odatda 0.35–0.65 cosine beradi; to'g'ridan-to'g'ri `cosine*100` ishlatilmaydi). Boshlang'ich kalibrlash — bo'lakli-chiziqli: cosine ≤ 0.20 → 0%, 0.45 → 70%, ≥ 0.75 → 100% (oraliqlarda chiziqli). Ya'ni 70% chegara ≈ cosine 0.45 (ArcFace uchun hujjat-vs-selfi amaliyotidagi odatiy qiymat). Nuqtalar settings'da (`FACE_CALIBRATION_POINTS`) — real ma'lumotlarda sozlanadi. Chegara: `FACE_MATCH_THRESHOLD_PERCENT = 70` (settings).
- `verify(reference_jpeg, frames: list[bytes]) -> FaceVerdict(similarity_pct (min across frames), liveness_ok, reason)`:
  - har bir kadrda aynan bitta yuz;
  - har bir kadr reference bilan `similarity_pct ≥ 70`;
  - kadrlar o'zaro bir odam (`similarity_pct ≥ 70`);
  - `max(yaw) - min(yaw) ≥ 15°` (liveness).
- Model fayllari Docker image build vaqtida yuklab olinadi (`~/.insightface/models`), runtime'da internet talab qilinmaydi.
- `requirements.txt` (UTF-16 → UTF-8 ga o'giriladi): `insightface`, `onnxruntime`, `opencv-python-headless`, `numpy`.

**`src/core/models/identity_verification.py` — `IdentityVerification`**

| Maydon | Turi |
|---|---|
| `id` | UUID pk |
| `pinfl` | char(14), index |
| `birth_date` | date |
| `person_data` | JSON (PersonData, `photo_bytes`dan tashqari) |
| `reference_photo` | ImageField (`identity/ref/`) |
| `status` | `LOOKED_UP`, `FACE_PASSED`, `PHONE_PENDING`, `PHONE_VERIFIED`, `COMPLETED`, `FAILED`, `EXPIRED` |
| `face_score` | float null |
| `face_attempts` | int, max 3 |
| `phone_number` | phonenumber null |
| `user` | FK CustomUser null (attach yoki complete'dan keyin) |
| `initiated_by` | FK CustomUser null (attach oqimi uchun — kim boshlagan) |
| `ip_address` | GenericIP |
| `created_at`, `expires_at` (created + 30 daqiqa) | datetime |

Selfi kadrlar **saqlanmaydi** — faqat ball.

### 1.2 Endpointlar — `/api/v1/auth/tsmu-id/`

Hammasi `verification_id` bilan ishlaydi; muddati o'tgan yoki noto'g'ri holatdagi sessiya → `409 {code: "INVALID_STATE"}`.

| Endpoint | Auth | Kirish | Chiqish / harakat |
|---|---|---|---|
| `lookup/` | ixtiyoriy | `{pinfl, birth_date}` | docrest chaqiradi → `{verification_id, masked_name}`. PINFL boshqa akkauntga bog'langan bo'lsa (va so'rovchi o'sha user bo'lmasa) → `409 {code: "PINFL_TAKEN"}`. Rate-limit: IP bo'yicha 10/soat, PINFL bo'yicha 5/soat |
| `face/` | ixtiyoriy | multipart `verification_id`, `frames[]` (3 ta JPEG, har biri ≤ 2MB) | `{passed, similarity_pct, attempts_left, reason}`. 3 urinishdan keyin `FAILED` |
| `phone/` | yo'q | `{verification_id, phone_number}` | holat `FACE_PASSED` talab; telefon band bo'lsa `409 PHONE_TAKEN`; OTP (purpose `TSMU_REGISTRATION`) yuboradi |
| `phone/verify/` | yo'q | `{verification_id, otp_code}` | → `PHONE_VERIFIED` |
| `complete/` | yo'q | `{verification_id, password, confirm_password}` | `CustomUser` (is_verified=True, docrest maydonlari, `photo`=reference) + `ApplicantProfile` yaratadi → `{tokens, user}`; → `COMPLETED` |
| `attach/` | **majburiy** | `{verification_id}` | holat `FACE_PASSED` va `initiated_by == request.user` talab; joriy user'ga docrest ma'lumotlarini yozadi, `is_verified=True` → `{user}` |

`masked_name`: `"MAXMUDJONOV MIRAZIZBEK"` → `"M********* M."`. To'liq ma'lumot va rasm `lookup` javobida **qaytarilmaydi** (JSHSHIR'ni bilgan begona odam shaxsiy ma'lumotni ko'ra olmasligi uchun). To'liq ma'lumot faqat `complete/` / `attach/` dan keyin `/auth/me/` va `/applicant/profile/` orqali ko'rinadi.

### 1.3 Mavjud kodga o'zgarishlar

- `CustomUser`: yangi maydonlar `identity_verified_at` (datetime null), `nationality`, `citizenship`, `birth_place` (agar yo'q bo'lsa). `is_verified` — TSMU ID tasdig'i belgisi.
- `OTP.purpose` ga `TSMU_REGISTRATION` qo'shiladi.
- **Xavfsizlik tuzatish:** `CompleteRegistrationSerializer` — telefon uchun so'nggi 30 daqiqada tasdiqlangan `REGISTRATION` OTP bo'lishini talab qiladi.
- Profil `PATCH /applicant/profile/`: `is_verified=True` bo'lsa TSMU ID'dan kelgan maydonlar (ism, familiya, otasining ismi, tug'ilgan sana/joy, jins, millat, fuqarolik, pinfl, pasport) o'zgartirilmaydi (serializer read-only). Tahrirlanadi: email, tashkilot, doimiy manzil.
- `GET /applicant/profile/` va `/auth/me/`: `is_verified`, `identity_verified_at`, `photo_url` qaytaradi.
- `ApplicationSubmission`: `applicant_snapshot` JSONField (null). `submissions/create/` — `request.user.is_verified` bo'lmasa `403 {code: "IDENTITY_NOT_VERIFIED"}`; yaratishda snapshot (F.I.Sh, pinfl, tug'ilgan sana, jins, pasport, telefon, email, tashkilot, manzil, rasm URL) yoziladi.

### 1.4 Docker

`Dockerfile`: `libgl1`, `libglib2.0-0` paketlari; build bosqichida `buffalo_l` modelini oldindan yuklash. `.env.example`: `DOCREST_API_URL`, `DOCREST_API_TOKEN`, `FACE_MATCH_THRESHOLD_PERCENT=70`.

## 2. Frontend (`phd-admin`, faqat `app/(applicant)` va `components/applicant`)

### 2.1 Dizayn tizimi

`app/globals.css` — Tailwind v4 `@theme` tokenlari (light + `.dark`):

| Token | Light | Dark |
|---|---|---|
| `--color-bg` | `#fafafa` | `#09090b` |
| `--color-surface` | `#ffffff` | `#18181b` |
| `--color-surface-2` | `#f4f4f5` | `#27272a` |
| `--color-border` | `#e4e4e7` | `#27272a` |
| `--color-text` | `#18181b` | `#fafafa` |
| `--color-muted` | `#71717a` | `#a1a1aa` |
| `--color-primary` | `#4f46e5` | `#6366f1` |
| `--color-success` / `warning` / `danger` | `#16a34a` / `#d97706` / `#dc2626` | `#22c55e` / `#f59e0b` / `#ef4444` |

- Radius: karta 12px, input/tugma 8px. Balandlik 40px (mobil 44px).
- Shrift: Inter (`next/font`) — antd `fontFamily` ga ham ulanadi. Raqamlar `tabular-nums`.
- `components/AntdProvider.tsx` antd tokenlarini shu qiymatlardan oladi (yagona manba: `lib/theme/tokens.ts`).
- Olib tashlanadi: applicant sahifalaridagi inline `style={{}}` ranglar, `styled-jsx` `!important` override'lar, global `td,th` padding override (admin panel o'zining jadval stilini o'zida saqlashi uchun override admin layout'iga ko'chiriladi).

### 2.2 Layout — `app/(applicant)/layout.tsx`

- Auth guard saqlanadi (public: `/`, `/login`, `/register`, `/register/tsmu-id`, `/forgot-password`).
- **≥1024px:** CSS grid — chap sidebar 248px (yig'iladi → 72px, holat localStorage'da), yupqa header (sahifa nomi, tema, avatar menyu), kontent `max-w-[1200px] mx-auto`.
- **768–1023px:** sidebar 72px icon-rail (tooltip).
- **<768px:** sidebar yo'q; yuqori bar (logo, avatar); **pastki tab-bar** (Profil, Arizalar, Mening arizalarim, Ko'proq → sheet: tema, Telegram yordam, chiqish), `env(safe-area-inset-bottom)`.
- Komponentlar: `components/applicant/shell/{AppShell,Sidebar,Topbar,BottomNav}.tsx`.
- Umumiy: `components/applicant/ui/{PageHeader,Card,StatusBadge,ResponsiveTable,Sheet,VerifiedBadge,EmptyState}.tsx`. `ResponsiveTable` — desktop'da jadval, mobil'da kartalar. Mobil'da modal → pastki sheet.

### 2.3 Sahifalar

1. **`/` Landing** — hero (sarlavha, qisqa matn, CTA: "TSMU ID orqali boshlash", "Kirish"), "Qanday ishlaydi" (3 qadam), footer.
2. **`/login`** — desktop: ikki ustun (chap brend paneli, o'ng forma); mobil: bitta ustun. Telefon (+998 mask) + parol, "Parolni unutdingizmi?", pastda "Akkauntingiz yo'qmi? Ro'yxatdan o'tish".
3. **`/register`** — tanlov ekrani: "TSMU ID orqali (tavsiya)" → `/register/tsmu-id`; "Telefon orqali" → mavjud 3 bosqichli oqim (yangi dizaynda, `?method=phone`).
4. **`/register/tsmu-id` wizard** — `components/applicant/tsmu-id/*`:
   - `StepLookup` — JSHSHIR (14 raqam mask) + tug'ilgan sana (DD.MM.YYYY).
   - `StepFace` — kamera ruxsati izohi → oval ramka → MediaPipe FaceLandmarker yo'naltirish: "To'g'riga qarang" → "Chapga buriling" → "O'ngga buriling", har bosqichda yaw chegarasiga yetganda avtomatik kadr; 3 kadr serverga. Natija: o'tdi / o'tmadi (`attempts_left`, sabab matni). Kamera yo'q / rad etilgan holatlari uchun tushunarli xabar.
   - `StepPhone` — "Siz: M\*\*\*\*\*\*\*\*\* M." tasdig'i, telefon, so'ng 6 katakli OTP (qayta yuborish taymeri).
   - `StepPassword` — parol + tasdiqlash (kuch ko'rsatkichi).
   - `StepDone` — tokenlar saqlanadi → `/dashboard`.
   - Mobil'da har qadam to'liq ekran, yuqorida progress.
   - `@mediapipe/tasks-vision` faqat `StepFace` ichida dynamic import; model fayli CDN'dan.
5. **`/verify-identity`** — (auth) telefon orqali ro'yxatdan o'tganlar uchun: `StepLookup` → `StepFace` → `attach/` → profil.
6. **`/dashboard` Profil** — yuqori karta: rasm, F.I.Sh, telefon, "✓ TSMU ID orqali tasdiqlangan" (yoki sariq banner + "Shaxsni tasdiqlash"). Bo'limlar: Shaxsiy, Pasport, Aloqa va ish joyi. TSMU ID maydonlari qulf ikonkasi bilan read-only; tahrirlash (sheet/modal) faqat email, tashkilot, manzil.
7. **`/applications`** — kartalar grid (1/2/3 ustun): sarlavha, muddat (qolgan kunlar), imtihon sanasi, to'lov summasi, "Ariza berish".
8. **`/applications/[id]`** — 1206 qatorlik fayl bo'linadi: `ApplicantSummaryCard` (snapshot ko'rinishi, read-only), `SpecialitySection`, `EducationFormSection`, `DynamicFieldsSection` (+ `DynamicField` per type), `SubmitBar` (mobil'da sticky). Tasdiqlanmagan user → forma o'rniga `VerifyIdentityGate` (CTA → `/verify-identity?next=...`). Backend `IDENTITY_NOT_VERIFIED` qaytarsa ham shu gate ko'rsatiladi.
9. **`/my-submissions`** — `ResponsiveTable`: raqam, ariza, mutaxassislik, holat, to'lov, sana.
10. **`/my-submissions/[id]`** — status timeline (Qoralama → Yuborildi → Ko'rib chiqilmoqda → Natija), ariza beruvchi snapshot, javoblar, hujjatlar, amallar (tahrirlash, hujjat yuklash, yuborish).
11. **`/forgot-password`** — yangi dizaynda, mantiq o'zgarmaydi.

### 2.4 API qatlami

- `lib/api/tsmuId.ts` — 6 endpoint uchun typed funksiyalar (`apiRequest` / `apiUpload` orqali).
- `types/index.ts` — `VerificationStatus`, `TsmuLookupResponse`, `FaceVerifyResponse`, `ApplicantSnapshot`; `User`ga `is_verified`, `photo_url`.
- Ishlatilmayotgan va eskirgan `lib/api/auth.ts` / `lib/stores/authStore.ts` applicant oqimida ishlatilmaydi (o'chirilmaydi — admin tomonda ishlatilishi tekshiriladi).

## 3. Xatoliklarni boshqarish

| Holat | Foydalanuvchiga |
|---|---|
| docrest topilmadi | "JSHSHIR yoki tug'ilgan sana noto'g'ri" |
| docrest ishlamayapti | "Davlat xizmati vaqtincha javob bermayapti, keyinroq urinib ko'ring" |
| `PINFL_TAKEN` | "Bu JSHSHIR bilan akkaunt mavjud — telefon va parol bilan kiring" + login havolasi |
| Yuz topilmadi / bir nechta yuz | "Kadrda faqat siz bo'lishingiz kerak, yorug' joyda turing" |
| O'xshashlik < 70% | "Yuz mos kelmadi (qolgan urinishlar: N)" |
| Liveness o'tmadi | "Boshingizni ko'rsatilgandek buring" |
| 3 urinish tugadi | "Urinishlar tugadi, 30 daqiqadan keyin qaytadan boshlang" |
| `PHONE_TAKEN` | "Bu raqam boshqa akkauntga bog'langan" |
| Sessiya muddati o'tgan | "Sessiya tugadi" → 1-qadamga qaytarish |
| Rate-limit | "Juda ko'p urinish, keyinroq" |

## 4. Testlash

**Backend (Django TestCase):**
- `docrest.client`: agregat va to'g'ridan-to'g'ri javob, topilmadi, timeout, pasport ajratish (`requests` mock).
- `face.matcher`: `similarity_percent` chegaralari; `verify` — InsightFace `FaceAnalysis` mock bilan: no-face, multi-face, past o'xshashlik, yaw farqi yetarli emas, muvaffaqiyat.
- Endpointlar: to'liq oqim (lookup → face → phone → verify → complete), holat buzilishi (409), urinishlar limiti, `PINFL_TAKEN`, `PHONE_TAKEN`, muddat o'tishi, `attach/` boshqa user sessiyasi bilan rad etilishi.
- `register/complete/` OTP'siz → 400.
- `submissions/create/` tasdiqlanmagan → 403; tasdiqlangan → snapshot yozilgan.
- Profil PATCH: read-only maydonlar o'zgarmaydi.

**Frontend:**
- `npm run build` va `npm run lint` xatosiz.
- Playwright: 375 / 768 / 1280px skrinshotlar — landing, login, register, wizard qadamlari (API mock), dashboard, applications, ariza formasi, my-submissions; gorizontal scroll yo'qligi tekshiriladi (`document.documentElement.scrollWidth <= innerWidth`).

## 5. Bosqichlar (implementatsiya tartibi)

- **A — Dizayn tizimi + responsive qobiq + mavjud sahifalar redesign** (backend o'zgarishsiz ishlaydi).
- **B — TSMU ID:** backend (docrest, face, model, endpointlar, OTP tuzatish) → frontend wizard va `/verify-identity`.
- **C — Ariza integratsiyasi:** `is_verified` gate, snapshot, `ApplicantSummaryCard`, profil read-only.

## Doiradan tashqari

Admin panel redesign; Payme; OneID; `mehnat` / `student_info` / `orta` ma'lumotlaridan foydalanish; selfi kadrlarni saqlash; native mobil ilova; anti-spoofing'ning kuchli turlari (3D/IR, video replay detection).
