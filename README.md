# Vector Graphics Control Protocol (VGCP) with ODViz

**Version:** 4.0  
**Status:** Draft Specification  
**Encoding:** UTF-8  
**Transport:** TCP (recommended), WebSocket, or SSE

---

## 1. Introduction

### 1.1 Purpose

The Vector Graphics Control Protocol (VGCP) with ODViz integration enables remote manipulation of a vector graphics scene graph with native support for tabular data and data-driven visualizations.

### 1.2 Design Philosophy

- **Stateless commands** — Each command is self-contained.
- **Human-readable** — Text-based with named fields for easy debugging.
- **Low overhead** — Minimal parsing required, no XML/JSON overhead.
- **Incremental updates** — Modify only what changed.
- **Unified data and visualization** — ODViz integration.
- **Centralized styles** — Only via style sheets.

---

## 2. Extended Data Model

### 2.1 Extended Document Structure

```
Document
├── Viewport (width, height)
├── Style Rule Store (selector → properties)    # ONLY source of styles
├── Symbol Store (symbol-id → element tree)
├── Element Tree (elements with hierarchy)
└── ODViz Data Store
    ├── Tables (table-name → table data)
    │   ├── Column definitions
    │   ├── Row data with hrefs
    │   ├── Statistics
    │   └── Metadata
    └── Relations (between tables)
```

### 2.2 ODViz References in Elements

Each graphical element can have a reference to a data row:

| Attribute | Usage | Description |
|-----------|-------|-------------|
| `href` | Graphical element | ID of the data row this element represents |
| `hrefs` | Data row | List of element IDs that represent this row |

### 2.3 Centralized Styles

**Fundamental principle:** Graphical styles can ONLY be defined through style rules (`STY`). Element creation commands DO NOT accept direct style parameters.

---

## 3. Element Commands (Modified)

### 3.1 Common Parameters (WITHOUT styles)

**Common Parameters:**
- `id` — Unique element identifier **(required)**
- `grp` — Parent group ID (default: `"root"`)
- `pos` — Insertion position: `f` (first) or `l` (last), default `l`
- `class` — Space-separated class names
- `tr` — Transform (see Section 4.7)
- `href` — ODViz reference to data row (optional)

### 3.2 `RCT` — Rectangle

**Syntax:**
```
RCT id=<id> [x=<x>] [y=<y>] [w=<width>] [h=<height>] [rx=<rx>] [ry=<ry>] [tr=<transform>] [grp=<parent>] [pos=f|l] [class=<names>] [href=<row-id>]
```

**Example:**
```
RCT id=bg x=0 y=0 w=1280 h=720 class=background
RCT id=bar-0 x=50 y=120 w=80 h=150 class="bar active" href=sale-0
```

### 3.3 `CIR` — Circle

**Syntax:**
```
CIR id=<id> [cx=<x>] [cy=<y>] [r=<radius>] [tr=<transform>] [grp=<parent>] [pos=f|l] [class=<names>] [href=<row-id>]
```

**Example:**
```
CIR id=dot cx=100 cy=100 r=20 class=data-point href=point-0
```

### 3.4 `ELL` — Ellipse

**Syntax:**
```
ELL id=<id> [cx=<x>] [cy=<y>] [rx=<rx>] [ry=<ry>] [tr=<transform>] [grp=<parent>] [pos=f|l] [class=<names>] [href=<row-id>]
```

### 3.5 `LIN` — Line

**Syntax:**
```
LIN id=<id> [x1=<x1>] [y1=<y1>] [x2=<x2>] [y2=<y2>] [tr=<transform>] [grp=<parent>] [pos=f|l] [class=<names>] [href=<row-id>]
```

### 3.6 `PAT` — Path

**Syntax:**
```
PAT id=<id> d="<path-data>" [tr=<transform>] [grp=<parent>] [pos=f|l] [class=<names>] [href=<row-id>]
```

### 3.7 `PGN` — Polygon

**Syntax:**
```
PGN id=<id> pts="<x1,y1 x2,y2 ...>" [tr=<transform>] [grp=<parent>] [pos=f|l] [class=<names>] [href=<row-id>]
```

### 3.8 `PLN` — Polyline

**Syntax:**
```
PLN id=<id> pts="<x1,y1 x2,y2 ...>" [tr=<transform>] [grp=<parent>] [pos=f|l] [class=<names>] [href=<row-id>]
```

