# Supabase 갤러리 + 방명록 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Supabase Storage에서 갤러리 이미지/영상을 동적으로 불러오고, Supabase DB를 이용한 방명록 섹션을 추가한다.

**Architecture:** 정적 HTML 파일에 Supabase JS 클라이언트(CDN)를 로드하여 직접 호출한다. 갤러리는 `gallery_items` 테이블(메타데이터) + `gallery` Storage 버킷(실제 파일) 조합으로 구성하고, 방명록은 `guestbook` 테이블에 직접 읽기/쓰기한다. 모든 Supabase 접근은 RLS(Row Level Security)로 보호한다.

**Tech Stack:** Supabase JS v2 (CDN UMD), Supabase Storage, Supabase PostgreSQL, Vercel (static hosting)

> **⚠️ anon key 노출:** Supabase anon key는 클라이언트 사이드용으로 설계된 공개 키이며, RLS가 실제 보안을 담당한다. HTML에 직접 포함해도 안전하다.

---

## 파일 구조

| 파일 | 변경 내용 |
|---|---|
| `public/index.html` | Supabase CDN 추가, 갤러리 fetch 로직, 방명록 섹션 HTML/CSS/JS |

> 모든 변경은 단일 파일(`public/index.html`)에만 이루어진다.

---

## Task 1: Supabase 데이터베이스 테이블 생성

> **Supabase Dashboard → SQL Editor** 에서 실행

**Files:**
- 없음 (Supabase Dashboard에서 직접 실행)

- [ ] **Step 1: Supabase 대시보드 접속 → SQL Editor 열기**

  https://supabase.com/dashboard → 프로젝트 선택 → 왼쪽 메뉴 "SQL Editor"

- [ ] **Step 2: gallery_items 테이블 생성 SQL 실행**

  ```sql
  create table gallery_items (
    id uuid default gen_random_uuid() primary key,
    file_path text not null,
    type text not null default 'image' check (type in ('image', 'video')),
    caption text,
    poster_path text,
    sort_order integer not null default 0,
    created_at timestamptz default now()
  );

  alter table gallery_items enable row level security;

  create policy "gallery public read"
    on gallery_items for select
    using (true);

  create policy "gallery auth insert"
    on gallery_items for insert
    to authenticated
    with check (true);

  create policy "gallery auth update"
    on gallery_items for update
    to authenticated
    using (true);

  create policy "gallery auth delete"
    on gallery_items for delete
    to authenticated
    using (true);
  ```

- [ ] **Step 3: guestbook 테이블 생성 SQL 실행**

  ```sql
  create table guestbook (
    id uuid default gen_random_uuid() primary key,
    name text not null,
    message text not null,
    created_at timestamptz default now()
  );

  alter table guestbook enable row level security;

  create policy "guestbook public read"
    on guestbook for select
    using (true);

  create policy "guestbook public insert"
    on guestbook for insert
    with check (
      length(trim(name)) > 0
      and length(name) <= 50
      and length(trim(message)) > 0
      and length(message) <= 500
    );
  ```

- [ ] **Step 4: 테이블 생성 확인**

  SQL Editor에서 실행:
  ```sql
  select table_name from information_schema.tables
  where table_schema = 'public'
  and table_name in ('gallery_items', 'guestbook');
  ```
  
  기대 결과: `gallery_items`, `guestbook` 두 행이 반환됨

---

## Task 2: Supabase Storage 버킷 생성

> **Supabase Dashboard → Storage** 에서 실행

**Files:**
- 없음 (Supabase Dashboard에서 직접 실행)

- [ ] **Step 1: Storage 버킷 생성**

  왼쪽 메뉴 "Storage" → "New bucket" 클릭
  - Bucket name: `gallery`
  - Public bucket: **ON** (체크)
  - "Save" 클릭

- [ ] **Step 2: 버킷 정책 확인**

  `gallery` 버킷 선택 → "Policies" 탭 → Public bucket이면 기본적으로 SELECT 허용됨.
  
  아래 SQL을 SQL Editor에서 추가로 실행하여 인증된 사용자만 업로드 가능하도록 설정:
  ```sql
  create policy "gallery storage public read"
    on storage.objects for select
    using (bucket_id = 'gallery');

  create policy "gallery storage auth upload"
    on storage.objects for insert
    to authenticated
    with check (bucket_id = 'gallery');
  ```

