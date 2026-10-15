# DOMjudge API Reference — Enclavitzy

> Dokumentasi lengkap integrasi DOMjudge 9.x untuk peserta lomba.
> Semua endpoint **terverifikasi** via `cmd/test-domjudge-api.sh` v5
> (PASS 28, SKIP 9, FAIL 0).
> Sumber kebenaran runtime: `https://<host>/api/doc.json`.

---

## 1. Base URL & Auth

```
https://upcvi.online/api/v4
```

| Mode | Cocok untuk | Cara |
|------|-------------|------|
| **Basic Auth** | Semua `/api/v4/*` | Header `Authorization: Basic <base64(user:pass)>` |
| **Session Cookie** | `/team/*` (samples.zip) | Login form + CSRF + cookie `PHPSESSID` |

**Auth rules:**
- **Basic Auth** = stateless, tidak perlu login form.
- **Session Cookie** = wajib untuk samples.zip. Login form ke `/login` → parse CSRF → POST `_username/_password/_csrf_token` → simpan cookie.

---

## 2. Endpoint yang BISA Diakses Peserta

### 2.1 List Contest
```http
GET /api/v4/contests
```

Response (contoh, terfilter otomatis oleh DOMjudge):
```json
[
  {
    "cid": 4,
    "id": "dj-4",
    "name": "TEX",
    "shortname": "test_exam",
    "start_time": "2026-10-06T20:15:00+07:00",
    "end_time": "2026-10-26T20:15:00+07:00",
    "duration": "480:00:00.000",
    "allow_submit": true,
    "penalty_time": 20
  }
]
```

**ATURAN WAJIB:**
- Pakai field **`id`** (`"dj-4"`) untuk URL. **BUKAN** `cid` numerik (4), **BUKAN** `shortname`.
- **Dynamic Discovery:** Jangan hardcode `contest_id` di file konfigurasi client. Gunakan response endpoint ini saat login: jika hanya 1 kontes yang berstatus aktif/diizinkan, aplikasi otomatis langsung masuk (*auto-enter*); jika terdapat beberapa kontes, tampilkan layar seleksi kontes.
- Cek `allow_submit` sebelum enable form.

### 2.2 Config Contest
```http
GET /api/v4/config
```

Field penting:

| Field | Contoh | Guna |
|-------|--------|------|
| `sourcesize_limit` | `256` | Validasi ukuran file (KB) |
| `sourcefiles_limit` | `100` | Batas jumlah file |
| `clar_categories` | `{general, tech}` | Kategori klarifikasi |
| `allow_team_submission_download` | `false` | **Jangan buat tombol download** |
| `print_command` | `""` | **Print service mati** |
| `show_compile` | `2` | Info saja — API tidak expose output |

### 2.3 List Team
```http
GET /api/v4/contests/{cid}/teams
```

```json
[{
  "id": "tes-3",
  "teamid": 97,
  "name": "tes-03",
  "display_name": "tes-03",
  "affiliation": "Institut Teknologi dan Bisnis Sabda Setia",
  "group_ids": ["participants"]
}]
```

**PENTING:** `team_id` ≠ username. `tes-03` (login) → `tes-3` (team_id). Selalu lookup dari endpoint ini.

### 2.4 List Languages
```http
GET /api/v4/contests/{cid}/languages
```

```json
[{
  "id": "cpp",
  "name": "C++",
  "extensions": ["cpp", "cc", "cxx", "c++"],
  "allow_judge": true
}]
```

**Warning:** Server bisa batasi bahasa per contest. Contoh: `c` tersedia di list tapi `Language 'c' not allowed for contest`.

### 2.5 List Problems
```http
GET /api/v4/contests/{cid}/problems
```

```json
[{
  "ordinal": 0,
  "probid": 1,
  "short_name": "A",
  "label": "A",
  "time_limit": 5.0,
  "id": "hello",
  "name": "Hello World",
  "attachments": [],
  "statement": [{
    "href": "contests/dj-4/problems/hello/statement",
    "mime": "application/pdf",
    "filename": "A.pdf"
  }]
}]
```

**PENTING:**
- `id` (string, misal `"hello"`) → untuk `/problems/{id}`.
- `probid` (numerik, misal `1`) → untuk `/team/{probid}/samples.zip`.
- **Selalu pakai `id`** kalau ada spasi/apostrophe di `name` (misal `Irene's Apple` error kalau pakai `name`).

