# Creating config.yaml - Complete Guide

## Overview

The `config.yaml` file defines XPath mappings for extracting metadata from XML documents in different domains. Each domain has variants with specific XPath expressions to extract title, authors, keywords, and body content.

---

## File Location

```
metR-Infinity-backend/
└── config/
    └── config.yaml
```

---

## Basic Structure

```yaml
{DOMAIN}:
  {VARIANT}:
    title:
      - "/xpath/expression/1"
      - "/xpath/expression/2"
    authors:
      - "/xpath/to/author"
    createdDate:
      - "/xpath/to/date/@attribute"
    keywords:
      - "/xpath/to/keywords/*"
    prodinfo:
      - "/xpath/to/prodinfo"
    othermeta:
      - "/xpath/to/othermeta"
    body:
      - "/xpath/to/body"
```

---

## Step-by-Step Creation

### Step 1: Identify Your Domain

Choose from supported domains:
- `space` - Manufacturing (Furnace, Batch Plant, EHS)
- `Time` - Equipment tracking
- `Soul` - Discipline-specific
- `Reality` - Batch processing
- `Mind` - Capital programs
- `Power` - Revenue lifecycle

### Step 2: Identify Document Variants

Each domain can have multiple document variants. Examples:
- `FES-Space` (Furnace Energy Space)
- `BAT-Space` (Batch Plant Space)
- `batch_1`, `batch_3` (Reality variants)

### Step 3: Analyze XML Structure

Examine your XML document to identify element paths:

```xml
<topic>
  <title>Document Title</title>
  <prolog>
    <author>John Doe</author>
    <critdates>
      <created date="2024-01-15"/>
    </critdates>
    <metadata>
      <keywords>
        <keyword>keyword1</keyword>
        <keyword>keyword2</keyword>
      </keywords>
    </metadata>
  </prolog>
  <body>
    <section>Content here</section>
  </body>
</topic>
```

### Step 4: Write XPath Expressions

Convert XML paths to XPath:

| XML Element | XPath Expression |
|-------------|-----------------|
| `/topic/title` | `/topic/title` |
| `/topic/prolog/author` | `/topic/prolog/author` |
| `/topic/prolog/critdates/created/@date` | `/topic/prolog/critdates/created/@date` |
| `/topic/prolog/metadata/keywords/*` | `/topic/prolog/metadata/keywords/*` |
| `/topic/body` | `/topic/body` |

### Step 5: Add Fallback Expressions

Include multiple XPath expressions for flexibility:

```yaml
title:
  - "/topic/title"                    # Primary
  - "//title"                         # Fallback (any title)
  - "/document/heading"               # Alternative
```

---

## Complete Examples

### Example 1: SPACE Domain (FES-Space)

```yaml
space:
  FES-Space:
    title:
      - "/topic/title"
    authors:
      - "/topic/prolog/author"
    createdDate:
      - "/topic/prolog/critdates/created/@date"
    keywords:
      - "/topic/prolog/metadata/keywords/*"
    prodinfo:
      - "/topic/prolog/metadata/prodinfo"
    othermeta:
      - "/topic/prolog/metadata/othermeta"
    body:
      - "/topic/body"
```

### Example 2: REALITY Domain (Batch Processing)

```yaml
Reality:
  batch_1:
    title:
      - "/bookmap/booktitle/mainbooktitle/text()"
      - "//mainbooktitle/text()"
      - "/topic/title"
    authors:
      - "/bookmap/bookmeta/authorinformation/organizationinfo/namedetails/organizationnamedetails/organizationname/text()"
      - "//organizationname/text()"
      - "/topic/prolog/author"
    createdDate:
      - "/bookmap/bookmeta/bookid/edition/text()"
      - "//edition/text()"
      - "/topic/prolog/critdates/created/@date"
    keywords:
      - "/bookmap/booktitle/booktitlealt[text()]/text()"
      - "//booktitlealt[text()]/text()"
      - "/topic/prolog/metadata/keywords/*"
    prodinfo:
      - "/bookmap/bookmeta/bookid/bookpartno/text()"
      - "//bookpartno/text()"
      - "/topic/prolog/metadata/prodinfo"
    othermeta:
      - "/bookmap/bookmeta/bookrights/copyrlast/year/text()"
      - "//copyrlast/year/text()"
      - "/bookmap/bookmeta/publisherinformation/organization/text()"
      - "/topic/prolog/metadata/othermeta"
    body:
      - "/bookmap/chapter[@navtitle]/@navtitle"
      - "//chapter[@navtitle]/@navtitle"
      - "//topicref[@navtitle]/@navtitle"
      - "/topic/body"
```

