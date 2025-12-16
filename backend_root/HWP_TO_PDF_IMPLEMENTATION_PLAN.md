---
sticker: emoji//1f4c5
---
# HWP to PDF 변환 구현 가이드

> "HWP 파일을 PDF로 보여줘야 하는데 어떻게 해?" 라는 질문에서 시작해보자.

---

## Q: 왜 HWP를 PDF로 변환해야 해?

기존 PDF 뷰어 모듈에서 **HWP 파일도 조회**하고 싶어서야.

```
사용자 → HWP 업로드 → ??? → PDF 뷰어에서 조회
                       ↑
                    여기가 문제!
```

---

## Q: 어떤 라이브러리를 써?

| 라이브러리 | 용도 | 라이선스 |
|-----------|------|----------|
| **hwplib** | HWP 파일 파싱 | Apache 2.0 (상업적 OK) |
| **hwpxlib** | HWPX 파일 파싱 | Apache 2.0 |
| **OpenPDF** | PDF 생성 | LGPL/MPL (상업적 OK) |

```gradle
dependencies {
    implementation 'kr.dogfoot:hwplib:1.1.1'
    implementation 'kr.dogfoot:hwpxlib:1.0.0'
    implementation 'com.github.librepdf:openpdf:1.4.2'
}
```

> **주의**: iTextPDF는 AGPL이라 상업용이면 OpenPDF를 써.

---

## Q: 전체 흐름은?

```
┌─────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────┐
│ HWP 파일 │ ──▶ │ hwplib 파싱 │ ──▶ │ 컨텐츠 추출  │ ──▶ │ PDF 생성 │
└─────────┘     └─────────────┘     └─────────────┘     └─────────┘
                                          │
                        ┌─────────────────┼─────────────────┐
                        ▼                 ▼                 ▼
                    텍스트 추출       표 추출           이미지 추출
```

---

## Q: 서비스 인터페이스는 어떻게 만들어?

```java
public interface HwpToPdfConversionService {

    // InputStream → PDF byte[]
    byte[] convertToPdf(InputStream inputStream) throws HwpConversionException;

    // 파일 경로 → PDF byte[]
    byte[] convertToPdf(String filePath) throws HwpConversionException;

    // HWP 파일인지 확인
    boolean canConvert(String fileName);
}
```

---

## Q: 구현체는?

```java
@Service
@RequiredArgsConstructor
public class HwpToPdfConversionServiceImpl implements HwpToPdfConversionService {

    private final HwpDocumentParser hwpDocumentParser;
    private final HwpContentExtractor hwpContentExtractor;
    private final PdfDocumentBuilder pdfDocumentBuilder;
    private final KoreanFontManager fontManager;

    @Override
    public byte[] convertToPdf(InputStream inputStream) throws HwpConversionException {
        try {
            // 1. HWP 파싱
            HWPFile hwpFile = hwpDocumentParser.parse(inputStream);

            // 2. 컨텐츠 추출
            HwpContent content = hwpContentExtractor.extract(hwpFile);

            // 3. PDF 생성
            return pdfDocumentBuilder
                .createDocument()
                .setFont(fontManager.getDefaultFont())
                .addContent(content)
                .build();

        } catch (Exception e) {
            throw new HwpConversionException("HWP 변환 실패", e);
        }
    }

    @Override
    public boolean canConvert(String fileName) {
        String ext = fileName.substring(fileName.lastIndexOf('.') + 1).toLowerCase();
        return Arrays.asList("hwp", "hwpx").contains(ext);
    }
}
```

---

## Q: HWP 파싱은 어떻게 해?

```java
@Component
public class HwpDocumentParser {

    public HWPFile parse(InputStream inputStream) throws HwpConversionException {
        File tempFile = null;
        try {
            // hwplib은 파일 경로가 필요해서 임시 파일 생성
            tempFile = File.createTempFile("hwp_convert_", ".hwp");
            copyInputStreamToFile(inputStream, tempFile);

            return HWPReader.fromFile(tempFile.getAbsolutePath());

        } catch (Exception e) {
            throw new HwpConversionException("HWP 파싱 실패", e);
        } finally {
            if (tempFile != null) tempFile.delete();
        }
    }
}
```

---

## Q: 텍스트는 어떻게 추출해?