### 2.6 Detail Problem
```http
GET /api/v4/contests/{cid}/problems/{id}
```

Sama seperti list, tapi satu object. `attachments` bisa `[]` atau `null` — cek panjang sebelum render.

### 2.7 Statement PDF & RAM-Only Ephemeral Caching
```http
GET /api/v4/contests/{cid}/problems/{id}/statement
```

Response: PDF binary (`application/pdf`).

**Aturan Implementasi & Keamanan:**
- **URL slug:** pakai field `id` (bukan `name` atau `label`).
- **RAM-Only Cache (Zero-Disk):** Backend Rust menerima binary stream langsung di memori (dengan timeout 60s untuk PDF besar hingga megabytes), mengonversinya ke Base64 Data URL (`data:application/pdf;base64,...`), dan menyimpannya di `statement_ram_cache: Arc<RwLock<HashMap<String, String>>>`.
- **Dilarang Menulis ke Disk:** Tidak ada file `.pdf` yang ditulis ke hard disk komputer lab demi mencegah kebocoran materi soal setelah kompetisi selesai atau saat PC mati/crash.
- **WebView2 Blob URL Streaming:** Frontend mengonversi Base64 Data URL menjadi ephemeral `blob:` URL (`URL.createObjectURL(blob)`) di memori RAM dan me-rendernya via `<object data="blob:...#toolbar=0">` dengan fallback `<iframe>`. Hal ini mencegah masalah URL length limit Chromium WebView2 yang membuat layar hitam/blank saat memuat PDF berukuran besar (misal 4.8MB). Otomatis dibersihkan dengan `URL.revokeObjectURL` saat ganti soal.
- **Hard Reload (`force_refresh`):** Mendukung parameter `force_refresh: true` pada command Tauri `domjudge_get_statement` untuk bypass RAM cache dan mengambil versi PDF terbaru jika admin lomba merevisi deskripsi soal.
- **CSP WebView:** Membutuhkan `frame-src 'self' data: blob:;` dan `object-src 'self' data: blob:;` di `tauri.conf.json` agar WebView dapat me-render Base64 dan Blob URL secara aman.
- **Scratchpad TMP:** Berkas `scratch_1..3` tidak memanggil endpoint ini karena bukan soal DOMjudge.

### 2.8 Submit Solution
```http
POST /api/v4/contests/{cid}/submissions
Content-Type: multipart/form-data
```

**Fields:**
- `problem_id` — dari `problem.id` (string, misal `"hello"`)
- `language_id` — dari `language.id` (misal `"cpp"`)
- `code` — file upload (wajib `@` di curl)

**Curl:**
```bash
curl -u "user:pass" -X POST \
  -F "problem_id=hello" \
  -F "language_id=cpp" \
  -F "code=@/path/to/solution.cpp" \
  https://upcvi.online/api/v4/contests/dj-4/submissions
```

**Response:**
```json
{
  "id": "855",
  "submitid": 855,
  "problem_id": "hello",
  "language_id": "cpp",
  "team_id": "tes-3",
  "time": "2026-10-07T03:54:03.498+07:00",
  "contest_time": "7:39:03.498"
}
```

### 2.9 List Submissions
```http
GET /api/v4/contests/{cid}/submissions?strict=false
```

**Response:** array of submission objects (metadata saja, verdict TIDAK ada di sini).

**Untuk hitung "sudah berapa try":**
```bash
jq -r 'group_by(.problem_id)[] | "\(.[0].problem_id): \(length)"' <<< "$SUBMISSIONS"
```

**Untuk "submission ke berapa":**
```bash
jq -r 'sort_by(.id) | to_entries[] | "#\(.key + 1) | id=\(.value.id) | \(.value.problem_id)"'
```

### 2.10 Detail Submission
```http
GET /api/v4/contests/{cid}/submissions/{id}
```

Sama seperti list, tapi satu object. **Tidak ada field `judgement_id`** — verdict wajib ambil dari `/judgements`.

### 2.11 Verdict (Judgement)
```http
GET /api/v4/contests/{cid}/judgements?submission_id={submitid}
```

**Response (untuk 1 submission):**
```json
[{
  "id": "874",
  "submission_id": "855",
  "judgement_type_id": "WA",
  "max_run_time": 0.001,
  "valid": true,
  "start_time": "2026-10-07T03:54:07.210+07:00",
  "end_time": "2026-10-07T03:54:08.475+07:00"
}]
```