- [ ] **Step 3: 테스트 이미지 업로드**

  Supabase Dashboard → Storage → `gallery` 버킷 → "Upload file" 로 테스트 이미지 1장 업로드
  
  업로드 후 파일 클릭 → "Get URL" → 브라우저에서 URL 접속하여 이미지가 보이는지 확인

- [ ] **Step 4: gallery_items 테이블에 테스트 행 삽입**

  SQL Editor에서 실행 (`파일명`을 실제 업로드한 파일명으로 교체):
  ```sql
  insert into gallery_items (file_path, type, caption, sort_order)
  values ('파일명.jpg', 'image', '처음 같이 뛰어들던 날', 1);
  ```

---

## Task 3: Supabase 클라이언트 HTML에 추가

**Files:**
- Modify: `public/index.html`

- [ ] **Step 1: `<head>` 안에 Supabase CDN 스크립트 추가**

  `</style>` 바로 위에 추가:
  ```html
  <script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/dist/umd/supabase.js"></script>
  ```

- [ ] **Step 2: Supabase 클라이언트 초기화 코드 추가**

  `<script>` 태그 (기존 GALLERY_ITEMS 배열이 있는 곳) 바로 위에 새 `<script>` 블록 추가:
  ```html
  <script>
  // ===== Supabase 설정 =====
  // Supabase Dashboard → Project Settings → API 에서 값 확인
  const SUPABASE_URL = 'https://여기에_프로젝트_URL.supabase.co';
  const SUPABASE_ANON_KEY = '여기에_anon_public_key';
  const db = window.supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);
  </script>
  ```

- [ ] **Step 3: 실제 값 입력**

  Supabase Dashboard → Project Settings → API 에서:
  - `Project URL` → `SUPABASE_URL` 값
  - `Project API Keys` → `anon public` → `SUPABASE_ANON_KEY` 값

- [ ] **Step 4: 브라우저 콘솔에서 연결 확인**

  개발 서버 실행 후 브라우저 콘솔에서:
  ```js
  db.from('guestbook').select('count').then(console.log)
  ```
  기대: `{ data: [{count: '0'}], error: null }` (에러 없음)

---

## Task 4: 갤러리 - Supabase에서 동적 로드

**Files:**
- Modify: `public/index.html` (기존 `GALLERY_ITEMS` 배열과 `renderGallery` 함수 교체)

- [ ] **Step 1: 기존 정적 `GALLERY_ITEMS` 배열 제거하고 fetch 함수로 교체**

  기존:
  ```js
  const GALLERY_ITEMS = [
    { type: 'image', src: '', caption: '처음 같이 뛰어들던 날' },
    { type: 'image', src: '', caption: '버디의 곁에서' },
    { type: 'image', src: '', caption: '수면 위로, 함께' },
  ];
  ```

  교체 (GALLERY_ITEMS 배열 전체를 아래로 대체):
  ```js
  let GALLERY_ITEMS = [];

  async function fetchGalleryItems() {
    const { data, error } = await db
      .from('gallery_items')
      .select('file_path, type, caption, poster_path, sort_order')
      .order('sort_order', { ascending: true });
    if (error) { console.error('gallery fetch error:', error); return []; }
    return (data || []).map(row => ({
      type: row.type,
      src: `${SUPABASE_URL}/storage/v1/object/public/gallery/${row.file_path}`,
      poster: row.poster_path
        ? `${SUPABASE_URL}/storage/v1/object/public/gallery/${row.poster_path}`
        : '',
      caption: row.caption || '',
    }));
  }
  ```