```java
@Component
public class HwpContentExtractor {

    public HwpContent extract(HWPFile hwpFile) {
        HwpContent content = new HwpContent();

        // 섹션 순회
        for (Section section : hwpFile.getBodyText().getSectionList()) {
            // 문단 순회
            for (Paragraph para : section.getParagraphs()) {
                // 텍스트 추출
                ParaText paraText = para.getText();
                if (paraText != null) {
                    String text = paraText.getNormalString(0);
                    content.addParagraph(new HwpParagraph(text));
                }

                // 컨트롤(표, 이미지) 추출
                if (para.getControlList() != null) {
                    for (Control control : para.getControlList()) {
                        extractControl(control, content);
                    }
                }
            }
        }

        return content;
    }

    private void extractControl(Control control, HwpContent content) {
        switch (control.getType()) {
            case Table:
                content.addTable(extractTable((ControlTable) control));
                break;
            case Gso:  // 이미지
                content.addImage(extractImage((GsoControl) control));
                break;
        }
    }
}
```

---

## Q: PDF는 어떻게 생성해?

```java
@Component
public class PdfDocumentBuilder {

    private Document document;
    private ByteArrayOutputStream outputStream;
    private Font defaultFont;

    public PdfDocumentBuilder createDocument() {
        this.outputStream = new ByteArrayOutputStream();
        this.document = new Document(PageSize.A4);
        PdfWriter.getInstance(document, outputStream);
        document.open();
        return this;
    }

    public PdfDocumentBuilder setFont(Font font) {
        this.defaultFont = font;
        return this;
    }

    public PdfDocumentBuilder addContent(HwpContent content) {
        // 문단 추가
        for (HwpParagraph para : content.getParagraphs()) {
            Paragraph p = new Paragraph(para.getText(), defaultFont);
            document.add(p);
        }

        // 표 추가
        for (HwpTable table : content.getTables()) {
            addTable(table);
        }

        // 이미지 추가
        for (HwpImage image : content.getImages()) {
            addImage(image);
        }

        return this;
    }

    public byte[] build() {
        document.close();
        return outputStream.toByteArray();
    }
}
```

---

## Q: 한글 폰트는 어떻게 처리해?

PDF에서 한글이 깨지지 않으려면 **폰트 임베딩**이 필요해.

```java
@Component
public class KoreanFontManager {

    private Font defaultFont;

    @PostConstruct
    public void init() {
        // 시스템 폰트 경로에서 찾기
        String[] fontPaths = {
            "C:/Windows/Fonts/malgun.ttf",                           // Windows
            "/usr/share/fonts/truetype/nanum/NanumGothic.ttf",       // Linux
        };

        for (String path : fontPaths) {
            if (new File(path).exists()) {
                BaseFont baseFont = BaseFont.createFont(
                    path,
                    BaseFont.IDENTITY_H,  // 유니코드 가로쓰기
                    BaseFont.EMBEDDED     // 폰트 임베딩!
                );
                this.defaultFont = new Font(baseFont, 10);
                return;
            }
        }

        // 못 찾으면 기본 폰트 (한글 깨짐 가능)
        this.defaultFont = FontFactory.getFont(FontFactory.HELVETICA, 10);
    }

    public Font getDefaultFont() {
        return defaultFont;
    }
}
```

---

## Q: Controller에서는 어떻게 써?

```java
@RestController
@RequestMapping("/api/pdfviewer")
@RequiredArgsConstructor
public class PdfViewerController {

    private final PdfViewerService pdfViewerService;
    private final HwpToPdfConversionService hwpConversionService;

    @PostMapping("/getData.do")
    public Map<String, Object> getData(@RequestBody Map<String, Object> params) {
        Map<String, Object> result = new HashMap<>();

        String atchFileNo = (String) params.get("ATCH_FILE_NO");
        byte[] fileData = pdfViewerService.getFileData(atchFileNo);
        String fileName = pdfViewerService.getFileName(atchFileNo);

        // HWP 파일이면 PDF로 변환!
        if (hwpConversionService.canConvert(fileName)) {
            fileData = hwpConversionService.convertToPdf(
                new ByteArrayInputStream(fileData)
            );
            fileName = fileName.replaceAll("\\.(hwp|hwpx)$", ".pdf");
        }

        result.put("fileData", Base64.getEncoder().encodeToString(fileData));
        result.put("fileName", fileName);

        return result;
    }
}
```

---

## Q: 지원/미지원 기능은?