**Baseline timing (dari test):**
- Verdict muncul dalam **1–6 detik** setelah submit (median ~3s).
- CE lebih cepat (< 2s) karena tidak ada eksekusi.
- Kalau ada rejudge, response berisi multiple entries → ambil **terbaru** (sort by `end_time`).

**Rumus CE, NO-OUTPUT & TOO-LATE:**
```
isCompileError = (verdict === "CE" && max_run_time === null)
isNoOutput = (verdict === "NO-OUTPUT") // program exit 0 tanpa mencetak data ke stdout
isTooLate = (verdict === "TOO-LATE" || (isContestOver && judgements.length === 0))
```
- **Katalog & Normalisasi Verdict Kanonikal Enclavitzy:**
  | Kode DOMjudge / Raw | Kode Kanonikal | Deskripsi UI | Warna UI |
  |---|---|---|---|
  | `AC`, `correct` | `AC` | `CORRECT` (Accepted) | Hijau Emerald |
  | `WA`, `wrong-answer` | `WA` | `WRONG-ANSWER` | Merah Rose |
  | `no-output`, `NO OUTPUT`, `NO-OUTPUT` | `NO-OUTPUT` | `NO-OUTPUT` (Stdout Kosong) | Oranye |
  | `TLE`, `timelimit` | `TLE` | `TIME-LIMIT` | Amber |
  | `RTE`, `run-error`, `RUN-ERROR` | `RTE` / `RUN-ERROR` | `RUN-ERROR` | Ungu Purple |
  | `CE`, `compiler-error` | `CE` | `COMPILER-ERROR` | Cyan |
  | `MLE`, `memory-limit` | `MLE` | `MEMORY-LIMIT` | Fuchsia |
  | `OLE`, `output-limit`, `OUTPUT-LIMIT` | `OLE` / `OUTPUT-LIMIT` | `OUTPUT-LIMIT` | Pink |
  | `PE`, `presentation-error` | `PE` | `PRESENTATION-ERROR` | Kuning |
  | `TOO-LATE` | `TOO-LATE` | `TOO-LATE` (Kontes Berakhir) | Abu-abu Slate |
  | `null` / kosong saat kontes aktif | `PENDING` | `JUDGING / QUEUED` | Biru (Spinning) |
- **Penanganan Submissions Kontes Berakhir (TOO-LATE):** Saat kontes telah berakhir (`now > end_time`), DOMjudge tetap menerima request POST `/submissions` dengan HTTP 200, tetapi tidak memproses penilaian di `/judgements` dan tidak menyertakannya pada filter `/submissions?strict=false`. Sistem klien Enclavitzy mendeteksi kondisi ini secara lokal di RAM session dan secara deterministik menetapkan vonis **`TOO-LATE`**.
- **Pre-Compilation Verification Guard:** Sebelum mengirim berkas ke endpoint `/submissions`, klien melakukan verifikasi kompilasi lokal (`compileOnly`). Jika kompilasi gagal, modal dialog peringatan (*Compile Warning Modal*) ditampilkan agar peserta dapat membatalkan atau mengonfirmasi pengiriman.

### 2.12 Detail Judgement
```http
GET /api/v4/contests/{cid}/judgements/{id}
```

Sama seperti di atas, satu object. **Tidak ada field `compile_output`** — konfirmasi dari test.

### 2.13 Clarifications (list + send)
```http
GET  /api/v4/contests/{cid}/clarifications
POST /api/v4/contests/{cid}/clarifications
```

**POST body (form-data):**
- `text` — isi klarifikasi
- `problem_id` — optional (kosong = general)

**Response example (thread lengkap dengan balasan jury):**
```json
[
  {
    "id": "3",
    "time": "2026-10-04T08:46:25.045+07:00",
    "contest_time": "19:41:25.045",
    "problem_id": "irenes-apple",
    "from_team_id": "tes-3",
    "to_team_id": null,
    "reply_to_id": null,
    "text": "ini apa bang?"
  },
  {
    "id": "4",
    "time": "2026-10-04T08:46:41.051+07:00",
    "problem_id": "irenes-apple",
    "from_team_id": null,
    "to_team_id": "tes-3",
    "reply_to_id": "3",
    "text": "> ini apa bang?\r\n\r\nRead the problem statement carefully."
  }
]
```

**Field semantics:**

| Field | Nilai | Arti |
|-------|-------|------|
| `from_team_id` | `"tes-3"` | Peserta kirim |
| `from_team_id` + `to_team_id` | `null` + `"tes-3"` | **Jury reply** |
| `reply_to_id` | `null` | Root klarifikasi |
| `reply_to_id` | `"3"` | Balasan untuk id=3 |