- [ ] **Step 2: `renderGallery` 함수 내 로딩 상태 추가**

  기존 `renderGallery` 함수를 찾아 아래로 교체:
  ```js
  function renderGallery() {
    const grid = document.getElementById('galleryGrid');
    if (!grid) return;
    grid.innerHTML = '';
    if (GALLERY_ITEMS.length === 0) {
      grid.innerHTML = '<p style="color:rgba(255,255,255,.3);font-size:12px;grid-column:1/-1;text-align:center;padding:40px 0">사진을 불러오는 중...</p>';
      return;
    }
    const ph = `<svg viewBox="0 0 40 40" fill="none"><rect x="4" y="8" width="32" height="24" rx="3" stroke="rgba(255,255,255,.35)" stroke-width="1.5"/><circle cx="14" cy="18" r="4" stroke="rgba(255,255,255,.35)" stroke-width="1.5"/><path d="M4 26l8-6 7 5 5-4 12 9" stroke="rgba(255,255,255,.35)" stroke-width="1.5" stroke-linejoin="round"/></svg>`;
    GALLERY_ITEMS.forEach((item, i) => {
      const div = document.createElement('div');
      div.className = 'gallery-item';
      if (!item.src) {
        div.innerHTML = `<div class="ph">${ph}<span>${item.caption || ''}</span></div>`;
      } else if (item.type === 'video') {
        div.innerHTML = `<video src="${item.src}" poster="${item.poster || ''}" preload="none" muted playsinline></video><div class="play-btn">▶</div>`;
      } else {
        div.innerHTML = `<img src="${item.src}" alt="${item.caption || ''}" loading="lazy">`;
      }
      if (item.src) div.addEventListener('click', () => openLightbox(i));
      grid.appendChild(div);
    });
  }
  ```

- [ ] **Step 3: `DOMContentLoaded` 핸들러 안에서 fetch 후 render 호출하도록 수정**

  기존 `DOMContentLoaded` 핸들러 안의 `renderGallery();` 를 아래로 교체:
  ```js
  fetchGalleryItems().then(items => {
    GALLERY_ITEMS = items;
    renderGallery();
  });
  ```

- [ ] **Step 4: 동작 확인**

  브라우저에서 갤러리 섹션으로 스크롤 → Task 2 Step 3에서 업로드한 테스트 이미지가 표시되는지 확인.
  
  브라우저 콘솔에 에러 없어야 함.

---

## Task 5: 방명록 섹션 HTML + CSS

**Files:**
- Modify: `public/index.html`

- [ ] **Step 1: 방명록 섹션 HTML 추가**

  기존 `<!-- Lightbox -->` 주석 바로 위에 삽입:
  ```html
  <!-- 방명록 -->
  <section class="guestbook-section" id="guestbookSection">
    <div class="gb-inner">
      <div class="gb-header">
        <div class="gb-label">Messages</div>
        <h2 class="gb-title">축하 메시지</h2>
        <p class="gb-subtitle">두 사람의 새로운 다이빙을 응원해 주세요</p>
      </div>

      <div class="gb-messages" id="gbMessages">
        <p class="gb-loading">불러오는 중...</p>
      </div>

      <form class="gb-form" id="gbForm">
        <input
          class="gb-input"
          id="gbName"
          type="text"
          placeholder="이름"
          maxlength="50"
          autocomplete="off"
        >
        <textarea
          class="gb-textarea"
          id="gbMessage"
          placeholder="축하 메시지를 남겨주세요 (최대 500자)"
          maxlength="500"
          rows="4"
        ></textarea>
        <button class="gb-submit" id="gbSubmit" type="submit">남기기</button>
      </form>
    </div>
  </section>
  ```