### 3.9 `TXT` — Text

**Syntax:**
```
TXT id=<id> [x=<x>] [y=<y>] txt="<content>" [tr=<transform>] [grp=<parent>] [pos=f|l] [class=<names>] [href=<row-id>]
```

**Text parameters:**
- `txt` — Text content (required)

**Note:** Text styles (`fs`, `ff`, `fw`, `ta`) are now defined through style rules.

**Example:**
```
TXT id=label x=100 y=200 txt="Product A" class="data-label" href=sale-0
```

### 3.10 `IMG` — Image

**Syntax:**
```
IMG id=<id> [x=<x>] [y=<y>] [w=<width>] [h=<height>] href-img="<base64-data-uri>" [pr=<preserveAspectRatio>] [tr=<transform>] [grp=<parent>] [pos=f|l] [class=<names>] [href=<row-id>]
```

**Note:** The image `href` parameter is now `href-img` to avoid conflict with ODViz `href`.

### 3.11 `GRP` — Group

**Syntax:**
```
GRP id=<id> [tr=<transform>] [grp=<parent>] [pos=f|l] [class=<names>] [href=<row-id>]
```

**Note:** Groups ONLY inherit styles from style rules, they have no direct styles.

---

## 4. ODViz Commands

### 4.1 `TABLE` — Create or Update Table

**Description:** Defines an ODViz data table with its columns and metadata.

**Syntax:**
```
TABLE name=<name> columns="<col1,col2,...>" [description=<desc>] [visualization=<type>] [version=<ver>] [source=<src>] [author=<author>] [license=<license>] [currency=<currency>] [unit=<unit>] [primaryKey=<key>]
```

**Parameters:**
- `name` — Unique table name (required)
- `columns` — Comma-separated column names (required)
- `description` — Human-readable description
- `visualization` — Suggested visualization type (`chart`, `map`, `graph`, `tree`)
- `version` — Version number
- `source` — Data source
- `author` — Creator name
- `license` — Usage license
- `currency` — ISO currency code
- `unit` — Unit of measurement
- `primaryKey` — Primary key column

**Response:** `OK` or `ERROR`

**Example:**
```
TABLE name=sales columns="id,product,value,color" description="Sales by product 2024" visualization=chart author="Ana Lopez" currency=USD primaryKey=id
```

### 4.2 `COLUMN` — Define Column

**Description:** Defines detailed metadata for a column.

**Syntax:**
```
COLUMN table=<name> name=<col> type=<type> [description=<desc>] [unit=<unit>] [enumeration=<values>] [min=<min>] [max=<max>] [required=<t|f>] [formula=<expr>] [readonly=<t|f>]
```

**Supported types:** `string`, `integer`, `decimal`, `date`, `datetime`, `boolean`, `percentage`, `currency`

**Example:**
```
COLUMN table=sales name=product type=string description="Product name" required=t
COLUMN table=sales name=value type=currency unit=USD min=0
COLUMN table=sales name=percentage type=percentage formula="(value/target)*100" readonly=t
```

### 4.3 `ROW` — Add or Update Row

**Description:** Adds or updates a data row in a table.

**Syntax:**
```
ROW table=<name> id=<row-id> [hrefs=<shape-ids>] data="<csv-data>"
```

**Parameters:**
- `table` — Table name (required)
- `id` — Unique row ID (required)
- `hrefs` — Comma-separated list of graphical element IDs (optional)
- `data` — CSV data for the row (required)

