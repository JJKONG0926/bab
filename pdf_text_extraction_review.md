# 오프라인 단일 HTML/CSS/JS 환경에서 PDF 텍스트 추출 페이지 구현 검토

## 결론
가능합니다. 인터넷 없이도 **로컬에 있는 `pdf.js` ESM(`.mjs`) 파일**을 직접 import하면, 브라우저에서 PDF를 읽고 각 페이지의 텍스트 레이어(정확히는 `getTextContent()` 결과)를 수집해 문자열로 출력할 수 있습니다.

---

## 전제 조건
1. 브라우저가 ES Modules를 지원해야 합니다.
2. 로컬 정적 파일로 실행할 때 CORS/Worker 이슈가 있을 수 있으므로, 가능하면 간단한 로컬 서버(`python -m http.server`)로 열어야 안정적입니다.
3. `pdf.js` 배포본에서 보통 아래 파일이 필요합니다.
   - `pdf.mjs` (메인)
   - `pdf.worker.mjs` (워커)

---

## 구현 포인트

### 1) 단일 파일 구조
- `index.html` 하나에 CSS/JS를 모두 포함합니다.
- `<script type="module">` 안에서 `pdf.mjs`를 import 합니다.

### 2) Worker 경로 설정
`pdf.js`는 worker를 별도 파일로 사용합니다. 반드시 아래처럼 경로를 지정합니다.

```js
pdfjsLib.GlobalWorkerOptions.workerSrc = './pdf.worker.mjs';
```

### 3) 파일 업로드 후 텍스트 추출
- `<input type="file" accept="application/pdf">`로 업로드
- `arrayBuffer()`로 읽기
- `getDocument({ data })`로 문서 로드
- 페이지별 `page.getTextContent()` 실행
- `items[].str`를 합쳐 출력

---

## 단일 파일 예시 (오프라인 대응)
> 아래 예시는 HTML 단일 파일입니다. 단, 같은 폴더에 `pdf.mjs`, `pdf.worker.mjs`가 있어야 동작합니다.

```html
<!doctype html>
<html lang="ko">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>PDF 텍스트 추출기</title>
  <style>
    body { font-family: system-ui, sans-serif; margin: 24px; }
    .row { display: flex; gap: 12px; align-items: center; flex-wrap: wrap; }
    button { padding: 8px 12px; cursor: pointer; }
    textarea { width: 100%; height: 50vh; margin-top: 16px; }
    .hint { color: #555; font-size: 14px; margin-top: 8px; }
    .error { color: #b00020; white-space: pre-wrap; }
  </style>
</head>
<body>
  <h1>PDF 텍스트 추출기</h1>

  <div class="row">
    <input id="pdfFile" type="file" accept="application/pdf" />
    <button id="extractBtn" type="button">텍스트 추출</button>
  </div>

  <p class="hint">※ 같은 폴더에 <code>pdf.mjs</code>, <code>pdf.worker.mjs</code>가 있어야 합니다.</p>
  <pre id="status"></pre>
  <textarea id="output" placeholder="추출된 텍스트가 여기에 표시됩니다."></textarea>

  <script type="module">
    import * as pdfjsLib from './pdf.mjs';

    pdfjsLib.GlobalWorkerOptions.workerSrc = './pdf.worker.mjs';

    const pdfFile = document.getElementById('pdfFile');
    const extractBtn = document.getElementById('extractBtn');
    const output = document.getElementById('output');
    const status = document.getElementById('status');

    const setStatus = (msg, isError = false) => {
      status.textContent = msg;
      status.className = isError ? 'error' : '';
    };

    async function extractTextFromPdf(file) {
      const data = new Uint8Array(await file.arrayBuffer());
      const loadingTask = pdfjsLib.getDocument({ data });
      const pdf = await loadingTask.promise;

      const pages = [];

      for (let pageNum = 1; pageNum <= pdf.numPages; pageNum++) {
        setStatus(`페이지 처리 중... ${pageNum}/${pdf.numPages}`);
        const page = await pdf.getPage(pageNum);
        const textContent = await page.getTextContent();

        const pageText = textContent.items
          .map(item => ('str' in item ? item.str : ''))
          .join(' ')
          .replace(/\s+/g, ' ')
          .trim();

        pages.push(`[Page ${pageNum}]\n${pageText}`);
      }

      return pages.join('\n\n');
    }

    extractBtn.addEventListener('click', async () => {
      try {
        const file = pdfFile.files?.[0];
        if (!file) {
          setStatus('PDF 파일을 먼저 선택하세요.', true);
          return;
        }

        output.value = '';
        setStatus('PDF 로드 중...');
        const text = await extractTextFromPdf(file);
        output.value = text;
        setStatus('완료: 텍스트 추출 성공');
      } catch (err) {
        setStatus(`실패:\n${err?.message ?? String(err)}`, true);
      }
    });
  </script>
</body>
</html>
```

---

## 주의 사항 (정확도/품질)
1. PDF는 본질적으로 “문서 레이아웃” 포맷이라 텍스트 순서가 기대와 다를 수 있습니다.
2. 스캔본(PDF가 이미지인 경우)은 OCR이 필요하므로 `getTextContent()`만으로는 텍스트가 안 나옵니다.
3. 표/다단 문서/각주가 많은 경우 줄바꿈/순서 보정 로직을 추가해야 품질이 좋아집니다.

---

## 빠른 실행 방법
1. `index.html`로 위 코드 저장
2. 같은 폴더에 `pdf.mjs`, `pdf.worker.mjs` 배치
3. (권장) 터미널에서 해당 폴더로 이동 후
   - `python -m http.server 8000`
4. 브라우저에서 `http://localhost:8000` 접속

---

## 요약
- 질문하신 조건(오프라인, 단일 HTML/CSS/JS, 로컬 `pdf.js` mjs 사용)에서 구현 가능합니다.
- 핵심은 `type="module"` import + `workerSrc` 경로 정확히 설정 + 페이지별 `getTextContent()` 집계입니다.