**Privacy (VERIFIED):** Peserta hanya lihat clar yang melibatkan dirinya. Filter di client:
```typescript
const myClars = all.filter(c =>
  c.from_team_id === myTeamId || c.to_team_id === myTeamId
);
```

**Rendering note:**
- Text pakai markdown blockquote (`> ` di awal baris).
- Line break `\r\n` → normalize ke `\n`.
- Ada trailing `>` di akhir → trim.

**Normalisasi:**
```typescript
function normalizeClarText(text: string): string {
  return text
    .replace(/\r\n/g, '\n')
    .replace(/\n>\s*$/g, '')
    .trim();
}
```

### 2.14 Scoreboard
```http
GET /api/v4/contests/{cid}/scoreboard
GET /api/v4/contests/{cid}/scoreboard?public=true
```

```json
{
  "event_id": "11493",
  "time": "2026-10-07T03:54:10.097+07:00",
  "contest_time": "7:39:10.096",
  "state": {
    "started": "2026-10-06T20:15:00.000+07:00",
    "ended": null,
    "frozen": null,
    "thawed": null,
    "finalized": null,
    "end_of_updates": null
  },
  "rows": [{
    "rank": 1,
    "team_id": "tes-3",
    "score": { "num_solved": 0, "total_time": 0 },
    "problems": [{
      "label": "A",
      "problem_id": "hello",
      "num_judged": 12,
      "num_pending": 0,
      "solved": false,
      "first_to_solve": false
    }]
  }]
}
```

**Field `state` adalah pengganti `/state`** yang 403 untuk peserta.

**Interpretasi `state`:**
- `started != null && ended == null` → contest sedang berjalan
- `ended != null` → contest berakhir
- `frozen != null && thawed == null` → scoreboard frozen (peserta tidak lihat update)
- `finalized != null` → scoreboard final

**Field per-problem berguna untuk UI:**
- `num_judged` — jumlah try (termasuk AC)
- `num_pending` — submission belum dijudge
- `solved` — AC atau belum

---

## 3. Endpoint yang TIDAK Bisa Diakses Peserta

| Endpoint | Status | Alasan |
|----------|--------|--------|
| `GET /contests/{cid}/state` | **403** | Role peserta tidak diizinkan |
| `GET /contests/{cid}/runs` | **403** | Jury/Judgehost only |
| `GET /contests/{cid}/event-feed` | **403** | Jury/API reader only |
| `GET /contests/{cid}/attachments` | **404** | Route tidak ada |
| `GET /contests/{cid}/balloons` | 403 | Jury only |
| `GET /contests/{cid}/judgements/{id}/runs` | 404 | Path deprecated (v9 pakai `/runs`) |
| `POST /contests/{cid}/print` | N/A | `print_command: ""` |

**Error 403 untuk peserta:**
```
Access Denied by #[IsGranted(new Expression("is_granted('ROLE_JURY')
  or is_granted('ROLE_JUDGEHOST')
  or is_granted('ROLE_API_READER')"))]
```

---

## 4. Samples Test Case (Scraping via Session Cookie)

**API tidak menyediakan samples.** Samples hanya via web UI:
```
GET https://upcvi.online/team/{probid}/samples.zip
```

**Auth:** Session cookie (`PHPSESSID`), bukan Basic Auth.

**Prosedur (verified v5):**
1. `GET /login` → parse CSRF token dari HTML
2. `POST /login` dengan `_username`, `_password`, `_csrf_token`
3. Simpan cookie `PHPSESSID`
4. `GET /team/{probid}/samples.zip` dengan cookie
5. Parse ZIP

**Format ZIP:**
```
samples_{probid}.zip
├── 1.in      ← input test case 1
├── 1.ans     ← expected output test case 1
├── 2.in
└── 2.ans
```

**Behavior per problem:**
- Tidak semua problem punya samples. Contoh: `hello` (probid=1) 404, `irenes-apple` (probid=4) 200.
- Cek `Content-Type` sebelum simpan: `application/zip` vs `text/html` (404 page).
- Ukuran valid: ~393 B untuk 2 test case.

**Aturan UI:**
- Samples ≠ test case lengkap. Lolos samples ≠ AC. Beri disclaimer.
- Setiap `.ans` punya trailing newline `\n` → **wajib `trim()`** saat compare.
- Normalize `\r\n` → `\n`, trailing space per baris.

