# Iteration 5 — Document parsers

> Parent: [implementation-plan.md](../implementation-plan.md) · Previous: [iteration-04-guardrail-rules.md](./iteration-04-guardrail-rules.md)

## Goal

Parse PDF, DOCX, and plain text into raw text. Registry selects parser by filename. **Prefer:** Parser selection and supported types driven by registry/config (e.g. which parsers are registered, which file types a domain supports from YAML) — no hardcoded parser list or extensions in code.

## Deliverables

| # | Component | Location in this doc |
|---|-----------|----------------------|
| 1 | `DocumentParser` | § Code 6.1 |
| 2 | `DocumentParserRegistry` | § Code 6.2 |
| 3 | `PdfDocumentParser` | § Code 6.3 |
| 4 | `DocxDocumentParser` | § Code 6.4 |
| 5 | `PlainTextParser` | § Code 6.5 |

## Acceptance criteria

- Registry tries parsers in order; first that `supports(filename)` wins; `parse(filename, bytes)` returns extracted text or throws.
- PDF: PDFBox-based extraction; DOCX: POI-based; TXT: UTF-8. Empty or unparseable can return null or throw as per design.

## Tests to add

- `PdfDocumentParserTest`: parse a small test PDF from `src/test/resources`; expect non-empty text; unsupported extension returns false from `supports`.
- `DocxDocumentParserTest`: parse a small test DOCX; expect non-empty text.
- `PlainTextParserTest`: parse UTF-8 bytes; expect same string; supports `.txt`/`.md`.
- `DocumentParserRegistryTest`: register PDF, DOCX, TXT; for `file.pdf` get PDF parser and parse; for `file.xyz` no parser or fallback behavior.

## Quality gates

- Tests use only test resources; no network; build passes with optional `poi-ooxml` and PDFBox.

See [iteration-00-quality-gates.md](./iteration-00-quality-gates.md) for gates that apply to every iteration.

---

## Code — Document parsers (`feature/ingest/parser/`)

### Code 6.1 DocumentParser

```java
package com.example.rag.feature.ingest.parser;

public interface DocumentParser {

    boolean supports(String filename);

    String extractText(byte[] content);
}
```

### Code 6.2 DocumentParserRegistry

```java
package com.example.rag.feature.ingest.parser;

import java.util.List;
import org.springframework.stereotype.Component;

@Component
public class DocumentParserRegistry {

    private final List<DocumentParser> parsers;

    public DocumentParserRegistry(List<DocumentParser> parsers) {
        this.parsers = parsers;
    }

    public String parse(String filename, byte[] content) {
        for (var parser : parsers) {
            if (parser.supports(filename)) {
                return parser.extractText(content);
            }
        }
        return null;
    }
}
```

### Code 6.3 PdfDocumentParser

```java
package com.example.rag.feature.ingest.parser;

import java.io.IOException;
import org.apache.pdfbox.Loader;
import org.apache.pdfbox.text.PDFTextStripper;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

@Component
@Order(1)
public class PdfDocumentParser implements DocumentParser {

    private static final Logger log = LoggerFactory.getLogger(PdfDocumentParser.class);

    @Override
    public boolean supports(String filename) {
        return filename.toLowerCase().endsWith(".pdf");
    }

    @Override
    public String extractText(byte[] content) {
        try (var document = Loader.loadPDF(content)) {
            var stripper = new PDFTextStripper();
            return stripper.getText(document);
        } catch (IOException e) {
            log.warn("PDF parse failed: {}", e.getMessage());
            return null;
        }
    }
}
```

### Code 6.4 DocxDocumentParser

```java
package com.example.rag.feature.ingest.parser;

import java.io.ByteArrayInputStream;
import java.io.IOException;
import org.apache.poi.xwpf.extractor.XWPFWordExtractor;
import org.apache.poi.xwpf.usermodel.XWPFDocument;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

@Component
@Order(2)
public class DocxDocumentParser implements DocumentParser {

    private static final Logger log = LoggerFactory.getLogger(DocxDocumentParser.class);

    @Override
    public boolean supports(String filename) {
        return filename.toLowerCase().endsWith(".docx");
    }

    @Override
    public String extractText(byte[] content) {
        try (var doc = new XWPFDocument(new ByteArrayInputStream(content));
             var extractor = new XWPFWordExtractor(doc)) {
            return extractor.getText();
        } catch (IOException e) {
            log.warn("DOCX parse failed: {}", e.getMessage());
            return null;
        }
    }
}
```

### Code 6.5 PlainTextParser

```java
package com.example.rag.feature.ingest.parser;

import java.nio.charset.StandardCharsets;
import java.util.Set;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

@Component
@Order(10)
public class PlainTextParser implements DocumentParser {

    private static final Set<String> EXTENSIONS = Set.of(".txt", ".md", ".csv");

    @Override
    public boolean supports(String filename) {
        var lower = filename.toLowerCase();
        return EXTENSIONS.stream().anyMatch(lower::endsWith);
    }

    @Override
    public String extractText(byte[] content) {
        return new String(content, StandardCharsets.UTF_8);
    }
}
```