- [ ] **Step 2: 방명록 CSS 추가**

  `/* Lightbox */` 주석 바로 위에 추가:
  ```css
  /* Guestbook */
  .guestbook-section {
    position: relative; z-index: 20;
    background: linear-gradient(180deg, #FFF8EE 0%, #FFF2DC 100%);
    min-height: 100vh; padding: 64px 24px 80px;
  }
  .gb-inner { max-width: 400px; margin: 0 auto; }
  .gb-header { text-align: center; margin-bottom: 40px; }
  .gb-label {
    font-family: 'Cormorant Garamond', serif; font-size: 11px;
    letter-spacing: 5px; text-transform: uppercase;
    color: rgba(90,62,40,.4); margin-bottom: 12px;
  }
  .gb-title { font-size: 24px; font-weight: 300; color: #3A2A1A; letter-spacing: 2px; margin-bottom: 10px; }
  .gb-subtitle { font-size: 13px; color: rgba(58,42,26,.5); line-height: 1.8; }

  .gb-messages { margin-bottom: 40px; display: flex; flex-direction: column; gap: 16px; }
  .gb-loading { font-size: 13px; color: rgba(58,42,26,.4); text-align: center; padding: 24px 0; }
  .gb-empty { font-size: 13px; color: rgba(58,42,26,.35); text-align: center; padding: 40px 0; line-height: 2; }
  .gb-card {
    background: #fff; border-radius: 16px;
    padding: 20px; box-shadow: 0 2px 12px rgba(90,62,40,.08);
  }
  .gb-card-top { display: flex; justify-content: space-between; align-items: baseline; margin-bottom: 10px; }
  .gb-card-name { font-size: 14px; font-weight: 400; color: #3A2A1A; }
  .gb-card-date { font-size: 11px; color: rgba(58,42,26,.35); }
  .gb-card-msg { font-size: 13px; color: rgba(58,42,26,.7); line-height: 1.8; white-space: pre-wrap; }

  .gb-form { display: flex; flex-direction: column; gap: 12px; }
  .gb-input, .gb-textarea {
    width: 100%; padding: 14px 16px; border-radius: 12px;
    border: 1px solid rgba(90,62,40,.15); background: #fff;
    font-family: 'Noto Serif KR', serif; font-size: 14px; color: #3A2A1A;
    outline: none; resize: none;
  }
  .gb-input:focus, .gb-textarea:focus { border-color: rgba(90,62,40,.4); }
  .gb-input::placeholder, .gb-textarea::placeholder { color: rgba(58,42,26,.35); }
  .gb-submit {
    padding: 16px; border-radius: 50px; border: none;
    background: #3A2A1A; color: #FFF8EE;
    font-family: 'Noto Serif KR', serif; font-size: 14px;
    letter-spacing: 2px; cursor: pointer; transition: background .2s;
  }
  .gb-submit:hover { background: #5A3E28; }
  .gb-submit:disabled { background: rgba(58,42,26,.3); cursor: not-allowed; }
  ```

- [ ] **Step 3: 브라우저에서 레이아웃 확인**

  페이지를 끝까지 스크롤 → 방명록 섹션이 크림색 배경으로 고정 패널 위를 덮으면서 나타나는지 확인.
  폼이 깔끔하게 표시되는지 확인.

---

## Task 6: 방명록 - 메시지 읽기

**Files:**
- Modify: `public/index.html`

- [ ] **Step 1: 방명록 fetch 함수 추가**

  기존 `document.addEventListener('DOMContentLoaded', ...` 이벤트 핸들러 바로 위에 추가:
  ```js
  // ===== 방명록 =====
  function formatDate(iso) {
    const d = new Date(iso);
    return `${d.getFullYear()}.${String(d.getMonth()+1).padStart(2,'0')}.${String(d.getDate()).padStart(2,'0')}`;
  }

  async function loadGuestbook() {
    const el = document.getElementById('gbMessages');
    if (!el) return;
    const { data, error } = await db
      .from('guestbook')
      .select('name, message, created_at')
      .order('created_at', { ascending: false })
      .limit(100);
    if (error) { el.innerHTML = '<p class="gb-empty">메시지를 불러올 수 없습니다.</p>'; return; }
    if (!data || data.length === 0) {
      el.innerHTML = '<p class="gb-empty">아직 메시지가 없어요<br>첫 번째 축하를 남겨주세요</p>';
      return;
    }
    el.innerHTML = data.map(row => `
      <div class="gb-card">
        <div class="gb-card-top">
          <span class="gb-card-name">${escapeHtml(row.name)}</span>
          <span class="gb-card-date">${formatDate(row.created_at)}</span>
        </div>
        <p class="gb-card-msg">${escapeHtml(row.message)}</p>
      </div>
    `).join('');
  }

  function escapeHtml(str) {
    return str.replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;').replace(/"/g,'&quot;');
  }
  ```