---

## 5. Verdict Mapping

| Verdict | Arti | `max_run_time` | Warna UI |
|---------|------|----------------|----------|
| `AC` | Accepted | > 0 | 🟢 Hijau |
| `WA` | Wrong Answer | > 0 | 🔴 Merah |
| `TLE` | Time Limit Exceeded | besar | 🟠 Orange |
| `MLE` | Memory Limit Exceeded | > 0 | 🟣 Magenta |
| `RTE` | Runtime Error | > 0 | 🟪 Ungu |
| **`CE`** | **Compile Error** | **`null`** | ⚫ Abu |
| `OLE` | Output Limit Exceeded | > 0 | 🟡 Kuning |
| `PE` | Presentation Error | > 0 | 🟡 Kuning |
| **`TOO-LATE`** | **Terkirim Setelah Kontes Berakhir** | **`null`** | ⚪ Abu Gelap / Netral |
| `null` | Belum dijudge | `null` | 🔵 Biru |

**Rumus CE:** `isCompileError = (verdict === "CE" && max_run_time === null)`  
**Rumus TOO-LATE:** `isTooLate = (now > contest.end_time || (sub.time > contest.end_time && judgements.is_empty()))`

### 5.1 Pre-Compilation Guard (Local Verification Before Submit)
Sebelum solusi dikirim ke endpoint `/submissions`, sistem menjalankan validasi kompilasi lokal via command Rust native MinGW:
1. `api.compileOnly(activeProblemId, code)`
2. Jika lolos (`success: true`) $\to$ langsung kirim ke DOMjudge via multipart form data.
3. Jika gagal (`success: false`) $\to$ sistem menampilkan modal peringatan interaktif (`CompileWarningModal`) dengan diagnostik error dari compiler `g++`. Peserta diberi 2 opsi:
   - **Batal Kirim (atau tombol X/Close)**: Pengiriman dibatalkan sepenuhnya untuk memperbaiki kode lokal.
   - **Tetap Lanjutkan Kirim**: Mengabaikan kesalahan kompilasi dan tetap mengirimkan berkas solusi ke server DOMjudge.

---

## 6. Polling Strategy

| Resource | Interval | Endpoint |
|----------|----------|----------|
| Submissions | 3s | `GET /contests/{cid}/submissions?strict=false` |
| Judgements | 3s | `GET /contests/{cid}/judgements?submission_id={id}` |
| Clarifications | 3s | `GET /contests/{cid}/clarifications` |
| Scoreboard | 10s | `GET /contests/{cid}/scoreboard` |

**Jangan < 3s.** Server rate-limit.

**Verdict polling:** awal 1s × 15, lanjut 3s × 10 (max ~45s total). Verdict biasanya muncul < 6s.

---

## 7. Server Time Sync

Response HTTP menyertakan `Date` header:
```
date: Tue, 06 Oct 2026 20:54:01 GMT
```

**TypeScript:**
```typescript
const serverTime = new Date(response.headers.get('date'));
const offsetMs = serverTime.getTime() - Date.now();
// Get server now: new Date(Date.now() + offsetMs)
```

---

## 8. Config Reference (dari `/config`)

| Field | Contoh | Guna UI |
|-------|--------|---------|
| `sourcesize_limit` | `256` | Validasi file (KB) |
| `sourcefiles_limit` | `100` | Batas multi-file |
| `penalty_time` | `20` | Menit penalty per WA |
| `compile_penalty` | `1` | Penalty CE |
| `clar_categories` | `{general, tech}` | Dropdown klarifikasi |
| `show_pending` | `true` | Tampilkan submission pending |
| `show_affiliations` | `true` | Asal institusi di scoreboard |
| `allow_team_submission_download` | `false` | **Jangan buat tombol download** |
| `show_compile` | `2` | Info saja — API tidak expose output |
| `print_command` | `""` | Print service mati |

---

## 9. Known Limitations

1. **Compile output tidak via API** — meski `show_compile: 2`. Fallback ke web UI `/team/submission/{id}`.
2. **Tidak bisa download source sendiri** — `allow_team_submission_download: false`.
3. **Tidak ada per-test-case runs** — 403 untuk peserta.
4. **Tidak ada real-time streaming** — event feed 403, wajib polling.
5. **Samples hanya via scraping** — bukan REST API resmi, rapuh.
6. **`/state` endpoint 403** — ambil dari `.state` di `/scoreboard`.
7. **Attachments kosong** di semua problem (tested).