### 완전 지원

| 기능 | 상태 |
|------|------|
| 일반 텍스트 | ✅ |
| 문단 정렬 (좌/중/우) | ✅ |
| 기본 표 | ✅ |
| 이미지 (PNG, JPEG) | ✅ |

### 부분 지원

| 기능 | 상태 | 비고 |
|------|------|------|
| 셀 병합 | ⚠️ | 제한적 |
| 글머리 기호 | ⚠️ | 텍스트로 변환 |
| 머리말/꼬리말 | ⚠️ | |

### 미지원

| 기능 | 상태 | 대안 |
|------|------|------|
| 암호화된 HWP | ❌ | 암호 해제 후 변환 |
| 도형 | ❌ | 이미지로 대체 |
| 수식 | ❌ | 이미지로 대체 |
| 다단 레이아웃 | ❌ | 단일 단으로 변환 |

---

## Q: 품질을 더 높이려면?

### 방법 1: HTML 중간 변환 (권장)

```
HWP → HTML/CSS → PDF
```

```gradle
// Flying Saucer (HTML to PDF)
implementation 'org.xhtmlrenderer:flying-saucer-pdf:9.7.2'

// 또는 OpenHTMLToPDF
implementation 'com.openhtmltopdf:openhtmltopdf-pdfbox:1.0.10'
```

HTML/CSS로 변환하면 레이아웃 품질이 **대폭 향상**돼.

### 방법 2: 스타일 정밀 매핑

hwplib의 `CharShape`, `ParaShape` 정보를 PDF 스타일로 정밀하게 매핑.

```java
// CharShape에서 폰트 크기 추출
float fontSize = charShape.getBaseSize() / 100.0f;  // 1/100 pt → pt

// ParaShape에서 정렬 추출
int align = paraShape.getProperty1().getAlignmentType().getValue();
// 0: 양쪽, 1: 왼쪽, 2: 오른쪽, 3: 가운데
```

---

## Q: 예상 품질은?

| | 기본 구현 | 품질 향상 적용 |
|---|---|---|
| 텍스트 레이아웃 | 기본 | 원본과 유사 |
| 폰트 스타일 | 크기/굵기만 | 대부분 지원 |
| 표 렌더링 | 단순 표만 | 셀 병합 지원 |
| **전체 충실도** | **60~70%** | **85~95%** |

---

## 디렉토리 구조

```
src/main/java/com/fw/cmm/hwpconvert/
├── HwpToPdfConversionService.java       # 인터페이스
├── HwpToPdfConversionServiceImpl.java   # 구현체
├── parser/
│   └── HwpDocumentParser.java           # HWP 파싱
├── extractor/
│   └── HwpContentExtractor.java         # 컨텐츠 추출
├── builder/
│   └── PdfDocumentBuilder.java          # PDF 생성
├── font/
│   └── KoreanFontManager.java           # 폰트 관리
├── model/
│   ├── HwpContent.java
│   ├── HwpParagraph.java
│   ├── HwpTable.java
│   └── HwpImage.java
└── exception/
    └── HwpConversionException.java
```

---

## 설정 (application.yml)

```yaml
hwp:
  convert:
    enabled: true
    font:
      path: ""  # 비어있으면 시스템 폰트 자동 탐색
    max-file-size: 50  # MB
    timeout: 60        # 초
```

---

## 체크리스트

### 개발 시

- [ ] hwplib, OpenPDF 의존성 추가
- [ ] 한글 폰트 경로 확인
- [ ] 테스트용 HWP 파일 준비

### 배포 시

- [ ] 서버에 한글 폰트 설치
- [ ] 임시 파일 디렉토리 권한 확인
- [ ] 메모리/타임아웃 설정

---

## 참고 자료

- [hwplib GitHub](https://github.com/neolord0/hwplib)
- [OpenPDF GitHub](https://github.com/LibrePDF/OpenPDF)
- [한글 문서 파일 구조 5.0](https://www.hancom.com/etc/hwpDownload.do)

---

## 이어서 읽기

- [[SPRING_BOOT_PROJECT_REVIEW]] - Spring Boot 프로젝트 구조
- [[REST_API_설계_가이드_레거시vs현대]] - REST API 설계 패턴
- [[ORACLE_CLOB_JSON_SERIALIZATION_ERROR]] - CLOB 직렬화 이슈