- [ ] **Step 2: `DOMContentLoaded` 핸들러에 `loadGuestbook()` 호출 추가**

  기존 핸들러 내부 마지막 줄(닫는 `});` 직전)에 추가:
  ```js
  loadGuestbook();
  ```

- [ ] **Step 3: 동작 확인**

  페이지를 끝까지 스크롤 → 방명록 섹션에 "아직 메시지가 없어요" 가 표시되는지 확인.
  
  SQL Editor에서 테스트 메시지 삽입 후 새로고침:
  ```sql
  insert into guestbook (name, message) values ('테스트', '결혼 축하해요!');
  ```
  → 방명록에 메시지 카드가 표시되는지 확인.

---

## Task 7: 방명록 - 메시지 작성

**Files:**
- Modify: `public/index.html`

- [ ] **Step 1: 폼 submit 핸들러 추가**

  `loadGuestbook` 함수 바로 다음에 추가:
  ```js
  async function submitGuestbook(name, message) {
    const { error } = await db
      .from('guestbook')
      .insert({ name: name.trim(), message: message.trim() });
    if (error) throw error;
  }
  ```

- [ ] **Step 2: DOMContentLoaded 핸들러에 폼 이벤트 연결 추가**

  `loadGuestbook();` 바로 다음 줄에 추가:
  ```js
  const gbForm = document.getElementById('gbForm');
  if (gbForm) {
    gbForm.addEventListener('submit', async e => {
      e.preventDefault();
      const nameEl = document.getElementById('gbName');
      const msgEl  = document.getElementById('gbMessage');
      const btn    = document.getElementById('gbSubmit');
      const name    = nameEl.value.trim();
      const message = msgEl.value.trim();
      if (!name)    { nameEl.focus(); return; }
      if (!message) { msgEl.focus();  return; }
      btn.disabled = true;
      btn.textContent = '저장 중...';
      try {
        await submitGuestbook(name, message);
        nameEl.value = '';
        msgEl.value  = '';
        await loadGuestbook();
        document.getElementById('gbMessages').scrollIntoView({ behavior: 'smooth' });
      } catch (err) {
        console.error('guestbook submit error:', err);
        alert('메시지 저장에 실패했습니다. 잠시 후 다시 시도해 주세요.');
      } finally {
        btn.disabled = false;
        btn.textContent = '남기기';
      }
    });
  }
  ```

- [ ] **Step 3: 전체 동작 확인**

  1. 페이지 끝까지 스크롤 → 방명록 섹션 확인
  2. 이름 + 메시지 입력 → "남기기" 클릭
  3. 버튼이 "저장 중..."으로 바뀌었다가 완료 후 복귀
  4. 새 메시지 카드가 목록 상단에 추가됨
  5. 이름 없이 제출 시 이름 필드로 포커스 이동
  6. 메시지 없이 제출 시 메시지 필드로 포커스 이동

- [ ] **Step 4: 글자 수 초과 테스트**

  이름 50자 초과 or 메시지 500자 초과로 Supabase 직접 insert 시도 → RLS가 막는지 확인:
  ```sql
  insert into guestbook (name, message)
  values (repeat('a', 51), 'test');
  -- 기대: ERROR: new row violates row-level security policy
  ```

---

## 갤러리에 파일 추가하는 방법 (참고)

이미지/영상을 추가할 때마다:
1. Supabase Dashboard → Storage → `gallery` 버킷 → 파일 업로드
2. SQL Editor에서 메타데이터 삽입:

```sql
-- 이미지 추가
insert into gallery_items (file_path, type, caption, sort_order)
values ('파일명.jpg', 'image', '사진 설명', 10);

-- 영상 추가 (poster는 썸네일 이미지 파일명)
insert into gallery_items (file_path, type, caption, poster_path, sort_order)
values ('영상.mp4', 'video', '영상 설명', '썸네일.jpg', 20);
```

`sort_order` 값이 작을수록 갤러리 앞에 표시됨.
