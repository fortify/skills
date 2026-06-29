# XML External Entity (XXE) Injection

## Risks

| # | Risk | Detection signal | Fortify Category | CWE | OWASP 2025 | Action |
|---|------|-----------------|-----------------|-----|------------|--------|
| R1 | XML External Entity (XXE) injection | XML parser instantiated without DTD processing disabled; `DocumentBuilderFactory`, `SAXParserFactory`, `XmlReaderSettings`, or equivalent created with default settings and used to parse untrusted input | XML External Entity Injection | CWE-611 | A02 Security Misconfiguration | Actions 1, 2, 3 |
| R2 | XPath injection | User input concatenated into XPath expression string; `XPath.evaluate(userInput)` or equivalent called without parameterized API | XPath Injection | CWE-643 | A05 Injection | Action 4 |

## Required Agent Actions

1. **Disable DTD processing entirely** *(R1)* — the safest mitigation. Most parsers support a flag to prevent DTD loading:
   - Java (DocumentBuilderFactory): `factory.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true)`
   - .NET (XmlReaderSettings): `settings.DtdProcessing = DtdProcessing.Prohibit`
   - Python (lxml): use `resolve_entities=False`; avoid the standard `xml.etree` for untrusted input — use `defusedxml`
   - libxml2: set `XML_PARSE_NOENT` off and `XML_PARSE_NONET` on

2. **If DTDs cannot be disabled, disable external entity resolution and external DTD loading** *(R1)* — apply these settings separately:
   - `factory.setFeature("http://xml.org/sax/features/external-general-entities", false)`
   - `factory.setFeature("http://xml.org/sax/features/external-parameter-entities", false)`
   - `factory.setAttribute(XMLConstants.ACCESS_EXTERNAL_DTD, "")`

3. **Use a safe XML library** *(R1)* — in Python, use `defusedxml` instead of `xml.etree.ElementTree` for untrusted input. For Node.js, check parser defaults explicitly (most are not safe by default).

4. **Parameterize XPath queries** *(R2)* — if user input is used in XPath expressions, use a parameterized XPath API rather than string concatenation.

## Completion Evidence

- [ ] *(R1)* DTD processing disabled, or external entity resolution and external DTD loading explicitly disabled on all XML parsers handling untrusted input
- [ ] *(R1)* Safe library or parser configuration used (`defusedxml`, `DtdProcessing.Prohibit`, etc.)
- [ ] *(R2)* XPath queries use parameterized APIs, not string concatenation