### Example 3: POWER Domain (Revenue Lifecycle)

```yaml
Power:
  Power-Adobe_Sign_Services:
    title:
      - "/task/title"
      - "//title"
    authors:
      - "/task/prolog/author"
      - "//author"
    createdDate:
      - "/task/prolog/critdates/created/@date"
      - "//critdates/created/@date"
    keywords:
      - "/task/prolog/metadata/keywords/indexterm/text()"
      - "//keywords/indexterm/text()"
    prodinfo:
      - "/task/prolog/metadata/prodinfo"
      - "//prodinfo"
    othermeta:
      - "/task/prolog/metadata/othermeta"
      - "//othermeta"
    body:
      - "/task/taskbody"
      - "//taskbody"
```

---

## XPath Syntax Guide

### Basic Selectors

```xpath
/topic/title              # Absolute path
//title                   # Any title element
/topic/title/text()       # Text content
/topic/title/@id          # Attribute value
```

### Predicates

```xpath
/topic/prolog/author[1]   # First author
/topic/prolog/author[last()]  # Last author
/topic/prolog/author[@type='primary']  # With attribute
```

### Wildcards

```xpath
/topic/prolog/metadata/*  # All children
/topic/prolog/metadata/keywords/*  # All keywords
//*/title                 # Title at any level
```

### Text Functions

```xpath
/topic/title/text()       # Text content
/topic/prolog/author/text()  # Author text
/topic/prolog/critdates/created/@date  # Attribute
```

---

## Common Metadata Fields

### Title
```yaml
title:
  - "/topic/title"
  - "//title"
  - "/document/heading"
```

### Authors
```yaml
authors:
  - "/topic/prolog/author"
  - "//author"
  - "/document/metadata/creator"
```

### Created Date
```yaml
createdDate:
  - "/topic/prolog/critdates/created/@date"
  - "//created/@date"
  - "/document/metadata/created"
```

### Keywords
```yaml
keywords:
  - "/topic/prolog/metadata/keywords/*"
  - "//keywords/keyword"
  - "/document/metadata/tags/*"
```

### Body/Content
```yaml
body:
  - "/topic/body"
  - "//body"
  - "/document/content"
```

---

## Validation Rules

### 1. Domain Name Case
- Use lowercase for domain names: `space`, `time`, `soul`, `reality`, `mind`, `power`

### 2. Variant Name Format
- Use descriptive names: `FES-Space`, `BAT-Space`, `batch_1`
- Separate words with hyphens or underscores

### 3. XPath Expressions
- Must be valid XPath syntax
- Use absolute paths (`/`) or relative paths (`//`)
- Include text() for text content
- Include @attribute for attributes

### 4. Multiple Expressions
- List as array with `-` prefix
- Order by priority (primary first)
- Include fallbacks for flexibility

---

## Testing Your config.yaml

### 1. Syntax Validation

```python
import yaml

with open('config/config.yaml', 'r') as f:
    config = yaml.safe_load(f)
    print(config)
```

### 2. XPath Testing

```python
from lxml import etree

# Load XML
tree = etree.parse('test_document.xml')
root = tree.getroot()

# Test XPath
title = root.xpath('/topic/title/text()')
print(f"Title: {title}")

authors = root.xpath('/topic/prolog/author/text()')
print(f"Authors: {authors}")
```

### 3. Full Integration Test

