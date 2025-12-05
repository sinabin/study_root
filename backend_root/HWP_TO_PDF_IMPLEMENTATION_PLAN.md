# HWP to PDF 변환 기능 구현 계획서

## hwplib + OpenPDF 커스텀 방식

---

## 1. 개요

### 1.1 목적
한글 문서 파일(.hwp, .hwpx)을 PDF로 변환하여 기존 PDF 뷰어 모듈에서 조회 및 편집할 수 있도록 기능을 확장합니다.

### 1.2 기술 스택
| 구분 | 기술 | 버전 | 용도 |
|------|------|------|------|
| HWP 파싱 | [hwplib](https://github.com/neolord0/hwplib) | 1.1.1+ | HWP 파일 읽기 및 구조 분석 |
| HWPX 파싱 | [hwpxlib](https://github.com/neolord0/hwpxlib) | 1.0.0+ | HWPX 파일 읽기 |
| PDF 생성 | [OpenPDF](https://github.com/LibrePDF/OpenPDF) | 1.4.2+ | PDF 문서 생성 (LGPL/MPL) |
| 이미지 처리 | Java ImageIO | JDK 내장 | 이미지 변환 |
| 폰트 처리 | OpenPDF Font | - | 한글 폰트 임베딩 |

### 1.3 라이선스
- **hwplib**: Apache 2.0 (상업적 사용 가능)
- **OpenPDF**: LGPL/MPL (상업적 사용 가능, AGPL 아님)

---

## 2. 아키텍처

### 2.1 전체 구조

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              Client                                      │
│                    (PdfViewerPOP.js - 기존 컴포넌트)                     │
└─────────────────────────────────┬───────────────────────────────────────┘
                                  │ POST /api/pdfviewer/getData.do
                                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         PdfViewerController                              │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  1. 파일 확장자 확인 (.hwp / .hwpx / .pdf)                      │   │
│  │  2. HWP/HWPX인 경우 → HwpToPdfConverter 호출                    │   │
│  │  3. PDF인 경우 → 기존 로직 유지                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────┬───────────────────────────────────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    ▼                           ▼
┌───────────────────────────────┐ ┌───────────────────────────────────────┐
│      HwpToPdfConverter        │ │         기존 PDF 처리 로직            │
│  ┌─────────────────────────┐  │ │                                       │
│  │   HwpDocumentParser     │  │ │  - 파일 스트림 반환                   │
│  │   (hwplib 기반)         │  │ │  - Base64 인코딩                      │
│  └───────────┬─────────────┘  │ └───────────────────────────────────────┘
│              ▼                │
│  ┌─────────────────────────┐  │
│  │   HwpContentExtractor   │  │
│  │   - 텍스트 추출         │  │
│  │   - 표 구조 추출        │  │
│  │   - 이미지 추출         │  │
│  │   - 스타일 정보 추출    │  │
│  └───────────┬─────────────┘  │
│              ▼                │
│  ┌─────────────────────────┐  │
│  │   PdfDocumentBuilder    │  │
│  │   (OpenPDF 기반)        │  │
│  │   - 텍스트 렌더링       │  │
│  │   - 표 렌더링           │  │
│  │   - 이미지 삽입         │  │
│  │   - 폰트 임베딩         │  │
│  └───────────┬─────────────┘  │
│              ▼                │
│  ┌─────────────────────────┐  │
│  │   PDF ByteArray 반환    │  │
│  └─────────────────────────┘  │
└───────────────────────────────┘
```

### 2.2 클래스 다이어그램

```
┌─────────────────────────────────────────────────────────────────┐
│                    HwpToPdfConversionService                    │
│  <<interface>>                                                  │
├─────────────────────────────────────────────────────────────────┤
│  + convertToPdf(inputStream: InputStream): byte[]               │
│  + convertToPdf(filePath: String): byte[]                       │
│  + getSupportedExtensions(): List<String>                       │
└─────────────────────────────────────────────────────────────────┘
                              △
                              │ implements
                              │
┌─────────────────────────────────────────────────────────────────┐
│                  HwpToPdfConversionServiceImpl                  │
├─────────────────────────────────────────────────────────────────┤
│  - hwpDocumentParser: HwpDocumentParser                         │
│  - hwpContentExtractor: HwpContentExtractor                     │
│  - pdfDocumentBuilder: PdfDocumentBuilder                       │
│  - fontManager: KoreanFontManager                               │
├─────────────────────────────────────────────────────────────────┤
│  + convertToPdf(inputStream: InputStream): byte[]               │
│  + convertToPdf(filePath: String): byte[]                       │
│  - validateFile(inputStream: InputStream): void                 │
└─────────────────────────────────────────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
          ▼                   ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────────────┐
│HwpDocumentParser│ │HwpContentExtract│ │  PdfDocumentBuilder     │
├─────────────────┤ ├─────────────────┤ ├─────────────────────────┤
│+ parse(): HWPFi │ │+ extractText()  │ │+ createDocument()       │
│+ parseHwpx()    │ │+ extractTables()│ │+ addParagraph()         │
└─────────────────┘ │+ extractImages()│ │+ addTable()             │
                    │+ extractStyles()│ │+ addImage()             │
                    └─────────────────┘ │+ build(): byte[]        │
                                        └─────────────────────────┘
```

---

## 3. 의존성 설정

### 3.1 build.gradle

```gradle
dependencies {
    // HWP 파싱 라이브러리
    implementation 'kr.dogfoot:hwplib:1.1.1'

    // HWPX 파싱 라이브러리 (선택)
    implementation 'kr.dogfoot:hwpxlib:1.0.0'

    // PDF 생성 라이브러리 (LGPL/MPL - AGPL 아님)
    implementation 'com.github.librepdf:openpdf:1.4.2'

    // 이미지 처리 (선택)
    implementation 'org.apache.commons:commons-imaging:1.0-alpha3'
}
```

### 3.2 Maven (pom.xml)

```xml
<dependencies>
    <!-- HWP 파싱 라이브러리 -->
    <dependency>
        <groupId>kr.dogfoot</groupId>
        <artifactId>hwplib</artifactId>
        <version>1.1.1</version>
    </dependency>

    <!-- HWPX 파싱 라이브러리 (선택) -->
    <dependency>
        <groupId>kr.dogfoot</groupId>
        <artifactId>hwpxlib</artifactId>
        <version>1.0.0</version>
    </dependency>

    <!-- PDF 생성 라이브러리 -->
    <dependency>
        <groupId>com.github.librepdf</groupId>
        <artifactId>openpdf</artifactId>
        <version>1.4.2</version>
    </dependency>
</dependencies>
```

---

## 4. 상세 구현

### 4.1 디렉토리 구조

```
src/main/java/com/fw/cmm/hwpconvert/
├── HwpToPdfConversionService.java          # 서비스 인터페이스
├── HwpToPdfConversionServiceImpl.java      # 서비스 구현체
├── parser/
│   ├── HwpDocumentParser.java              # HWP 파일 파싱
│   └── HwpxDocumentParser.java             # HWPX 파일 파싱
├── extractor/
│   ├── HwpContentExtractor.java            # 컨텐츠 추출 인터페이스
│   ├── HwpTextExtractor.java               # 텍스트 추출
│   ├── HwpTableExtractor.java              # 표 추출
│   ├── HwpImageExtractor.java              # 이미지 추출
│   └── HwpStyleExtractor.java              # 스타일 정보 추출
├── builder/
│   ├── PdfDocumentBuilder.java             # PDF 문서 빌더
│   ├── PdfTextRenderer.java                # 텍스트 렌더링
│   ├── PdfTableRenderer.java               # 표 렌더링
│   └── PdfImageRenderer.java               # 이미지 렌더링
├── font/
│   └── KoreanFontManager.java              # 한글 폰트 관리
├── model/
│   ├── HwpContent.java                     # 추출된 컨텐츠 모델
│   ├── HwpParagraph.java                   # 문단 모델
│   ├── HwpTable.java                       # 표 모델
│   ├── HwpImage.java                       # 이미지 모델
│   └── HwpStyle.java                       # 스타일 모델
└── exception/
    ├── HwpConversionException.java         # 변환 예외
    └── UnsupportedHwpFeatureException.java # 미지원 기능 예외
```

### 4.2 핵심 클래스 구현

#### 4.2.1 서비스 인터페이스

```java
package com.fw.cmm.hwpconvert;

import java.io.InputStream;
import java.util.List;

/**
 * HWP to PDF 변환 서비스 인터페이스
 */
public interface HwpToPdfConversionService {

    /**
     * HWP 파일을 PDF로 변환
     * @param inputStream HWP 파일 입력 스트림
     * @return PDF 바이트 배열
     */
    byte[] convertToPdf(InputStream inputStream) throws HwpConversionException;

    /**
     * HWP 파일을 PDF로 변환
     * @param filePath HWP 파일 경로
     * @return PDF 바이트 배열
     */
    byte[] convertToPdf(String filePath) throws HwpConversionException;

    /**
     * 지원하는 파일 확장자 목록
     */
    List<String> getSupportedExtensions();

    /**
     * 파일 변환 가능 여부 확인
     */
    boolean canConvert(String fileName);
}
```

#### 4.2.2 서비스 구현체

```java
package com.fw.cmm.hwpconvert;

import com.fw.cmm.hwpconvert.parser.HwpDocumentParser;
import com.fw.cmm.hwpconvert.extractor.HwpContentExtractor;
import com.fw.cmm.hwpconvert.builder.PdfDocumentBuilder;
import com.fw.cmm.hwpconvert.font.KoreanFontManager;
import com.fw.cmm.hwpconvert.model.HwpContent;
import com.fw.cmm.hwpconvert.exception.HwpConversionException;

import kr.dogfoot.hwplib.object.HWPFile;
import org.springframework.stereotype.Service;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

import java.io.*;
import java.util.Arrays;
import java.util.List;

@Slf4j
@Service
@RequiredArgsConstructor
public class HwpToPdfConversionServiceImpl implements HwpToPdfConversionService {

    private static final List<String> SUPPORTED_EXTENSIONS = Arrays.asList("hwp", "hwpx");

    private final HwpDocumentParser hwpDocumentParser;
    private final HwpContentExtractor hwpContentExtractor;
    private final PdfDocumentBuilder pdfDocumentBuilder;
    private final KoreanFontManager fontManager;

    @Override
    public byte[] convertToPdf(InputStream inputStream) throws HwpConversionException {
        try {
            log.info("HWP to PDF 변환 시작");

            // 1. HWP 파일 파싱
            HWPFile hwpFile = hwpDocumentParser.parse(inputStream);
            log.debug("HWP 파일 파싱 완료");

            // 2. 컨텐츠 추출
            HwpContent content = hwpContentExtractor.extract(hwpFile);
            log.debug("컨텐츠 추출 완료 - 문단 수: {}, 표 수: {}, 이미지 수: {}",
                content.getParagraphs().size(),
                content.getTables().size(),
                content.getImages().size());

            // 3. PDF 생성
            byte[] pdfBytes = pdfDocumentBuilder
                .createDocument()
                .setFont(fontManager.getDefaultFont())
                .addContent(content)
                .build();

            log.info("HWP to PDF 변환 완료 - PDF 크기: {} bytes", pdfBytes.length);
            return pdfBytes;

        } catch (Exception e) {
            log.error("HWP to PDF 변환 실패", e);
            throw new HwpConversionException("HWP 파일 변환 중 오류가 발생했습니다.", e);
        }
    }

    @Override
    public byte[] convertToPdf(String filePath) throws HwpConversionException {
        try (InputStream is = new FileInputStream(filePath)) {
            return convertToPdf(is);
        } catch (IOException e) {
            throw new HwpConversionException("파일을 읽을 수 없습니다: " + filePath, e);
        }
    }

    @Override
    public List<String> getSupportedExtensions() {
        return SUPPORTED_EXTENSIONS;
    }

    @Override
    public boolean canConvert(String fileName) {
        if (fileName == null || fileName.isEmpty()) {
            return false;
        }
        String extension = fileName.substring(fileName.lastIndexOf('.') + 1).toLowerCase();
        return SUPPORTED_EXTENSIONS.contains(extension);
    }
}
```

#### 4.2.3 HWP 문서 파서

```java
package com.fw.cmm.hwpconvert.parser;

import kr.dogfoot.hwplib.object.HWPFile;
import kr.dogfoot.hwplib.reader.HWPReader;
import com.fw.cmm.hwpconvert.exception.HwpConversionException;

import org.springframework.stereotype.Component;
import lombok.extern.slf4j.Slf4j;

import java.io.*;

@Slf4j
@Component
public class HwpDocumentParser {

    /**
     * HWP 파일을 파싱하여 HWPFile 객체로 변환
     */
    public HWPFile parse(InputStream inputStream) throws HwpConversionException {
        File tempFile = null;
        try {
            // hwplib은 파일 경로를 필요로 하므로 임시 파일 생성
            tempFile = File.createTempFile("hwp_convert_", ".hwp");
            copyInputStreamToFile(inputStream, tempFile);

            return HWPReader.fromFile(tempFile.getAbsolutePath());

        } catch (Exception e) {
            log.error("HWP 파일 파싱 실패", e);
            throw new HwpConversionException("HWP 파일을 파싱할 수 없습니다.", e);
        } finally {
            if (tempFile != null && tempFile.exists()) {
                tempFile.delete();
            }
        }
    }

    /**
     * HWP 파일을 파싱 (파일 경로)
     */
    public HWPFile parse(String filePath) throws HwpConversionException {
        try {
            return HWPReader.fromFile(filePath);
        } catch (Exception e) {
            log.error("HWP 파일 파싱 실패: {}", filePath, e);
            throw new HwpConversionException("HWP 파일을 파싱할 수 없습니다: " + filePath, e);
        }
    }

    private void copyInputStreamToFile(InputStream inputStream, File file) throws IOException {
        try (OutputStream outputStream = new FileOutputStream(file)) {
            byte[] buffer = new byte[8192];
            int bytesRead;
            while ((bytesRead = inputStream.read(buffer)) != -1) {
                outputStream.write(buffer, 0, bytesRead);
            }
        }
    }
}
```

#### 4.2.4 컨텐츠 추출기

```java
package com.fw.cmm.hwpconvert.extractor;

import kr.dogfoot.hwplib.object.HWPFile;
import kr.dogfoot.hwplib.object.bodytext.Section;
import kr.dogfoot.hwplib.object.bodytext.paragraph.Paragraph;
import kr.dogfoot.hwplib.object.bodytext.paragraph.text.ParaText;
import kr.dogfoot.hwplib.object.bodytext.control.Control;
import kr.dogfoot.hwplib.object.bodytext.control.ControlTable;
import kr.dogfoot.hwplib.object.bodytext.control.ControlType;
import kr.dogfoot.hwplib.object.bodytext.control.gso.GsoControl;
import kr.dogfoot.hwplib.object.bodytext.control.gso.GsoControlType;
import kr.dogfoot.hwplib.object.bodytext.control.gso.textbox.TextBox;
import kr.dogfoot.hwplib.tool.textextractor.TextExtractor;

import com.fw.cmm.hwpconvert.model.*;
import com.fw.cmm.hwpconvert.exception.HwpConversionException;

import org.springframework.stereotype.Component;
import lombok.extern.slf4j.Slf4j;

import java.util.ArrayList;
import java.util.List;

@Slf4j
@Component
public class HwpContentExtractor {

    /**
     * HWP 파일에서 모든 컨텐츠 추출
     */
    public HwpContent extract(HWPFile hwpFile) throws HwpConversionException {
        try {
            HwpContent content = new HwpContent();

            // 문서 정보 추출
            extractDocumentInfo(hwpFile, content);

            // 본문 섹션 순회
            for (Section section : hwpFile.getBodyText().getSectionList()) {
                extractSectionContent(section, content);
            }

            // 이미지(바이너리 데이터) 추출
            extractBinaryData(hwpFile, content);

            return content;

        } catch (Exception e) {
            log.error("컨텐츠 추출 실패", e);
            throw new HwpConversionException("HWP 컨텐츠를 추출할 수 없습니다.", e);
        }
    }

    /**
     * 문서 정보 추출 (용지 크기, 여백 등)
     */
    private void extractDocumentInfo(HWPFile hwpFile, HwpContent content) {
        try {
            // 첫 번째 섹션의 용지 설정 사용
            if (!hwpFile.getBodyText().getSectionList().isEmpty()) {
                Section firstSection = hwpFile.getBodyText().getSectionList().get(0);

                // 용지 크기 (HWPUNIT: 1/7200 인치)
                long paperWidth = firstSection.getDocumentGrid().getProperty().getPaperWidth();
                long paperHeight = firstSection.getDocumentGrid().getProperty().getPaperHeight();

                content.setPageWidth(hwpUnitToPoint(paperWidth));
                content.setPageHeight(hwpUnitToPoint(paperHeight));

                log.debug("용지 크기: {}pt x {}pt", content.getPageWidth(), content.getPageHeight());
            }
        } catch (Exception e) {
            log.warn("문서 정보 추출 실패, 기본값 사용", e);
            // A4 기본값 (595 x 842 pt)
            content.setPageWidth(595f);
            content.setPageHeight(842f);
        }
    }

    /**
     * 섹션 컨텐츠 추출
     */
    private void extractSectionContent(Section section, HwpContent content) {
        for (Paragraph paragraph : section.getParagraphs()) {
            // 문단 텍스트 추출
            HwpParagraph hwpPara = extractParagraph(paragraph);
            if (hwpPara != null && !hwpPara.getText().isEmpty()) {
                content.addParagraph(hwpPara);
            }

            // 컨트롤(표, 이미지 등) 추출
            if (paragraph.getControlList() != null) {
                for (Control control : paragraph.getControlList()) {
                    extractControl(control, content);
                }
            }
        }
    }

    /**
     * 문단 추출
     */
    private HwpParagraph extractParagraph(Paragraph paragraph) {
        HwpParagraph hwpPara = new HwpParagraph();

        try {
            ParaText paraText = paragraph.getText();
            if (paraText != null) {
                String text = paraText.getNormalString(0);
                hwpPara.setText(text != null ? text : "");
            }

            // 문단 스타일 정보 추출
            extractParagraphStyle(paragraph, hwpPara);

        } catch (Exception e) {
            log.warn("문단 추출 실패", e);
        }

        return hwpPara;
    }

    /**
     * 문단 스타일 추출
     */
    private void extractParagraphStyle(Paragraph paragraph, HwpParagraph hwpPara) {
        try {
            // 글자 모양 ID
            if (paragraph.getCharShape() != null &&
                paragraph.getCharShape().getPositonShapeIdPairList() != null &&
                !paragraph.getCharShape().getPositonShapeIdPairList().isEmpty()) {

                int charShapeId = (int) paragraph.getCharShape()
                    .getPositonShapeIdPairList().get(0).getShapeId();
                hwpPara.setCharShapeId(charShapeId);
            }

            // 문단 모양 ID
            hwpPara.setParaShapeId(paragraph.getHeader().getParaShapeId());

        } catch (Exception e) {
            log.warn("문단 스타일 추출 실패", e);
        }
    }

    /**
     * 컨트롤 추출 (표, 이미지 등)
     */
    private void extractControl(Control control, HwpContent content) {
        try {
            ControlType type = control.getType();

            switch (type) {
                case Table:
                    HwpTable table = extractTable((ControlTable) control);
                    if (table != null) {
                        content.addTable(table);
                    }
                    break;

                case Gso:
                    extractGsoControl((GsoControl) control, content);
                    break;

                default:
                    log.debug("미지원 컨트롤 타입: {}", type);
                    break;
            }
        } catch (Exception e) {
            log.warn("컨트롤 추출 실패: {}", control.getType(), e);
        }
    }

    /**
     * 표 추출
     */
    private HwpTable extractTable(ControlTable controlTable) {
        HwpTable table = new HwpTable();

        try {
            // 행/열 정보
            int rowCount = controlTable.getTable().getRowCount();
            int colCount = controlTable.getTable().getColCount();

            table.setRowCount(rowCount);
            table.setColCount(colCount);

            // 셀 데이터 추출
            List<List<String>> cellData = new ArrayList<>();
            // TODO: 셀 순회 및 데이터 추출 로직 구현

            table.setCellData(cellData);

            log.debug("표 추출 완료: {}행 x {}열", rowCount, colCount);

        } catch (Exception e) {
            log.warn("표 추출 실패", e);
        }

        return table;
    }

    /**
     * GSO 컨트롤 추출 (그림, 도형 등)
     */
    private void extractGsoControl(GsoControl gsoControl, HwpContent content) {
        try {
            GsoControlType gsoType = gsoControl.getGsoType();

            switch (gsoType) {
                case Picture:
                    // 이미지 추출
                    HwpImage image = extractImage(gsoControl);
                    if (image != null) {
                        content.addImage(image);
                    }
                    break;

                case Rectangle:
                case Ellipse:
                case Line:
                    // 도형은 현재 미지원
                    log.debug("도형 컨트롤 스킵: {}", gsoType);
                    break;

                default:
                    log.debug("미지원 GSO 타입: {}", gsoType);
                    break;
            }
        } catch (Exception e) {
            log.warn("GSO 컨트롤 추출 실패", e);
        }
    }

    /**
     * 이미지 추출
     */
    private HwpImage extractImage(GsoControl gsoControl) {
        HwpImage image = new HwpImage();

        try {
            // 이미지 바이너리 데이터 ID 추출
            // TODO: 실제 이미지 데이터 추출 로직

        } catch (Exception e) {
            log.warn("이미지 추출 실패", e);
        }

        return image;
    }

    /**
     * 바이너리 데이터(이미지 등) 추출
     */
    private void extractBinaryData(HWPFile hwpFile, HwpContent content) {
        try {
            if (hwpFile.getBinData() != null &&
                hwpFile.getBinData().getEmbeddedBinaryDataList() != null) {

                for (var binData : hwpFile.getBinData().getEmbeddedBinaryDataList()) {
                    HwpImage image = new HwpImage();
                    image.setData(binData.getData());
                    image.setFormat(binData.getCompressMethod().name());
                    content.addImage(image);
                }

                log.debug("바이너리 데이터 추출 완료: {} 개",
                    hwpFile.getBinData().getEmbeddedBinaryDataList().size());
            }
        } catch (Exception e) {
            log.warn("바이너리 데이터 추출 실패", e);
        }
    }

    /**
     * HWP 단위를 포인트로 변환 (1 HWPUNIT = 1/7200 inch, 1 inch = 72 pt)
     */
    private float hwpUnitToPoint(long hwpUnit) {
        return (float) (hwpUnit * 72.0 / 7200.0);
    }
}
```

#### 4.2.5 PDF 문서 빌더

```java
package com.fw.cmm.hwpconvert.builder;

import com.fw.cmm.hwpconvert.model.*;
import com.fw.cmm.hwpconvert.font.KoreanFontManager;

import com.lowagie.text.*;
import com.lowagie.text.pdf.*;

import org.springframework.stereotype.Component;
import lombok.extern.slf4j.Slf4j;

import java.io.ByteArrayOutputStream;
import java.io.IOException;

@Slf4j
@Component
public class PdfDocumentBuilder {

    private Document document;
    private ByteArrayOutputStream outputStream;
    private PdfWriter writer;
    private Font defaultFont;
    private float pageWidth = 595f;  // A4 기본값
    private float pageHeight = 842f;

    /**
     * 새 PDF 문서 생성
     */
    public PdfDocumentBuilder createDocument() {
        this.outputStream = new ByteArrayOutputStream();
        this.document = new Document(PageSize.A4);

        try {
            this.writer = PdfWriter.getInstance(document, outputStream);
            document.open();
        } catch (DocumentException e) {
            log.error("PDF 문서 생성 실패", e);
            throw new RuntimeException("PDF 문서를 생성할 수 없습니다.", e);
        }

        return this;
    }

    /**
     * 페이지 크기 설정
     */
    public PdfDocumentBuilder setPageSize(float width, float height) {
        this.pageWidth = width;
        this.pageHeight = height;
        this.document.setPageSize(new Rectangle(width, height));
        return this;
    }

    /**
     * 기본 폰트 설정
     */
    public PdfDocumentBuilder setFont(Font font) {
        this.defaultFont = font;
        return this;
    }

    /**
     * HWP 컨텐츠 추가
     */
    public PdfDocumentBuilder addContent(HwpContent content) {
        try {
            // 페이지 크기 설정
            if (content.getPageWidth() > 0 && content.getPageHeight() > 0) {
                setPageSize(content.getPageWidth(), content.getPageHeight());
            }

            // 문단 추가
            for (HwpParagraph paragraph : content.getParagraphs()) {
                addParagraph(paragraph);
            }

            // 표 추가
            for (HwpTable table : content.getTables()) {
                addTable(table);
            }

            // 이미지 추가
            for (HwpImage image : content.getImages()) {
                addImage(image);
            }

        } catch (Exception e) {
            log.error("컨텐츠 추가 실패", e);
            throw new RuntimeException("PDF 컨텐츠 추가에 실패했습니다.", e);
        }

        return this;
    }

    /**
     * 문단 추가
     */
    public PdfDocumentBuilder addParagraph(HwpParagraph hwpPara) {
        try {
            if (hwpPara.getText() == null || hwpPara.getText().trim().isEmpty()) {
                // 빈 문단은 줄바꿈으로 처리
                document.add(Chunk.NEWLINE);
                return this;
            }

            Paragraph paragraph = new Paragraph(hwpPara.getText(), defaultFont);

            // 문단 정렬 설정
            switch (hwpPara.getAlignment()) {
                case CENTER:
                    paragraph.setAlignment(Element.ALIGN_CENTER);
                    break;
                case RIGHT:
                    paragraph.setAlignment(Element.ALIGN_RIGHT);
                    break;
                case JUSTIFY:
                    paragraph.setAlignment(Element.ALIGN_JUSTIFIED);
                    break;
                default:
                    paragraph.setAlignment(Element.ALIGN_LEFT);
            }

            // 줄 간격 설정
            paragraph.setLeading(defaultFont.getSize() * 1.5f);

            document.add(paragraph);

        } catch (DocumentException e) {
            log.warn("문단 추가 실패: {}", hwpPara.getText(), e);
        }

        return this;
    }

    /**
     * 표 추가
     */
    public PdfDocumentBuilder addTable(HwpTable hwpTable) {
        try {
            if (hwpTable.getColCount() <= 0) {
                return this;
            }

            PdfPTable pdfTable = new PdfPTable(hwpTable.getColCount());
            pdfTable.setWidthPercentage(100);
            pdfTable.setSpacingBefore(10f);
            pdfTable.setSpacingAfter(10f);

            // 셀 데이터 추가
            if (hwpTable.getCellData() != null) {
                for (List<String> row : hwpTable.getCellData()) {
                    for (String cellText : row) {
                        PdfPCell cell = new PdfPCell(new Phrase(cellText, defaultFont));
                        cell.setPadding(5f);
                        cell.setBorderWidth(0.5f);
                        pdfTable.addCell(cell);
                    }
                }
            }

            document.add(pdfTable);

        } catch (DocumentException e) {
            log.warn("표 추가 실패", e);
        }

        return this;
    }

    /**
     * 이미지 추가
     */
    public PdfDocumentBuilder addImage(HwpImage hwpImage) {
        try {
            if (hwpImage.getData() == null || hwpImage.getData().length == 0) {
                return this;
            }

            Image image = Image.getInstance(hwpImage.getData());

            // 이미지 크기 조정 (페이지 너비에 맞춤)
            float maxWidth = pageWidth - 72; // 여백 고려
            if (image.getWidth() > maxWidth) {
                float ratio = maxWidth / image.getWidth();
                image.scalePercent(ratio * 100);
            }

            image.setAlignment(Element.ALIGN_CENTER);
            document.add(image);

        } catch (Exception e) {
            log.warn("이미지 추가 실패", e);
        }

        return this;
    }

    /**
     * PDF 바이트 배열 생성
     */
    public byte[] build() {
        try {
            document.close();
            return outputStream.toByteArray();
        } finally {
            try {
                outputStream.close();
            } catch (IOException e) {
                log.warn("출력 스트림 닫기 실패", e);
            }
        }
    }
}
```

#### 4.2.6 한글 폰트 매니저

```java
package com.fw.cmm.hwpconvert.font;

import com.lowagie.text.Font;
import com.lowagie.text.FontFactory;
import com.lowagie.text.pdf.BaseFont;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
import lombok.extern.slf4j.Slf4j;

import javax.annotation.PostConstruct;
import java.io.File;

@Slf4j
@Component
public class KoreanFontManager {

    @Value("${hwp.convert.font.path:}")
    private String customFontPath;

    @Value("${hwp.convert.font.name:malgun}")
    private String defaultFontName;

    private Font defaultFont;
    private BaseFont baseFont;

    // 시스템 폰트 경로 목록
    private static final String[] FONT_PATHS = {
        // Windows
        "C:/Windows/Fonts/malgun.ttf",
        "C:/Windows/Fonts/NanumGothic.ttf",
        "C:/Windows/Fonts/gulim.ttc",
        // Linux
        "/usr/share/fonts/truetype/nanum/NanumGothic.ttf",
        "/usr/share/fonts/truetype/nanum/NanumMyeongjo.ttf",
        // macOS
        "/Library/Fonts/AppleGothic.ttf",
        "/System/Library/Fonts/AppleSDGothicNeo.ttc"
    };

    @PostConstruct
    public void init() {
        try {
            // 커스텀 폰트 경로가 설정된 경우
            if (customFontPath != null && !customFontPath.isEmpty()) {
                loadFont(customFontPath);
                return;
            }

            // 시스템 폰트 자동 탐색
            for (String fontPath : FONT_PATHS) {
                if (new File(fontPath).exists()) {
                    loadFont(fontPath);
                    return;
                }
            }

            // 폰트를 찾지 못한 경우 기본 폰트 사용
            log.warn("한글 폰트를 찾지 못했습니다. 기본 폰트를 사용합니다.");
            this.defaultFont = FontFactory.getFont(FontFactory.HELVETICA, 12);

        } catch (Exception e) {
            log.error("폰트 초기화 실패", e);
            this.defaultFont = FontFactory.getFont(FontFactory.HELVETICA, 12);
        }
    }

    private void loadFont(String fontPath) {
        try {
            this.baseFont = BaseFont.createFont(
                fontPath,
                BaseFont.IDENTITY_H,  // 유니코드 가로쓰기
                BaseFont.EMBEDDED     // 폰트 임베딩
            );
            this.defaultFont = new Font(baseFont, 10);
            log.info("한글 폰트 로드 완료: {}", fontPath);
        } catch (Exception e) {
            log.error("폰트 로드 실패: {}", fontPath, e);
            throw new RuntimeException("폰트를 로드할 수 없습니다: " + fontPath, e);
        }
    }

    /**
     * 기본 폰트 반환
     */
    public Font getDefaultFont() {
        return defaultFont;
    }

    /**
     * 지정된 크기의 폰트 반환
     */
    public Font getFont(float size) {
        if (baseFont != null) {
            return new Font(baseFont, size);
        }
        return FontFactory.getFont(FontFactory.HELVETICA, size);
    }

    /**
     * 지정된 크기와 스타일의 폰트 반환
     */
    public Font getFont(float size, int style) {
        if (baseFont != null) {
            return new Font(baseFont, size, style);
        }
        return FontFactory.getFont(FontFactory.HELVETICA, size, style);
    }

    /**
     * 볼드 폰트 반환
     */
    public Font getBoldFont(float size) {
        return getFont(size, Font.BOLD);
    }

    /**
     * 이탤릭 폰트 반환
     */
    public Font getItalicFont(float size) {
        return getFont(size, Font.ITALIC);
    }
}
```

#### 4.2.7 데이터 모델

```java
// HwpContent.java
package com.fw.cmm.hwpconvert.model;

import lombok.Data;
import java.util.ArrayList;
import java.util.List;

@Data
public class HwpContent {
    private float pageWidth = 595f;   // A4 기본값 (pt)
    private float pageHeight = 842f;
    private float marginTop = 72f;
    private float marginBottom = 72f;
    private float marginLeft = 72f;
    private float marginRight = 72f;

    private List<HwpParagraph> paragraphs = new ArrayList<>();
    private List<HwpTable> tables = new ArrayList<>();
    private List<HwpImage> images = new ArrayList<>();

    public void addParagraph(HwpParagraph paragraph) {
        this.paragraphs.add(paragraph);
    }

    public void addTable(HwpTable table) {
        this.tables.add(table);
    }

    public void addImage(HwpImage image) {
        this.images.add(image);
    }
}

// HwpParagraph.java
package com.fw.cmm.hwpconvert.model;

import lombok.Data;

@Data
public class HwpParagraph {
    private String text = "";
    private int charShapeId;
    private int paraShapeId;
    private TextAlignment alignment = TextAlignment.LEFT;
    private float fontSize = 10f;
    private boolean bold = false;
    private boolean italic = false;
    private boolean underline = false;

    public enum TextAlignment {
        LEFT, CENTER, RIGHT, JUSTIFY
    }
}

// HwpTable.java
package com.fw.cmm.hwpconvert.model;

import lombok.Data;
import java.util.List;

@Data
public class HwpTable {
    private int rowCount;
    private int colCount;
    private List<List<String>> cellData;
    private float[] columnWidths;
}

// HwpImage.java
package com.fw.cmm.hwpconvert.model;

import lombok.Data;

@Data
public class HwpImage {
    private byte[] data;
    private String format;  // PNG, JPEG, BMP, etc.
    private float width;
    private float height;
    private float x;
    private float y;
}
```

### 4.3 컨트롤러 통합

```java
package com.fw.cmm.pdfviewer;

import com.fw.cmm.hwpconvert.HwpToPdfConversionService;
import org.springframework.web.bind.annotation.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

import java.util.Base64;
import java.util.HashMap;
import java.util.Map;

@Slf4j
@RestController
@RequestMapping("/api/pdfviewer")
@RequiredArgsConstructor
public class PdfViewerController {

    private final PdfViewerService pdfViewerService;
    private final HwpToPdfConversionService hwpConversionService;

    @PostMapping("/getData.do")
    public Map<String, Object> getData(@RequestBody Map<String, Object> params) {
        Map<String, Object> result = new HashMap<>();

        try {
            String atchFileNo = (String) params.get("ATCH_FILE_NO");
            String sn = String.valueOf(params.get("SN"));

            // 파일 정보 조회
            Map<String, Object> fileInfo = pdfViewerService.getFileInfo(atchFileNo, sn);
            String fileName = (String) fileInfo.get("FILE_NM");
            byte[] fileData = (byte[]) fileInfo.get("FILE_DATA");

            // HWP 파일인 경우 PDF로 변환
            if (hwpConversionService.canConvert(fileName)) {
                log.info("HWP 파일 감지, PDF 변환 시작: {}", fileName);

                fileData = hwpConversionService.convertToPdf(
                    new java.io.ByteArrayInputStream(fileData)
                );
                fileName = fileName.replaceAll("\\.(hwp|hwpx)$", ".pdf");

                log.info("HWP to PDF 변환 완료: {}", fileName);
            }

            // Base64 인코딩
            String base64Data = Base64.getEncoder().encodeToString(fileData);

            result.put("success", true);
            result.put("fileData", base64Data);
            result.put("fileName", fileName);

        } catch (Exception e) {
            log.error("파일 데이터 조회 실패", e);
            result.put("success", false);
            result.put("message", e.getMessage());
        }

        return result;
    }
}
```

---

## 5. 설정

### 5.1 application.yml

```yaml
# HWP 변환 설정
hwp:
  convert:
    enabled: true
    font:
      # 커스텀 폰트 경로 (비어있으면 시스템 폰트 자동 탐색)
      path: ""
      # 기본 폰트 이름
      name: malgun
    # 임시 파일 디렉토리
    temp-dir: ${java.io.tmpdir}/hwp-convert
    # 최대 파일 크기 (MB)
    max-file-size: 50
    # 변환 타임아웃 (초)
    timeout: 60
```

---

## 6. 제한사항 및 미지원 기능

### 6.1 완전 지원
| 기능 | 지원 상태 | 비고 |
|------|----------|------|
| 일반 텍스트 | ✅ 지원 | |
| 문단 정렬 | ✅ 지원 | 좌/중/우/양쪽 정렬 |
| 기본 폰트 | ✅ 지원 | 크기, 굵기, 기울임 |
| 단순 표 | ✅ 지원 | 기본 테두리 |
| 이미지 삽입 | ✅ 지원 | PNG, JPEG, BMP |

### 6.2 부분 지원
| 기능 | 지원 상태 | 비고 |
|------|----------|------|
| 복잡한 표 | ⚠️ 부분 | 셀 병합 제한적 |
| 글머리 기호 | ⚠️ 부분 | 텍스트로 변환 |
| 각주/미주 | ⚠️ 부분 | 본문에 포함 |
| 머리말/꼬리말 | ⚠️ 부분 | |

### 6.3 미지원
| 기능 | 지원 상태 | 대안 |
|------|----------|------|
| 암호화된 HWP | ❌ 미지원 | 암호 해제 후 변환 |
| 도형 | ❌ 미지원 | 이미지로 대체 권장 |
| 수식 | ❌ 미지원 | 이미지로 대체 권장 |
| OLE 객체 | ❌ 미지원 | |
| 양식 필드 | ❌ 미지원 | |
| 다단 레이아웃 | ❌ 미지원 | 단일 단으로 변환 |
| 세로쓰기 | ❌ 미지원 | |

---

## 7. 구현 일정 (예상)

### Phase 1: 기본 구조 (1주)
- [ ] 프로젝트 구조 생성
- [ ] 의존성 설정
- [ ] 기본 클래스 스켈레톤 작성
- [ ] 단위 테스트 환경 구성

### Phase 2: 텍스트 변환 (1주)
- [ ] HWP 파일 파싱 구현
- [ ] 텍스트 추출 구현
- [ ] PDF 텍스트 렌더링 구현
- [ ] 한글 폰트 처리 구현

### Phase 3: 표 및 이미지 (1주)
- [ ] 표 추출 및 렌더링
- [ ] 이미지 추출 및 삽입
- [ ] 페이지 레이아웃 처리

### Phase 4: 통합 및 테스트 (1주)
- [ ] 기존 PDF 뷰어와 통합
- [ ] 다양한 HWP 파일 테스트
- [ ] 성능 최적화
- [ ] 에러 처리 강화

---

## 8. 테스트 계획

### 8.1 단위 테스트

```java
@SpringBootTest
class HwpToPdfConversionServiceTest {

    @Autowired
    private HwpToPdfConversionService conversionService;

    @Test
    void 텍스트만_있는_HWP_변환() throws Exception {
        // given
        InputStream hwpStream = getClass().getResourceAsStream("/test/text-only.hwp");

        // when
        byte[] pdfBytes = conversionService.convertToPdf(hwpStream);

        // then
        assertThat(pdfBytes).isNotNull();
        assertThat(pdfBytes.length).isGreaterThan(0);
        assertThat(pdfBytes).startsWith(new byte[]{'%', 'P', 'D', 'F'});
    }

    @Test
    void 표가_있는_HWP_변환() throws Exception {
        // given
        InputStream hwpStream = getClass().getResourceAsStream("/test/with-table.hwp");

        // when
        byte[] pdfBytes = conversionService.convertToPdf(hwpStream);

        // then
        assertThat(pdfBytes).isNotNull();
    }

    @Test
    void 이미지가_있는_HWP_변환() throws Exception {
        // given
        InputStream hwpStream = getClass().getResourceAsStream("/test/with-image.hwp");

        // when
        byte[] pdfBytes = conversionService.convertToPdf(hwpStream);

        // then
        assertThat(pdfBytes).isNotNull();
    }

    @Test
    void 지원하지_않는_확장자() {
        // given
        String fileName = "document.docx";

        // when
        boolean canConvert = conversionService.canConvert(fileName);

        // then
        assertThat(canConvert).isFalse();
    }
}
```

### 8.2 통합 테스트

```java
@SpringBootTest
@AutoConfigureMockMvc
class PdfViewerControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void HWP_파일_PDF_변환_API_테스트() throws Exception {
        // given
        Map<String, Object> params = Map.of(
            "ATCH_FILE_NO", "TEST001",
            "SN", "1"
        );

        // when & then
        mockMvc.perform(post("/api/pdfviewer/getData.do")
                .contentType(MediaType.APPLICATION_JSON)
                .content(new ObjectMapper().writeValueAsString(params)))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.success").value(true))
            .andExpect(jsonPath("$.fileData").isNotEmpty());
    }
}
```

---

## 9. 참고 자료

### 9.1 공식 문서
- [hwplib GitHub](https://github.com/neolord0/hwplib)
- [hwpxlib GitHub](https://github.com/neolord0/hwpxlib)
- [OpenPDF GitHub](https://github.com/LibrePDF/OpenPDF)
- [한글 문서 파일 구조 5.0](https://www.hancom.com/etc/hwpDownload.do)

### 9.2 관련 프로젝트
- [hwp.js](https://github.com/nicknisi/hwp.js) - JavaScript HWP 뷰어
- [hwp2hwpx](https://github.com/nicknisi/hwp2hwpx) - HWP to HWPX 변환기

### 9.3 튜토리얼
- [OpenPDF 튜토리얼](https://www.netjstech.com/2018/10/how-to-create-pdf-in-java-using-openpdf.html)
- [Creating PDF Files in Java | Baeldung](https://www.baeldung.com/java-pdf-creation)

---

## 10. 결론

### 10.1 장점
- **완전한 제어**: 변환 로직을 완전히 제어 가능
- **무료**: 모든 라이브러리가 오픈소스 (Apache 2.0, LGPL/MPL)
- **서버 설치 불필요**: LibreOffice 같은 외부 프로그램 불필요
- **커스터마이징**: 필요에 따라 변환 로직 수정 가능

### 10.2 단점
- **구현 복잡도**: 상당한 개발 노력 필요
- **제한된 기능**: 복잡한 HWP 기능(도형, 수식 등) 미지원
- **품질**: 상용 솔루션 대비 변환 품질이 낮을 수 있음
- **유지보수**: HWP 파일 형식 변경 시 대응 필요

### 10.3 권장 사용 케이스
- 텍스트 위주의 단순한 HWP 문서
- 기본 표와 이미지만 포함된 문서
- 변환 품질보다 비용이 중요한 경우
- 외부 의존성을 최소화해야 하는 경우

---

## 11. 변환 품질 향상 방안

hwplib + OpenPDF 직접 변환 방식의 품질을 높이기 위한 다양한 전략을 제시합니다.

### 11.1 방안 1: HTML 중간 변환 방식 (권장)

HWP → HTML → PDF 파이프라인을 통해 레이아웃 품질을 대폭 향상시킬 수 있습니다.

#### 아키텍처

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   HWP 파일  │ ──▶ │  hwplib     │ ──▶ │   HTML/CSS  │ ──▶ │    PDF      │
│             │     │  파싱       │     │   생성      │     │  (고품질)   │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
                                              │
                                              ▼
                                    ┌─────────────────────┐
                                    │  Flying Saucer 또는 │
                                    │  OpenHTMLToPDF      │
                                    └─────────────────────┘
```

#### 추가 의존성

```gradle
dependencies {
    // 기존 의존성
    implementation 'kr.dogfoot:hwplib:1.1.1'

    // HTML to PDF (Flying Saucer + OpenPDF)
    implementation 'org.xhtmlrenderer:flying-saucer-pdf:9.7.2'

    // 또는 OpenHTMLToPDF (PDFBox 기반, 더 현대적)
    implementation 'com.openhtmltopdf:openhtmltopdf-pdfbox:1.0.10'
    implementation 'com.openhtmltopdf:openhtmltopdf-svg-support:1.0.10'
}
```

#### HwpToHtmlConverter 구현

```java
package com.fw.cmm.hwpconvert.converter;

import kr.dogfoot.hwplib.object.HWPFile;
import kr.dogfoot.hwplib.object.bodytext.Section;
import kr.dogfoot.hwplib.object.bodytext.paragraph.Paragraph;
import kr.dogfoot.hwplib.object.docinfo.CharShape;
import kr.dogfoot.hwplib.object.docinfo.ParaShape;
import kr.dogfoot.hwplib.object.docinfo.FaceName;

import org.springframework.stereotype.Component;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@Component
public class HwpToHtmlConverter {

    /**
     * HWP 파일을 HTML로 변환
     */
    public String convertToHtml(HWPFile hwpFile) {
        StringBuilder html = new StringBuilder();

        // HTML 헤더 및 스타일 생성
        html.append(generateHtmlHeader(hwpFile));

        // 본문 시작
        html.append("<body>\n");
        html.append("<div class=\"document\">\n");

        // 섹션별 처리
        for (Section section : hwpFile.getBodyText().getSectionList()) {
            html.append(convertSection(section, hwpFile));
        }

        html.append("</div>\n");
        html.append("</body>\n</html>");

        return html.toString();
    }

    /**
     * HTML 헤더 및 CSS 스타일 생성
     */
    private String generateHtmlHeader(HWPFile hwpFile) {
        StringBuilder header = new StringBuilder();
        header.append("<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n");
        header.append("<!DOCTYPE html PUBLIC \"-//W3C//DTD XHTML 1.0 Strict//EN\" ");
        header.append("\"http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd\">\n");
        header.append("<html xmlns=\"http://www.w3.org/1999/xhtml\">\n");
        header.append("<head>\n");
        header.append("<meta http-equiv=\"Content-Type\" content=\"text/html; charset=UTF-8\"/>\n");
        header.append("<style type=\"text/css\">\n");

        // 기본 스타일
        header.append("body { font-family: 'Malgun Gothic', '맑은 고딕', sans-serif; }\n");
        header.append(".document { width: 210mm; margin: 0 auto; padding: 20mm; }\n");
        header.append("p { margin: 0; padding: 0; line-height: 1.6; }\n");
        header.append("table { border-collapse: collapse; width: 100%; }\n");
        header.append("td, th { border: 1px solid #000; padding: 5px; }\n");

        // CharShape 기반 스타일 클래스 생성
        header.append(generateCharShapeStyles(hwpFile));

        // ParaShape 기반 스타일 클래스 생성
        header.append(generateParaShapeStyles(hwpFile));

        header.append("</style>\n");
        header.append("</head>\n");

        return header.toString();
    }

    /**
     * CharShape(글자 모양)에서 CSS 스타일 생성
     */
    private String generateCharShapeStyles(HWPFile hwpFile) {
        StringBuilder styles = new StringBuilder();

        if (hwpFile.getDocInfo() == null || hwpFile.getDocInfo().getCharShapeList() == null) {
            return "";
        }

        int index = 0;
        for (CharShape charShape : hwpFile.getDocInfo().getCharShapeList()) {
            styles.append(String.format(".cs%d {\n", index));

            // 폰트 크기 (HWP 단위: 1/100 pt)
            float fontSize = charShape.getBaseSize() / 100.0f;
            styles.append(String.format("  font-size: %.1fpt;\n", fontSize));

            // 폰트 패밀리
            int fontId = charShape.getFaceNameIds().getHangul();
            String fontName = getFontName(hwpFile, fontId);
            if (fontName != null) {
                styles.append(String.format("  font-family: '%s', sans-serif;\n", fontName));
            }

            // 굵기
            if (charShape.isBold()) {
                styles.append("  font-weight: bold;\n");
            }

            // 기울임
            if (charShape.isItalic()) {
                styles.append("  font-style: italic;\n");
            }

            // 밑줄
            if (charShape.getUnderLineSort() != null &&
                charShape.getUnderLineSort().getValue() > 0) {
                styles.append("  text-decoration: underline;\n");
            }

            // 취소선
            if (charShape.getStrikeLineSort() != null &&
                charShape.getStrikeLineSort().getValue() > 0) {
                styles.append("  text-decoration: line-through;\n");
            }

            // 글자 색상
            long textColor = charShape.getTextColor().getValue();
            if (textColor != 0) {
                String colorHex = String.format("#%06X", textColor & 0xFFFFFF);
                styles.append(String.format("  color: %s;\n", colorHex));
            }

            // 장평 (글자 폭 비율)
            int charRatio = charShape.getCharSpaceRatios().getHangul();
            if (charRatio != 100) {
                styles.append(String.format("  transform: scaleX(%.2f);\n", charRatio / 100.0));
            }

            styles.append("}\n");
            index++;
        }

        return styles.toString();
    }

    /**
     * ParaShape(문단 모양)에서 CSS 스타일 생성
     */
    private String generateParaShapeStyles(HWPFile hwpFile) {
        StringBuilder styles = new StringBuilder();

        if (hwpFile.getDocInfo() == null || hwpFile.getDocInfo().getParaShapeList() == null) {
            return "";
        }

        int index = 0;
        for (ParaShape paraShape : hwpFile.getDocInfo().getParaShapeList()) {
            styles.append(String.format(".ps%d {\n", index));

            // 정렬 (0: 양쪽, 1: 왼쪽, 2: 오른쪽, 3: 가운데)
            switch (paraShape.getProperty1().getAlignmentType().getValue()) {
                case 1:
                    styles.append("  text-align: left;\n");
                    break;
                case 2:
                    styles.append("  text-align: right;\n");
                    break;
                case 3:
                    styles.append("  text-align: center;\n");
                    break;
                default:
                    styles.append("  text-align: justify;\n");
            }

            // 들여쓰기 (HWP 단위 → pt)
            long indent = paraShape.getIndent();
            if (indent > 0) {
                styles.append(String.format("  text-indent: %.1fpt;\n", hwpUnitToPoint(indent)));
            }

            // 왼쪽 여백
            long leftMargin = paraShape.getLeftMargin();
            if (leftMargin > 0) {
                styles.append(String.format("  margin-left: %.1fpt;\n", hwpUnitToPoint(leftMargin)));
            }

            // 오른쪽 여백
            long rightMargin = paraShape.getRightMargin();
            if (rightMargin > 0) {
                styles.append(String.format("  margin-right: %.1fpt;\n", hwpUnitToPoint(rightMargin)));
            }

            // 문단 앞 간격
            long spaceBefore = paraShape.getSpaceParaUp();
            if (spaceBefore > 0) {
                styles.append(String.format("  margin-top: %.1fpt;\n", hwpUnitToPoint(spaceBefore)));
            }

            // 문단 뒤 간격
            long spaceAfter = paraShape.getSpaceParaDown();
            if (spaceAfter > 0) {
                styles.append(String.format("  margin-bottom: %.1fpt;\n", hwpUnitToPoint(spaceAfter)));
            }

            // 줄 간격 (%)
            int lineSpacing = paraShape.getLineSpace();
            if (lineSpacing > 0) {
                styles.append(String.format("  line-height: %.1f%%;\n", lineSpacing * 1.0));
            }

            styles.append("}\n");
            index++;
        }

        return styles.toString();
    }

    /**
     * 폰트 이름 조회
     */
    private String getFontName(HWPFile hwpFile, int fontId) {
        try {
            if (hwpFile.getDocInfo() != null &&
                hwpFile.getDocInfo().getHangulFaceNameList() != null &&
                fontId < hwpFile.getDocInfo().getHangulFaceNameList().size()) {

                FaceName faceName = hwpFile.getDocInfo().getHangulFaceNameList().get(fontId);
                return faceName.getName();
            }
        } catch (Exception e) {
            log.warn("폰트 이름 조회 실패: fontId={}", fontId);
        }
        return null;
    }

    /**
     * HWP 단위를 포인트로 변환
     */
    private float hwpUnitToPoint(long hwpUnit) {
        return (float) (hwpUnit * 72.0 / 7200.0);
    }

    /**
     * 섹션을 HTML로 변환
     */
    private String convertSection(Section section, HWPFile hwpFile) {
        StringBuilder html = new StringBuilder();
        html.append("<div class=\"section\">\n");

        for (Paragraph para : section.getParagraphs()) {
            html.append(convertParagraph(para, hwpFile));
        }

        html.append("</div>\n");
        return html.toString();
    }

    /**
     * 문단을 HTML로 변환
     */
    private String convertParagraph(Paragraph para, HWPFile hwpFile) {
        StringBuilder html = new StringBuilder();

        // 문단 스타일 클래스
        int paraShapeId = para.getHeader().getParaShapeId();
        html.append(String.format("<p class=\"ps%d\">\n", paraShapeId));

        // 텍스트와 글자 스타일 처리
        if (para.getText() != null) {
            html.append(convertTextWithStyles(para, hwpFile));
        }

        html.append("</p>\n");
        return html.toString();
    }

    /**
     * 텍스트를 스타일과 함께 HTML로 변환
     */
    private String convertTextWithStyles(Paragraph para, HWPFile hwpFile) {
        StringBuilder html = new StringBuilder();

        String text = para.getText().getNormalString(0);
        if (text == null || text.isEmpty()) {
            return "";
        }

        // 글자 모양 적용 (간단 버전: 첫 번째 스타일만 적용)
        if (para.getCharShape() != null &&
            para.getCharShape().getPositonShapeIdPairList() != null &&
            !para.getCharShape().getPositonShapeIdPairList().isEmpty()) {

            int charShapeId = (int) para.getCharShape()
                .getPositonShapeIdPairList().get(0).getShapeId();

            html.append(String.format("<span class=\"cs%d\">", charShapeId));
            html.append(escapeHtml(text));
            html.append("</span>");
        } else {
            html.append(escapeHtml(text));
        }

        return html.toString();
    }

    /**
     * HTML 이스케이프
     */
    private String escapeHtml(String text) {
        return text
            .replace("&", "&amp;")
            .replace("<", "&lt;")
            .replace(">", "&gt;")
            .replace("\"", "&quot;")
            .replace("'", "&#39;");
    }
}
```

#### Flying Saucer를 이용한 PDF 생성

```java
package com.fw.cmm.hwpconvert.converter;

import org.xhtmlrenderer.pdf.ITextRenderer;
import com.lowagie.text.pdf.BaseFont;

import org.springframework.stereotype.Component;
import lombok.extern.slf4j.Slf4j;

import java.io.ByteArrayOutputStream;

@Slf4j
@Component
public class HtmlToPdfConverter {

    /**
     * HTML을 PDF로 변환 (Flying Saucer 사용)
     */
    public byte[] convertToPdf(String html) throws Exception {
        ByteArrayOutputStream outputStream = new ByteArrayOutputStream();

        ITextRenderer renderer = new ITextRenderer();

        // 한글 폰트 등록
        registerKoreanFonts(renderer);

        // HTML 설정 및 렌더링
        renderer.setDocumentFromString(html);
        renderer.layout();
        renderer.createPDF(outputStream);

        return outputStream.toByteArray();
    }

    /**
     * 한글 폰트 등록
     */
    private void registerKoreanFonts(ITextRenderer renderer) {
        try {
            // Windows 폰트
            renderer.getFontResolver().addFont(
                "C:/Windows/Fonts/malgun.ttf",
                BaseFont.IDENTITY_H,
                BaseFont.EMBEDDED
            );
            renderer.getFontResolver().addFont(
                "C:/Windows/Fonts/malgunbd.ttf",
                BaseFont.IDENTITY_H,
                BaseFont.EMBEDDED
            );

            // Linux 폰트
            renderer.getFontResolver().addFont(
                "/usr/share/fonts/truetype/nanum/NanumGothic.ttf",
                BaseFont.IDENTITY_H,
                BaseFont.EMBEDDED
            );
        } catch (Exception e) {
            log.warn("폰트 등록 실패", e);
        }
    }
}
```

#### OpenHTMLToPDF를 이용한 PDF 생성 (대안)

```java
package com.fw.cmm.hwpconvert.converter;

import com.openhtmltopdf.pdfboxout.PdfRendererBuilder;

import org.springframework.stereotype.Component;
import lombok.extern.slf4j.Slf4j;

import java.io.ByteArrayOutputStream;
import java.io.File;

@Slf4j
@Component
public class OpenHtmlToPdfConverter {

    /**
     * HTML을 PDF로 변환 (OpenHTMLToPDF 사용)
     */
    public byte[] convertToPdf(String html) throws Exception {
        ByteArrayOutputStream outputStream = new ByteArrayOutputStream();

        PdfRendererBuilder builder = new PdfRendererBuilder();

        // 한글 폰트 등록
        registerKoreanFonts(builder);

        // HTML 설정
        builder.withHtmlContent(html, "/");
        builder.toStream(outputStream);
        builder.run();

        return outputStream.toByteArray();
    }

    /**
     * 한글 폰트 등록
     */
    private void registerKoreanFonts(PdfRendererBuilder builder) {
        try {
            File malgunFont = new File("C:/Windows/Fonts/malgun.ttf");
            if (malgunFont.exists()) {
                builder.useFont(malgunFont, "Malgun Gothic");
            }

            File nanumFont = new File("/usr/share/fonts/truetype/nanum/NanumGothic.ttf");
            if (nanumFont.exists()) {
                builder.useFont(nanumFont, "NanumGothic");
            }
        } catch (Exception e) {
            log.warn("폰트 등록 실패", e);
        }
    }
}
```

---

### 11.2 방안 2: 정밀한 CharShape/ParaShape 매핑

hwplib에서 제공하는 스타일 정보를 완벽하게 PDF로 매핑합니다.

#### 향상된 스타일 추출기

```java
package com.fw.cmm.hwpconvert.extractor;

import kr.dogfoot.hwplib.object.HWPFile;
import kr.dogfoot.hwplib.object.docinfo.CharShape;
import kr.dogfoot.hwplib.object.docinfo.ParaShape;
import kr.dogfoot.hwplib.object.docinfo.charshape.*;

import com.fw.cmm.hwpconvert.model.HwpCharStyle;
import com.fw.cmm.hwpconvert.model.HwpParaStyle;

import org.springframework.stereotype.Component;
import lombok.extern.slf4j.Slf4j;

import java.awt.Color;
import java.util.HashMap;
import java.util.Map;

@Slf4j
@Component
public class HwpStyleExtractor {

    /**
     * 글자 모양 스타일 추출
     */
    public Map<Integer, HwpCharStyle> extractCharStyles(HWPFile hwpFile) {
        Map<Integer, HwpCharStyle> styles = new HashMap<>();

        if (hwpFile.getDocInfo() == null ||
            hwpFile.getDocInfo().getCharShapeList() == null) {
            return styles;
        }

        int index = 0;
        for (CharShape charShape : hwpFile.getDocInfo().getCharShapeList()) {
            HwpCharStyle style = new HwpCharStyle();

            // 폰트 크기 (1/100 pt → pt)
            style.setFontSize(charShape.getBaseSize() / 100.0f);

            // 폰트 이름
            int fontId = charShape.getFaceNameIds().getHangul();
            style.setFontName(getFontName(hwpFile, fontId));

            // 굵기
            style.setBold(charShape.isBold());

            // 기울임
            style.setItalic(charShape.isItalic());

            // 밑줄
            UnderLineSort underLine = charShape.getUnderLineSort();
            style.setUnderline(underLine != null && underLine.getValue() > 0);

            // 취소선
            StrikeLineSort strikeLine = charShape.getStrikeLineSort();
            style.setStrikethrough(strikeLine != null && strikeLine.getValue() > 0);

            // 글자 색상
            long colorValue = charShape.getTextColor().getValue();
            style.setTextColor(hwpColorToAwtColor(colorValue));

            // 장평 (글자 폭 비율)
            style.setCharWidthRatio(charShape.getCharSpaceRatios().getHangul() / 100.0f);

            // 자간 (글자 간격)
            style.setCharSpacing(charShape.getCharSpacing());

            // 위/아래 첨자 (0: 없음, 1: 위 첨자, 2: 아래 첨자)
            int subSuperScript = charShape.getProperty().getSubSuperScript().getValue();
            style.setSuperscript(subSuperScript == 1);
            style.setSubscript(subSuperScript == 2);

            // 외곽선, 그림자, 강조점
            style.setOutline(charShape.getOutterLineSort() != null &&
                charShape.getOutterLineSort().getValue() > 0);
            style.setShadow(charShape.getShadowSort() != null &&
                charShape.getShadowSort().getValue() > 0);
            style.setEmphasis(charShape.getEmphasisSort() != null &&
                charShape.getEmphasisSort().getValue() > 0);

            styles.put(index, style);
            index++;
        }

        return styles;
    }

    /**
     * HWP 색상을 AWT Color로 변환
     */
    private Color hwpColorToAwtColor(long hwpColor) {
        int r = (int) (hwpColor & 0xFF);
        int g = (int) ((hwpColor >> 8) & 0xFF);
        int b = (int) ((hwpColor >> 16) & 0xFF);
        return new Color(r, g, b);
    }

    private String getFontName(HWPFile hwpFile, int fontId) {
        try {
            if (hwpFile.getDocInfo().getHangulFaceNameList() != null &&
                fontId < hwpFile.getDocInfo().getHangulFaceNameList().size()) {
                return hwpFile.getDocInfo().getHangulFaceNameList().get(fontId).getName();
            }
        } catch (Exception e) {
            log.warn("폰트 이름 조회 실패: fontId={}", fontId);
        }
        return "맑은 고딕";
    }
}
```

---

### 11.3 방안 3: 정밀한 표(Table) 렌더링

#### 표 셀 병합 및 스타일 지원

```java
package com.fw.cmm.hwpconvert.extractor;

import kr.dogfoot.hwplib.object.bodytext.control.ControlTable;
import kr.dogfoot.hwplib.object.bodytext.control.table.Row;
import kr.dogfoot.hwplib.object.bodytext.control.table.Cell;
import kr.dogfoot.hwplib.object.bodytext.paragraph.Paragraph;

import com.fw.cmm.hwpconvert.model.HwpTable;
import com.fw.cmm.hwpconvert.model.HwpTableCell;

import org.springframework.stereotype.Component;
import lombok.extern.slf4j.Slf4j;

import java.util.ArrayList;
import java.util.List;

@Slf4j
@Component
public class HwpTableExtractor {

    /**
     * 표를 상세하게 추출
     */
    public HwpTable extractTable(ControlTable controlTable) {
        HwpTable table = new HwpTable();

        try {
            table.setRowCount(controlTable.getTable().getRowCount());
            table.setColCount(controlTable.getTable().getColCount());

            List<List<HwpTableCell>> rows = new ArrayList<>();

            for (Row row : controlTable.getRowList()) {
                List<HwpTableCell> rowCells = new ArrayList<>();

                for (Cell cell : row.getCellList()) {
                    HwpTableCell tableCell = new HwpTableCell();

                    // 셀 병합 정보
                    tableCell.setColSpan(cell.getListHeader().getColSpan());
                    tableCell.setRowSpan(cell.getListHeader().getRowSpan());

                    // 셀 크기
                    tableCell.setWidth(hwpUnitToPoint(cell.getListHeader().getWidth()));
                    tableCell.setHeight(hwpUnitToPoint(cell.getListHeader().getHeight()));

                    // 셀 내 텍스트 추출
                    StringBuilder text = new StringBuilder();
                    for (Paragraph para : cell.getParagraphList()) {
                        if (para.getText() != null) {
                            String paraText = para.getText().getNormalString(0);
                            if (paraText != null) {
                                if (text.length() > 0) text.append("\n");
                                text.append(paraText);
                            }
                        }
                    }
                    tableCell.setText(text.toString());

                    rowCells.add(tableCell);
                }
                rows.add(rowCells);
            }

            table.setCells(rows);

        } catch (Exception e) {
            log.error("표 추출 실패", e);
        }

        return table;
    }

    private float hwpUnitToPoint(long hwpUnit) {
        return (float) (hwpUnit * 72.0 / 7200.0);
    }
}
```

---

### 11.4 방안 4: 다중 폰트 지원

#### 폰트 매핑 및 폴백

```java
package com.fw.cmm.hwpconvert.font;

import com.lowagie.text.Font;
import com.lowagie.text.pdf.BaseFont;

import org.springframework.stereotype.Component;
import lombok.extern.slf4j.Slf4j;

import java.io.File;
import java.util.HashMap;
import java.util.Map;

@Slf4j
@Component
public class MultiFontManager {

    // HWP 폰트 이름 → 시스템 폰트 경로 매핑
    private static final Map<String, String[]> FONT_MAPPING = new HashMap<>();

    static {
        FONT_MAPPING.put("맑은 고딕", new String[]{
            "C:/Windows/Fonts/malgun.ttf",
            "/usr/share/fonts/truetype/nanum/NanumGothic.ttf"
        });
        FONT_MAPPING.put("함초롬돋움", new String[]{"C:/Windows/Fonts/H2GTRM.TTF"});
        FONT_MAPPING.put("함초롬바탕", new String[]{"C:/Windows/Fonts/H2MJSM.TTF"});
        FONT_MAPPING.put("바탕", new String[]{"C:/Windows/Fonts/batang.ttc"});
        FONT_MAPPING.put("돋움", new String[]{"C:/Windows/Fonts/dotum.ttc"});
        FONT_MAPPING.put("굴림", new String[]{"C:/Windows/Fonts/gulim.ttc"});
        FONT_MAPPING.put("나눔고딕", new String[]{
            "C:/Windows/Fonts/NanumGothic.ttf",
            "/usr/share/fonts/truetype/nanum/NanumGothic.ttf"
        });
        FONT_MAPPING.put("나눔명조", new String[]{
            "C:/Windows/Fonts/NanumMyeongjo.ttf",
            "/usr/share/fonts/truetype/nanum/NanumMyeongjo.ttf"
        });
    }

    private Map<String, BaseFont> loadedFonts = new HashMap<>();
    private BaseFont fallbackFont;

    /**
     * HWP 폰트 이름으로 PDF 폰트 가져오기
     */
    public Font getFont(String hwpFontName, float size, int style) {
        BaseFont baseFont = getBaseFont(hwpFontName);
        return new Font(baseFont, size, style);
    }

    private BaseFont getBaseFont(String hwpFontName) {
        if (loadedFonts.containsKey(hwpFontName)) {
            return loadedFonts.get(hwpFontName);
        }

        String[] paths = FONT_MAPPING.get(hwpFontName);
        if (paths != null) {
            for (String path : paths) {
                if (new File(path).exists()) {
                    try {
                        BaseFont baseFont = BaseFont.createFont(
                            path, BaseFont.IDENTITY_H, BaseFont.EMBEDDED);
                        loadedFonts.put(hwpFontName, baseFont);
                        return baseFont;
                    } catch (Exception e) {
                        log.warn("폰트 로드 실패: {}", path);
                    }
                }
            }
        }
        return fallbackFont;
    }
}
```

---

### 11.5 품질 향상 방안 비교표

| 방안 | 품질 향상 | 구현 복잡도 | 추가 의존성 | 권장 우선순위 |
|------|----------|------------|------------|--------------|
| **HTML 중간 변환** | ★★★★★ | 중간 | Flying Saucer / OpenHTMLToPDF | **1순위** |
| **CharShape/ParaShape 정밀 매핑** | ★★★★☆ | 높음 | 없음 | 2순위 |
| **표 셀 병합 및 스타일** | ★★★☆☆ | 중간 | 없음 | 3순위 |
| **다중 폰트 지원** | ★★★★☆ | 낮음 | 없음 | 2순위 |

---

### 11.6 권장 구현 순서

1. **Phase 1**: HTML 중간 변환 방식 구현 (가장 큰 품질 향상)
2. **Phase 2**: 다중 폰트 지원 추가
3. **Phase 3**: CharShape/ParaShape 정밀 매핑
4. **Phase 4**: 표 셀 병합 및 스타일 지원

---

### 11.7 예상 품질 향상 효과

| 항목 | 기본 구현 | 품질 향상 적용 후 |
|------|----------|------------------|
| 텍스트 레이아웃 | 기본 | 원본과 유사 |
| 폰트 스타일 | 크기/굵기만 | 대부분 스타일 지원 |
| 표 렌더링 | 단순 표만 | 셀 병합, 스타일 지원 |
| 문단 서식 | 기본 | 들여쓰기, 간격 등 |
| **전체 충실도** | **60~70%** | **85~95%** |

---

## 12. 참고 자료 (추가)

### 12.1 HTML to PDF 라이브러리
- [Flying Saucer GitHub](https://github.com/flyingsaucerproject/flyingsaucer)
- [OpenHTMLToPDF GitHub](https://github.com/openhtmltopdf/openhtmltopdf)
- [Flying Saucer + OpenPDF 튜토리얼](https://www.netjstech.com/2021/02/html-to-pdf-java-flying-saucer-openpdf.html)

### 12.2 hwplib 스타일 관련
- [hwplib CharShape 샘플](https://jar-download.com/artifacts/kr.dogfoot/hwplib/1.0.1/source-code/kr/dogfoot/hwplib/sample/Inserting_CharShape.java)
- [LibreOffice HWP Filter 참고](https://docs.libreoffice.org/hwpfilter/html/structCharShape.html)

### 12.3 OpenPDF 고급 기능
- [OpenPDF Paragraph API](https://librepdf.github.io/OpenPDF/docs-1-2-7/com/lowagie/text/Paragraph.html)
- [iText 줄 간격 설정](https://kb.itextpdf.com/it5kb/how-to-change-the-line-spacing-of-text)

---

*문서 작성일: 2025-12-03*
*최종 수정일: 2025-12-03*
*작성자: Claude Code*