**Response:** `OK` or `ERROR NOT_FOUND` (if table doesn't exist)

**Example:**
```
ROW table=sales id=sale-0 hrefs="bar-0,label-0,value-0" data="0,Product A,150,#2196F3"
ROW table=sales id=sale-1 hrefs="bar-1,label-1" data="1,Product B,85,#4CAF50"
```

### 4.4 `ROWS` — Load Multiple Rows

**Description:** Loads multiple rows at once.

**Syntax:**
```
ROWS table=<name> data="<csv-data-with-ids>"
```

**Data format:** CSV where the first column is the row ID and the second (optional) is `hrefs`.

**Example:**
```
ROWS table=sales data="sale-0,bar-0 label-0,0,Product A,150,#2196F3
sale-1,bar-1 label-1,1,Product B,85,#4CAF50
sale-2,bar-2 label-2,2,Product C,234,#FF9800"
```

### 4.5 `ROW-DEL` — Delete Row

**Description:** Deletes a row from a table.

**Syntax:**
```
ROW-DEL table=<name> id=<row-id>
ROW-DEL table=<name> all=t
```

**Example:**
```
ROW-DEL table=sales id=sale-0
ROW-DEL table=sales all=t
```

### 4.6 `ROW-GET` — Get Row

**Description:** Retrieves data from a row.

**Syntax:**
```
ROW-GET table=<name> id=<row-id>
ROW-GET table=<name> all=t
```

**Response:** `OK` with row data or rows.

**Example:**
```
ROW-GET table=sales id=sale-0
SERVER> OK id=sale-0 hrefs=bar-0,label-0 data="0,Product A,150,#2196F3"
```

### 4.7 `RELATION` — Define Table Relation

**Description:** Defines a foreign key relationship between tables.

**Syntax:**
```
RELATION table=<name> target=<table> localKey=<column> foreignKey=<column> [type=<join-type>]
```

**Join types:** `inner`, `left`, `right`, `full` (default: `inner`)

**Example:**
```
RELATION table=sales target=products localKey=product_id foreignKey=id type=inner
RELATION table=sales target=regions localKey=region_code foreignKey=code type=left
```

### 4.8 `STATS` — Calculate Statistics

**Description:** Calculates and stores statistics for a column.

**Syntax:**
```
STATS table=<name> column=<col> [groupBy=<col>]
```

**Example:**
```
STATS table=sales column=value
STATS table=sales column=value groupBy=product
```

**Response:** `OK` with calculated statistics.

### 4.9 `STATS-GET` — Get Statistics

**Description:** Retrieves previously calculated statistics.

**Syntax:**
```
STATS-GET table=<name> column=<col> [groupBy=<col>]
```

**Example:**
```
STATS-GET table=sales column=value
SERVER> OK total=469 average=156.33 min=85 max=234
```

### 4.10 `TABLE-DEL` — Delete Table

**Description:** Deletes an entire table.

**Syntax:**
```
TABLE-DEL name=<name>
TABLE-DEL all=t
```

**Example:**
```
TABLE-DEL name=sales
TABLE-DEL all=t
```

### 4.11 `TABLE-GET` — Get Table Metadata

**Description:** Retrieves table metadata.

**Syntax:**
```
TABLE-GET name=<name>
TABLE-GET all=t
```

**Example:**
```
TABLE-GET name=sales
SERVER> OK name=sales columns="id,product,value,color" visualization=chart rows=3
```

---

## 5. Style Commands (Unchanged)

### 5.1 `STY` — Create or Update Style Rule

**Syntax:**
```
STY sel=<selector> [property=value...]
```

**Supported properties:** `fill`, `stroke`, `sw` (stroke-width), `op` (opacity), `fs` (font-size), `ff` (font-family), `fw` (font-weight), `ta` (text-anchor)

**Examples:**
```
STY sel=.bar fill=#2196F3
STY sel=.bar.active fill=#FF9800 stroke=#FF6F00 sw=2
STY sel=.data-label fill=white fs=14 ff=Arial ta=m
STY sel=RCT fill=blue
STY sel=#logo op=0.8
```

### 5.2 `STY-DEL` — Delete Style Rules

**Syntax:**
```
STY-DEL sel=<selector>
STY-DEL all=t
```

### 5.3 `STY-GET` — Query Style Rules

**Syntax:**
```
STY-GET sel=<selector>
STY-GET all=t
```

---

## 6. Attribute Commands (Modified)

### 6.1 `ATTR` — Update Element Attributes

**Description:** Modifies attributes of an existing element. **DOES NOT allow direct style modifications.**

**Syntax:**
```
ATTR id=<id> [param=value...]
```

**Allowed parameters:** All except `fill`, `stroke`, `sw`, `op`, `fs`, `ff`, `fw`, `ta`

**Example:**
```
ATTR id=logo href-img="data:image/png;base64,newBase64DataHere..."
ATTR id=star pts="110,20 50,190 200,70 20,70 170,190"
ATTR id=bar-0 href=sale-0
```

### 6.2 `STY` — Update Element Styles (via class)

**Description:** Modifies an element's class to change its style.

**Syntax:**
```
STY id=<id> class=<new-class>
STY id=<id> class=<add-class> action=add
STY id=<id> class=<remove-class> action=remove
```

**Example:**
```
STY id=bar-0 class="bar active"
STY id=bar-1 class="bar" action=add
STY id=bar-2 class="active" action=remove
```

---

## 7. Symbol Commands (Modified)

### 7.1 `SYM` — Define Symbol

**Syntax:**
```
SYM id=<id> vb="<min-x min-y width height>" [pr=<preserveAspectRatio>]
```

**Note:** Elements within a symbol also use styles through style rules, not direct styles.

**Example:**
```
SYM id=button vb="0 0 100 50"
RCT id=btnBg x=0 y=0 w=100 h=50 rx=5 class=button-bg
TXT id=btnText x=50 y=25 txt="Button" class=button-text ta=m
```

### 7.2 `USE` — Instantiate Symbol

**Syntax:**
```
USE id=<id> sym=<symbol-id> [x=<x>] [y=<y>] [w=<width>] [h=<height>] [tr=<transform>] [grp=<parent>] [pos=f|l] [class=<names>] [href=<row-id>]
```

**Note:** The instance can have its own class and ODViz reference.

**Example:**
```
USE id=btn1 sym=button x=10 y=70 class=primary
USE id=btn2 sym=button x=10 y=140 w=150 h=60 class=success href=action-1
```

---

## 8. Complete Session Example

```
CLIENT> VIEW w=800 h=600
SERVER> OK

# Define styles
CLIENT> STY sel=.bar fill=#2196F3
SERVER> OK
CLIENT> STY sel=.bar.highlight fill=#FF9800 stroke=#E65100 sw=2
SERVER> OK
CLIENT> STY sel=.label fill=white fs=14 ff=Arial ta=m
SERVER> OK
CLIENT> STY sel=.value fill=white fs=12 ff=Arial ta=m
SERVER> OK
CLIENT> STY sel=.title fill=#333 fs=24 ff=Arial ta=m fw=b
SERVER> OK

# Define ODViz table
CLIENT> TABLE name=sales columns="id,product,value,color" visualization=chart
SERVER> OK

# Define columns
CLIENT> COLUMN table=sales name=product type=string description="Product name"
SERVER> OK
CLIENT> COLUMN table=sales name=value type=currency unit=USD min=0
SERVER> OK

# Add rows with references to graphical elements
CLIENT> ROW table=sales id=sale-0 hrefs="bar-0,label-0,value-0" data="0,Product A,150,#2196F3"
SERVER> OK
CLIENT> ROW table=sales id=sale-1 hrefs="bar-1,label-1,value-1" data="1,Product B,85,#4CAF50"
SERVER> OK
CLIENT> ROW table=sales id=sale-2 hrefs="bar-2,label-2,value-2" data="2,Product C,234,#FF9800"
SERVER> OK

# Calculate statistics
CLIENT> STATS table=sales column=value
SERVER> OK total=469 average=156.33 min=85 max=234

# Create graphical elements (without direct styles)
CLIENT> GRP id=chart tr="t(50,50)"
SERVER> OK

CLIENT> RCT id=bar-0 x=50 y=120 w=80 h=150 class="bar" href=sale-0
SERVER> OK
CLIENT> RCT id=bar-1 x=160 y=170 w=80 h=85 class="bar" href=sale-1
SERVER> OK
CLIENT> RCT id=bar-2 x=270 y=50 w=80 h=234 class="bar highlight" href=sale-2
SERVER> OK

CLIENT> TXT id=label-0 x=90 y=290 txt="Product A" class="label" href=sale-0
SERVER> OK
CLIENT> TXT id=label-1 x=200 y=290 txt="Product B" class="label" href=sale-1
SERVER> OK
CLIENT> TXT id=label-2 x=310 y=290 txt="Product C" class="label" href=sale-2
SERVER> OK

CLIENT> TXT id=value-0 x=90 y=115 txt="150" class="value" href=sale-0
SERVER> OK
CLIENT> TXT id=value-1 x=200 y=165 txt="85" class="value" href=sale-1
SERVER> OK
CLIENT> TXT id=value-2 x=310 y=45 txt="234" class="value" href=sale-2
SERVER> OK

CLIENT> TXT id=title x=400 y=30 txt="Sales by Product" class="title"
SERVER> OK

# Query rows
CLIENT> ROW-GET table=sales all=t
SERVER> OK id=sale-0 hrefs=bar-0,label-0,value-0 data="0,Product A,150,#2196F3"
SERVER> OK id=sale-1 hrefs=bar-1,label-1,value-1 data="1,Product B,85,#4CAF50"
SERVER> OK id=sale-2 hrefs=bar-2,label-2,value-2 data="2,Product C,234,#FF9800"

# Query statistics
CLIENT> STATS-GET table=sales column=value
SERVER> OK total=469 average=156.33 min=85 max=234

# Dynamically change styles
CLIENT> STY-DEL sel=.bar.highlight
SERVER> OK
CLIENT> STY sel=.bar.highlight fill=#FF5722 stroke=#BF360C sw=2
SERVER> OK

# Verify style application
CLIENT> GET id=bar-2
SERVER> OK id=bar-2 type=RCT x=270 y=50 w=80 h=234 class="bar highlight" href=sale-2
```

---

## 9. Benefits of Integration

### 9.1 Key Advantages

1. **Clear separation of concerns**:
   - Data → ODViz (tables, rows, columns)
   - Styles → Style rules (STY)
   - Geometry → Graphical elements

2. **Bidirectional traceability**:
   - Each element knows what data it represents (`href`)
   - Each row knows which elements represent it (`hrefs`)

3. **Efficient updates**:
   - Changing a style updates all elements with that class
   - Modifying ODViz data doesn't require changes to graphical elements

4. **Interoperability**:
   - Generated SVG is ODViz-compatible
   - Data can be exported/imported as CSV

### 9.2 Comparison

| Feature | Original VGCP | VGCP with ODViz |
|---------|---------------|-----------------|
| Direct styles | Yes | No (only via STY) |
| Embedded data | No | Yes (ODViz) |
| Data ↔ visual references | No | Yes (href/hrefs) |
| Data metadata | No | Yes (columns, statistics) |
| Data relationships | No | Yes (relations) |
| Traceability | Low | High |

---

## 10. Appendix: Compatibility

### 10.1 Migration from VGCP 3.x

Legacy commands with direct styles must be replaced by:

1. Define style rules with `STY`
2. Assign classes to elements
3. Remove style parameters from creation commands

### 10.2 Generating ODViz SVG

The server must be able to generate an ODViz SVG document from internal state:

```xml
<svg xmlns="http://www.w3.org/2000/svg" xmlns:v="https://odviz.org/1.0/">
  <metadata>
    <v:table name="sales" columns="id,product,value,color">
      <v:row id="sale-0" hrefs="bar-0,label-0,value-0">
        0,Product A,150,#2196F3
      </v:row>
      <!-- ... -->
    </v:table>
  </metadata>
  
  <style>
    .bar { fill: #2196F3; }
    .bar.highlight { fill: #FF9800; stroke: #E65100; stroke-width: 2; }
    /* ... */
  </style>
  
  <g id="chart">
    <rect id="bar-0" href="#sale-0" x="50" y="120" width="80" height="150" class="bar"/>
    <!-- ... -->
  </g>
</svg>
```

---

## 11. Summary of Changes

| Command | Change |
|---------|--------|
| `RCT`, `CIR`, `ELL`, `LIN`, `PAT`, `PGN`, `PLN`, `TXT`, `IMG`, `GRP` | Removed style parameters (`fill`, `stroke`, `sw`, `op`, `fs`, `ff`, `fw`, `ta`) |
| `TXT` | Removed `fs`, `ff`, `fw`, `ta` |
| `IMG` | `href` renamed to `href-img` to avoid conflict |
| `ATTR` | Does not allow direct style modifications |
| `STY` | Unchanged (now the ONLY source of styles) |
| `SYM` | Internal elements use only classes |
| `USE` | Removed style parameters |
| **New:** `TABLE`, `COLUMN`, `ROW`, `ROWS`, `ROW-DEL`, `ROW-GET`, `RELATION`, `STATS`, `STATS-GET`, `TABLE-DEL`, `TABLE-GET` | ODViz commands |
| **New:** `ATTR-STY` | Modify classes to change styles |