```python
from app.ingestion.chunker import ConfigurableXMLChunker

chunker = ConfigurableXMLChunker(domain="SPACE")
result = chunker.process_and_chunk(xml_content)

print(f"Title: {result.get('title')}")
print(f"Authors: {result.get('authors')}")
print(f"Keywords: {result.get('keywords')}")
```

---

## Troubleshooting

### Issue: XPath returns empty results

**Solution**: 
1. Check XML namespace declarations
2. Use `//` for namespace-agnostic search
3. Verify element names match exactly

```yaml
# Before (may fail with namespaces)
title:
  - "/topic/title"

# After (namespace-agnostic)
title:
  - "//title"
  - "/topic/title"
```

### Issue: Multiple values not extracted

**Solution**: Use wildcard `*` to select all matching elements

```yaml
# Before (single element)
keywords:
  - "/topic/prolog/metadata/keywords/keyword"

# After (all keywords)
keywords:
  - "/topic/prolog/metadata/keywords/*"
```

### Issue: Attributes not extracted

**Solution**: Use `@attribute` syntax

```yaml
# Before (wrong)
date:
  - "/topic/prolog/critdates/created"

# After (correct)
date:
  - "/topic/prolog/critdates/created/@date"
```

---

## Best Practices

### 1. Order by Priority
```yaml
title:
  - "/topic/title"           # Most specific
  - "//mainbooktitle/text()" # More general
  - "//title"                # Fallback
```

### 2. Include Fallbacks
```yaml
authors:
  - "/topic/prolog/author"
  - "//author"
  - "/document/creator"
```

### 3. Use Text Functions
```yaml
# For text content
title:
  - "/topic/title/text()"

# For attributes
date:
  - "/topic/prolog/critdates/created/@date"
```

### 4. Document Your Mappings
```yaml
# SPACE Domain - Furnace Energy Space
space:
  FES-Space:
    # Title from topic element
    title:
      - "/topic/title"
    # Authors from prolog
    authors:
      - "/topic/prolog/author"
```

---

## Complete Template

```yaml
# Domain Name (lowercase)
{domain}:
  # Variant Name
  {variant}:
    # Document title
    title:
      - "/primary/xpath"
      - "//fallback/xpath"
    
    # Document authors
    authors:
      - "/primary/xpath"
      - "//fallback/xpath"
    
    # Creation date
    createdDate:
      - "/primary/xpath/@attribute"
      - "//fallback/xpath"
    
    # Keywords/tags
    keywords:
      - "/primary/xpath/*"
      - "//fallback/xpath"
    
    # Product information
    prodinfo:
      - "/primary/xpath"
      - "//fallback/xpath"
    
    # Other metadata
    othermeta:
      - "/primary/xpath"
      - "//fallback/xpath"
    
    # Document body/content
    body:
      - "/primary/xpath"
      - "//fallback/xpath"
```

---

## Adding New Domain

### Step 1: Create Domain Section
```yaml
newdomain:
  variant_name:
    # Add mappings
```

### Step 2: Add to Settings
```python
# app/config.py
DOMAINS: List[str] = [
    "MIND", "SOUL", "TIME", "SPACE", "REALITY", "POWER", "NEWDOMAIN"
]
```

### Step 3: Create Domain Config
```yaml
# config/domains/newdomain.yaml
domain:
  name: "newdomain"
  display_name: "New Domain"
  description: "Description"
```

### Step 4: Test
```bash
pytest tests/unit/test_ingestion/ -v
```

---

## Validation Checklist

- [ ] Domain name is lowercase
- [ ] Variant name is descriptive
- [ ] All XPath expressions are valid
- [ ] Multiple expressions ordered by priority
- [ ] Text content uses `/text()`
- [ ] Attributes use `/@attribute`
- [ ] Wildcards use `/*`
- [ ] YAML syntax is valid
- [ ] Tested with sample XML
- [ ] Fallback expressions included

---

## References

- [XPath Tutorial](https://www.w3schools.com/xml/xpath_intro.asp)
- [YAML Syntax](https://yaml.org/spec/1.2/spec.html)
- [lxml XPath](https://lxml.de/xpathxslt.html)