---

## 10. Error Handling

| Status | Aksi |
|--------|------|
| `200` | Sukses |
| `201` | Submit sukses (jarang, biasanya 200) |
| `400` | Cek `problem_id` / `language_id` / format file |
| `401` | Kredensial salah / session expired → prompt login ulang |
| `403` | Role tidak diizinkan → fitur tidak tersedia |
| `404` | Resource tidak ada → cek `cid`, `problem_id`, `submission_id` |
| `500` | Server error → retry dengan backoff, tampilkan notif |

**Bahasa tidak diizinkan:** DOMjudge bisa return 400 `Language 'X' not allowed for contest [cid]`. Handle gracefully.

---

## 11. Sample Code

### 11.1 Login + List Contest (TypeScript)

```typescript
const HOST = 'https://upcvi.online';
const auth = btoa(`${username}:${password}`);

const contests = await fetch(`${HOST}/api/v4/contests`, {
  headers: { 'Authorization': `Basic ${auth}` },
}).then(r => r.json());

// Setiap contest: pakai field `id` untuk URL berikutnya
const activeContest = contests.find(c => c.allow_submit);
```

### 11.2 Submit Solution

```typescript
const form = new FormData();
form.append('problem_id', 'hello');
form.append('language_id', 'cpp');
form.append('code', new Blob([sourceCode], { type: 'text/x-c++src' }), 'main.cpp');

const submission = await fetch(
  `${HOST}/api/v4/contests/${contestId}/submissions`,
  {
    method: 'POST',
    headers: { 'Authorization': `Basic ${auth}` },
    body: form,
  }
).then(r => r.json());

const submitId = submission.id;
```

### 11.3 Poll Verdict

```typescript
async function pollVerdict(submitId: string): Promise<Judgement | null> {
  for (let attempt = 0; attempt < 25; attempt++) {
    const delay = attempt < 15 ? 1000 : 3000;
    await new Promise(r => setTimeout(r, delay));

    const judgements = await fetch(
      `${HOST}/api/v4/contests/${contestId}/judgements?submission_id=${submitId}`,
      { headers: { 'Authorization': `Basic ${auth}` } }
    ).then(r => r.json());

    if (judgements.length > 0) {
      // Ambil terbaru kalau ada rejudge
      return judgements.sort((a, b) =>
        new Date(a.end_time).getTime() - new Date(b.end_time).getTime()
      ).at(-1) ?? null;
    }
  }
  return null;
}
```

### 11.4 Parse Clar Thread untuk UI

```typescript
interface ClarNode extends Clarification {
  isMine: boolean;
  isJury: boolean;
  children: ClarNode[];
}

function buildClarThread(clars: Clarification[], myTeamId: string): ClarNode[] {
  const nodes = new Map<string, ClarNode>();

  for (const c of clars) {
    nodes.set(c.id, {
      ...c,
      isMine: c.from_team_id === myTeamId,
      isJury: c.from_team_id === null && c.to_team_id === myTeamId,
      children: [],
    });
  }

  const roots: ClarNode[] = [];
  for (const node of nodes.values()) {
    if (node.reply_to_id && nodes.has(node.reply_to_id)) {
      nodes.get(node.reply_to_id)!.children.push(node);
    } else {
      roots.push(node);
    }
  }

  return roots.sort((a, b) => parseInt(a.id) - parseInt(b.id));
}
```

### 11.5 Hitung Submission Count per Problem

```typescript
function countSubmissionsByProblem(subs: Submission[]) {
  return subs.reduce<Record<string, number>>((acc, s) => {
    acc[s.problem_id] = (acc[s.problem_id] ?? 0) + 1;
    return acc;
  }, {});
}
```

---

## 12. Changelog

- **2026-10-07 v5.1** — Patch section 30 (jq error), docs lengkap dengan:
  - Clar thread structure (verifikasi 4 klarifikasi thread)
  - Clar privacy (verified — tidak ada leak)
  - Submission grouping + timeline
  - Attachments per problem (kosong di semua)
  - Sample code lengkap TypeScript
- **2026-10-07 v5.0** — Tambah test clar thread, privacy, attachments, grouping, timeline.
- **2026-10-07 v4.0** — Loop samples semua problem, save artifacts, exponential backoff.
- **2026-10-06 v3.0** — Fix `/runs` path, samples zip, CE test.