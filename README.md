# Vector Graphics Control Protocol (VGCP)

## Specification Version 3.3

**Status:** Draft Specification  
**Category:** Application Protocol  
**Encoding:** UTF-8  
**Transport:** TCP (recommended), WebSocket, or SSE

---

## 1. Introduction

### 1.1 Purpose

The Vector Graphics Control Protocol (VGCP) enables remote manipulation of a vector graphics scene graph through a simple, text-based command interface using a `key=value` format. Clients can create, modify, query, and delete graphic elements, manage style rules, and manage symbol definitions for reuse.

### 1.2 Design Philosophy

- **Stateless commands** — Each command is self-contained.
- **Human-readable** — Text-based with named fields for easy debugging.
- **Low overhead** — Minimal parsing required, no XML/JSON overhead.
- **Incremental updates** — Modify only what changed.
- **Parser-friendly** — Simple split operations, no complex grammars.
- **Reusability** — Symbols can be defined once and instantiated many times.

### 1.3 Relationship to Existing Standards

This protocol is distinct from:

- **SVG Streaming (W3C Working Group Note, 2003)** — A media transport for pre-rendered SVG animations synchronized with timeline.
- **SVG DOM (W3C Recommendation)** — Client-side scripting API.
- **HTTP/2 Server Push** — Web transport mechanism.

VGCP focuses on **interactive remote control** of vector graphics, not linear animation delivery.

---

## 2. Transport Protocol

### 2.1 Connection Establishment

The server MUST listen for TCP connections on a configurable port. Upon connection acceptance, the server MUST:

1. Initialize an empty vector graphics document model.
2. Set default viewport: `width="800" height="600"`.
3. Initialize empty style rule store.
4. Initialize empty symbol store.
5. Begin processing commands.

### 2.2 Message Framing

Each command message is a single line terminated by LF (`\n`) or CRLF (`\r\n`):

```
COMMAND param1=value1 param2=value2 param3=value3
```

**Rules:**

- Commands SHALL be terminated by newline character (`\n` or `\r\n`).
- Parameters SHALL be in `key=value` format.
- Spaces separate parameters.
- Values MAY contain alphanumeric characters, hyphens, underscores, hash symbols (for colors), and Base64 characters (`A-Za-z0-9+/=`).
- No multi-line commands are required; complex values use multiple parameters.

### 2.3 Response Format

All responses follow the same single-line format:

```
OK
```

or

```
ERROR category: description
```

**Rules:**
- Responses SHALL be terminated by newline character.
- `OK` indicates successful command execution.
- `ERROR` includes a category and human-readable description.

### 2.4 Character Encoding

All messages MUST use UTF-8 encoding without BOM.

### 2.5 Connection Management

| Action            | Behavior                                          |
| ----------------- | ------------------------------------------------- |
| Idle timeout      | Server MAY close after 300 seconds of inactivity |
| Graceful shutdown | Server sends `BYE` then closes                    |
| Abrupt close      | Either party may close the connection at any time |

---

## 3. Data Model

### 3.1 Document Structure

The server maintains an in-memory representation consisting of:

```
Document
├── Viewport (width, height)
├── Style Rule Store (selector → properties)
├── Symbol Store (symbol-id → element tree)
└── Element Tree (elements with hierarchy, including symbol instances)
```

### 3.2 Element Identity

Every element MUST have a unique `id` attribute within the document scope. The server SHALL reject elements with duplicate IDs. Symbol instances are regular elements with their own IDs.

### 3.3 Grouping and Hierarchy

Special parameters control element placement:

| Parameter | Type              | Description                                          |
| --------- | ----------------- | ---------------------------------------------------- |
| `grp`     | string            | ID of parent group (default: `"root"`)               |
| `pos`     | `f` or `l`        | Insertion position: `f` (first) or `l` (last), default `l` |

The root group is implicitly defined and cannot be deleted.

**Example with positioning:**
```
CIR id=led grp=dashboard pos=f cx=50 cy=50 r=20
```

### 3.4 Class Attribute

The `class` attribute MAY contain **space-separated class names**. Style rules apply if an element matches **any** of its classes.

**Example:**
```
RCT id=status class="led active warning"
STY sel=.led fill=blue      # Applies (matches "led")
STY sel=.active stroke=red  # Applies (matches "active")
```

### 3.5 Style Rule Resolution

Style rules SHALL be applied according to standard CSS specificity rules with the following precedence (highest to lowest):

1. **Inline styles** (attributes set directly via element commands or `ATTR`)
2. **ID selectors** (`#id`)
3. **Class selectors** (`.class`)
4. **Type selectors** (`RCT`, `CIR`, etc.)
5. **Inherited styles** (values propagated upward through the `grp` hierarchy)

### 3.6 Style Inheritance

Visual properties applied to a group (`GRP`) — either inline or via a style rule (`STY`) — SHALL automatically cascade down to all descendant elements within that group hierarchy, unless explicitly overridden by a rule of higher precedence on the descendant itself.

#### 3.6.1 Inheritable Properties

The server MUST propagate the following attributes through the `grp` chain:
- `fill`
- `stroke`
- `sw` (stroke-width)
- Font properties for `TXT` elements: `fs`, `ff`, `fw`, `ta`.

*Note on Opacity (`op`):* Opacity is cumulative. The effective rendering opacity of a child element MUST be the mathematical product of its own opacity and the opacities of all its parent groups ($op_{final} = op_{child} \times op_{parent}$).

#### 3.6.2 Resolution Walk

When rendering an element, if a property (e.g., `fill`) is not resolved by steps 1-4 of Section 3.5, the server MUST traverse up the tree using the `grp` attribute. It will use the first defined value it finds. If it reaches `"root"` without finding a definition, it SHALL apply the viewport's default style.

### 3.7 Symbols and Instances

Symbols are reusable templates defined with the `SYM` command. Once defined, symbols can be instantiated multiple times using the `USE` command.

- Symbol definitions are stored separately from the main element tree.
- Symbol instances behave like groups: they can have their own transforms and attributes.
- Changes to a symbol definition do NOT affect existing instances (symbols are copied at instantiation time).
- To update all instances, redefine the symbol and re-instantiate.

---

## 4. Command Reference

### 4.1 Document Structure Commands

#### 4.1.1 `VIEW` — Set Viewport Dimensions
**Description:** Defines the width and height of the drawing area.

**Syntax:**
```
VIEW w=<width> h=<height>
```

**Response:** `OK`

**Example:**
```
VIEW w=1280 h=720
```

#### 4.1.2 `CLR` — Clear All Elements (Preserve Styles, Symbols, and Viewport)
**Description:** Deletes all elements but keeps style rules, symbol definitions, and viewport.

**Syntax:**
```
CLR
```

**Response:** `OK`

#### 4.1.3 `RST` — Full Document Reset
**Description:** Clears all elements, deletes all style rules, deletes all symbol definitions, and resets viewport to 800x600.

**Syntax:**
```
RST
```

**Response:** `OK`

#### 4.1.4 `BYE` — Terminate Session
**Description:** Gracefully closes the connection.

**Syntax:**
```
BYE
```

**Response:** `BYE` (then connection closed)

---

### 4.2 Primitive Element Commands

Each command creates an element of its respective type. All share common parameters:

**Common Parameters:**
- `id` — Unique element identifier **(required)**
- `grp` — Parent group ID (default: `"root"`)
- `pos` — Insertion position: `f` (first) or `l` (last), default `l`
- `class` — Space-separated class names

**Visual Parameters (shared):**
- `fill` — Fill color (hex `#RRGGBB` or `#RRGGBBAA`, or color name)
- `stroke` — Stroke color (hex or color name)
- `sw` — Stroke width (default: `0`)
- `op` — Opacity (0-1, default: `1`)
- `tr` — Transform (matrix or transform functions, see Section 4.7)

#### 4.2.1 `RCT` — Rectangle
**Description:** Creates a rectangle element.

**Syntax:**
```
RCT id=<id> [x=<x>] [y=<y>] [w=<width>] [h=<height>] [rx=<rx>] [ry=<ry>] [fill=<color>] [stroke=<color>] [sw=<width>] [op=<opacity>] [tr=<transform>] [grp=<parent>] [pos=f|l] [class=<names>]
```

**Example:**
```
RCT id=bg x=0 y=0 w=1280 h=720 fill=#1a1a2e
RCT id=box x=100 y=100 w=200 h=150 rx=10 fill=blue stroke=white sw=2
```

#### 4.2.2 `CIR` — Circle
**Description:** Creates a circle element.

**Syntax:**
```
CIR id=<id> [cx=<x>] [cy=<y>] [r=<radius>] [fill=<color>] [stroke=<color>] [sw=<width>] [op=<opacity>] [tr=<transform>] [grp=<parent>] [pos=f|l] [class=<names>]
```

**Example:**
```
CIR id=dot cx=50 cy=50 r=20 fill=red stroke=black sw=1
```

#### 4.2.3 `ELL` — Ellipse
**Description:** Creates an ellipse element.

**Syntax:**
```
ELL id=<id> [cx=<x>] [cy=<y>] [rx=<rx>] [ry=<ry>] [fill=<color>] [stroke=<color>] [sw=<width>] [op=<opacity>] [tr=<transform>] [grp=<parent>] [pos=f|l] [class=<names>]
```

**Example:**
```
ELL id=oval cx=100 cy=100 rx=60 ry=30 fill=yellow stroke=orange sw=2
```

#### 4.2.4 `LIN` — Line
**Description:** Creates a line element.

**Syntax:**
```
LIN id=<id> [x1=<x1>] [y1=<y1>] [x2=<x2>] [y2=<y2>] [stroke=<color>] [sw=<width>] [op=<opacity>] [tr=<transform>] [grp=<parent>] [pos=f|l] [class=<names>]
```

**Example:**
```
LIN id=edge x1=0 y1=0 x2=100 y2=100 stroke=white sw=3
```

#### 4.2.5 `PAT` — Path
**Description:** Creates a path element. The `d` attribute contains path data using standard SVG path commands (M, L, C, Q, A, Z, etc.).

**Syntax:**
```
PAT id=<id> d="<path-data>" [fill=<color>] [stroke=<color>] [sw=<width>] [op=<opacity>] [tr=<transform>] [grp=<parent>] [pos=f|l] [class=<names>]
```

**Example:**
```
PAT id=arrow d="M10 10 L100 100 L90 110 Z" fill=none stroke=blue sw=2
```

#### 4.2.6 `PGN` — Polygon
**Description:** Creates a polygon element (closed shape with straight lines).

**Syntax:**
```
PGN id=<id> pts="<x1,y1 x2,y2 ...>" [fill=<color>] [stroke=<color>] [sw=<width>] [op=<opacity>] [tr=<transform>] [grp=<parent>] [pos=f|l] [class=<names>]
```

**Points format:** Space-separated `x,y` pairs. Commas between coordinates are optional.

**Example:**
```
PGN id=star pts="100,10 40,180 190,60 10,60 160,180" fill=yellow stroke=black sw=1
PGN id=triangle pts="0,0 50,100 100,0" fill=red
```

#### 4.2.7 `PLN` — Polyline
**Description:** Creates a polyline element (open shape with straight lines).

**Syntax:**
```
PLN id=<id> pts="<x1,y1 x2,y2 ...>" [fill=<color>] [stroke=<color>] [sw=<width>] [op=<opacity>] [tr=<transform>] [grp=<parent>] [pos=f|l] [class=<names>]
```

**Points format:** Space-separated `x,y` pairs. Commas between coordinates are optional.

**Example:**
```
PLN id=wave pts="0,100 50,150 100,80 150,120" fill=none stroke=cyan sw=2
```

#### 4.2.8 `TXT` — Text
**Description:** Creates a text element.

**Syntax:**
```
TXT id=<id> [x=<x>] [y=<y>] txt="<content>" [fill=<color>] [fs=<font-size>] [ff=<font-family>] [fw=<bold|normal>] [ta=<s|m|e>] [op=<opacity>] [tr=<transform>] [grp=<parent>] [pos=f|l] [class=<names>]
```

**Text parameters:**
- `txt` — Text content (required)
- `fs` — Font size (default: `14`)
- `ff` — Font family (default: `Helvetica`)
- `fw` — Font weight: `b` (bold) or `n` (normal), default `n`
- `ta` — Text anchor: `s` (start), `m` (middle), `e` (end), default `s`

**Example:**
```
TXT id=label x=100 y=200 txt="Hello World" fill=white fs=24 ff=Arial ta=m
```

#### 4.2.9 `IMG` — Image
**Description:** Creates an image element. The image data MUST be embedded as a Base64-encoded data URI.

**Syntax:**
```
IMG id=<id> [x=<x>] [y=<y>] [w=<width>] [h=<height>] href="<base64-data-uri>" [pr=<preserveAspectRatio>] [op=<opacity>] [tr=<transform>] [grp=<parent>] [pos=f|l] [class=<names>]
```

**href format:**
```
href=data:image/<format>;base64,<base64-data>
```
Where `<format>` is `png`, `jpeg`, `jpg`, `gif`, `webp`, or `svg+xml`.

**Image parameters:**
- `pr` — Preserve aspect ratio: `none` | `meet` | `slice`, default `meet`

**Example:**
```
IMG id=logo x=50 y=50 w=200 h=200 href="data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAUA..."
```

#### 4.2.10 `GRP` — Group

**Description:** Creates a group container for organizing elements and establishing a local style context for inheritance.

**Syntax:**
```
GRP id= [tr=] [grp=] [pos=f|l] [class=] [fill=] [stroke=] [sw=] [op=]
```

**Example:**

Creates a group where all children will be red with a 2px black border by default

```
GRP id=layer1 grp=root fill=#ff0000 stroke=#000000 sw=2
RCT id=rectInGroup grp=layer1 x=10 y=10 w=50 h=50
```

---

### 4.3 Symbol Commands

#### 4.3.1 `SYM` — Define Symbol
**Description:** Creates a reusable symbol template. Symbol definitions are stored separately and can be instantiated using `USE`.

**Syntax:**
```
SYM id=<id> vb="<min-x min-y width height>" [pr=<preserveAspectRatio>]
```

**Symbol parameters:**
- `id` — Unique symbol identifier (required)
- `vb` — ViewBox: `"min-x min-y width height"` (required)
- `pr` — Preserve aspect ratio: `none` | `meet` | `slice`, default `meet`

**Note:** After defining a symbol, use element commands (`RCT`, `CIR`, `PAT`, etc.) to build its content. All elements created after `SYM` are added to the symbol definition until another `SYM` command or until symbol definition ends implicitly.

**Alternative syntax (explicit block):**
```
SYM id=<id> vb="..." begin
... element commands ...
SYM end
```

**Example:**
```
SYM id=iconHeart vb="0 0 24 24" pr="xMidYMid meet"
PAT id=heartPath d="M12 21.35l-1.45-1.32C5.4 15.36 2 12.28 2 8.5 2 5.42 4.42 3 7.5 3c1.74 0 3.41.81 4.5 2.09C13.09 3.81 14.76 3 16.5 3 19.58 3 22 5.42 22 8.5c0 3.78-3.4 6.86-8.55 11.54L12 21.35z" fill=red
```

#### 4.3.2 `SYM-DEL` — Delete Symbol
**Description:** Removes a symbol definition.

**Syntax:**
```
SYM-DEL id=<id>
SYM-DEL all=t
```

**Response:** `OK` or `ERROR NOT_FOUND`

**Example:**
```
SYM-DEL id=iconHeart
SYM-DEL all=t
```

#### 4.3.3 `USE` — Instantiate Symbol
**Description:** Creates an instance of a previously defined symbol.

**Syntax:**
```
USE id=<id> sym=<symbol-id> [x=<x>] [y=<y>] [w=<width>] [h=<height>] [tr=<transform>] [grp=<parent>] [pos=f|l] [class=<names>]
```

**Parameters:**
- `id` — Unique instance identifier (required)
- `sym` — Symbol ID to instantiate (required)
- `x`, `y` — Position offset (default: `0`, `0`)
- `w`, `h` — Override width/height (scales the symbol)
- `tr` — Transform to apply to this instance
- `grp`, `pos`, `class` — Standard hierarchy parameters

**Note:** Attributes specified in `USE` (like `fill`, `stroke`) override those defined in the symbol.

**Response:** `OK` or `ERROR NOT_FOUND` (if symbol doesn't exist)

**Example:**
```
# Define symbol once
SYM id=button vb="0 0 100 50"
RCT id=btnBg x=0 y=0 w=100 h=50 rx=5 fill=blue
TXT id=btnText x=50 y=25 txt="Button" ta=m fill=white fs=14

# Reuse symbol multiple times
USE id=btn1 sym=button x=10 y=10
USE id=btn2 sym=button x=10 y=70
USE id=btn3 sym=button x=10 y=130 w=150 h=60 fill=green

# With transform
USE id=btn4 sym=button tr="matrix(1.5,0,0,1.5,200,100)"

# Override specific attributes
USE id=btn5 sym=button x=300 y=50 fill=orange
```

---

### 4.4 Attribute Commands

#### 4.4.1 `ATTR` — Update Element Attributes
**Description:** Modifies one or more attributes of an existing element or symbol instance.

**Syntax:**
```
ATTR id=<id> [param=value...]
```

**Response:** `OK` or `ERROR NOT_FOUND`

**Example:**
```
ATTR id=logo href="data:image/png;base64,newBase64DataHere..."
ATTR id=star fill=orange pts="110,20 50,190 200,70 20,70 170,190"
ATTR id=btn1 fill=red
```

#### 4.4.2 `DEL` — Delete Elements
**Description:** Removes elements (including symbol instances) by ID, list, or all.

**Syntax:**
```
DEL id=<id>
DEL id=<id1>,<id2>,<id3>
DEL all=t
```

**Response:** `OK`

**Example:**
```
DEL id=star
DEL id=bg,logo,title
DEL all=t
```

#### 4.4.3 `GET` — Query Elements
**Description:** Returns current attributes of specified elements.

**Syntax:**
```
GET id=<id>
GET id=<id1>,<id2>,<id3>
GET all=t
```

**Response:** One or more `OK` lines with element data.

**Note:** For `IMG` elements, the `href` value SHOULD be truncated or omitted in responses.

**Example:**
```
GET id=star
SERVER> OK id=star type=PGN pts="100,10 40,180 190,60 10,60 160,180" fill=yellow stroke=black sw=1
```

#### 4.4.4 `EX` — Check Element Existence
**Description:** Tests whether an element ID exists.

**Syntax:**
```
EX id=<id>
```

**Response:** `OK EX=t` or `OK EX=f`

**Example:**
```
EX id=star
SERVER> OK EX=t
```

---

### 4.5 Interaction Events

**Description:** Server-to-client notifications about user interactions.

**Syntax:**
```
EVENT type=<event_type> target=<id> x=<local_x> y=<local_y> gx=<global_x> gy=<global_y>
```

**Supported event types:** `click`, `dblclick`, `mousedown`, `mouseup`, `mousemove`, `mouseenter`, `mouseleave`

**Example:**
```
SERVER> EVENT type=click target=star x=45 y=32 gx=145 gy=132 button=left
```

---

### 4.6 Style Commands

#### 4.6.1 `STY` — Create or Update Style Rule
**Description:** Defines or replaces a CSS style rule.

**Syntax:**
```
STY sel=<selector> [property=value...]
```

**Supported properties:** `fill`, `stroke`, `sw` (stroke-width), `op` (opacity), `fs` (font-size), `ff` (font-family), `fw` (font-weight), `ta` (text-anchor)

**Example:**
```
STY sel=.hl fill=gold stroke=orange sw=3
STY sel=RCT fill=blue
STY sel=#logo op=0.8
```

#### 4.6.2 `STY-DEL` — Delete Style Rules
**Description:** Removes style rules by selector.

**Syntax:**
```
STY-DEL sel=<selector>
STY-DEL sel=<selector1>,<selector2>
STY-DEL all=t
```

**Example:**
```
STY-DEL sel=.hl
STY-DEL sel=.warning,.error
STY-DEL all=t
```

#### 4.6.3 `STY-GET` — Query Style Rules
**Description:** Retrieves current style rules.

**Syntax:**
```
STY-GET sel=<selector>
STY-GET all=t
```

**Response:** One or more `OK` lines with rule data.

**Example:**
```
STY-GET sel=.hl
SERVER> OK sel=.hl fill=gold stroke=orange sw=3
```

---

### 4.7 Transformations

#### 4.7.1 Overview

The `tr` (transform) attribute applies geometric transformations to elements, groups, and symbol instances. Transformations can be specified as:

1. **Matrix notation** — Complete 6-parameter affine transformation matrix
2. **Function notation** — Individual transform functions

#### 4.7.2 Transform Functions

| Function | Syntax | Description |
|----------|--------|-------------|
| `matrix` | `matrix(a,b,c,d,e,f)` | 2D affine transformation matrix |
| `translate` | `t(tx)` or `t(tx,ty)` | Translates by tx, ty (y defaults to 0) |
| `rotate` | `r(angle)` or `r(angle,cx,cy)` | Rotates by angle degrees around (cx,cy) |
| `scale` | `s(sx)` or `s(sx,sy)` | Scales by sx, sy (sy defaults to sx) |
| `skewX` | `skewX(angle)` | Skews along X axis |
| `skewY` | `skewY(angle)` | Skews along Y axis |

**Abbreviations:**
- `t()` for `translate()`
- `r()` for `rotate()`
- `s()` for `scale()`

#### 4.7.3 Transform Composition

Multiple transform functions can be combined by separating them with spaces. They are applied from **right to left** (last specified is applied first).

```
tr="t(100,100) r(45) s(2)"
```

This translates, then rotates, then scales.

#### 4.7.4 Examples

```bash
# Matrix transformation
GRP id=scene tr="matrix(1.7656463,0,0,1.7656463,324.90716,255.00942)"

# Simple translation
GRP id=panel tr="t(10,20)"

# Rotation around center (50,50)
CIR id=wheel tr="r(90,50,50)" cx=50 cy=50 r=30

# Scaling
IMG id=logo tr="s(2,2)" x=0 y=0 w=100 h=100

# Combined transformations (scale, then rotate, then translate)
RCT id=box tr="t(100,100) r(45) s(1.5)" x=0 y=0 w=50 h=50

# Skew transformations
GRP id=skewed tr="skewX(30) skewY(15)"
```

#### 4.7.5 Transform Inheritance

Transformations applied to groups affect all child elements. Child transformations are composed with parent transformations.

```bash
GRP id=parent tr="t(50,50)"
RCT id=child x=0 y=0 w=100 h=100           # Actually drawn at (50,50)
CIR id=grandchild tr="t(25,25)" cx=0 cy=0  # Drawn at (75,75) relative to parent
```

#### 4.7.6 Formal ABNF for Transformations

```abnf
transform       = "none" / transform-list
transform-list  = transform *( " " transform )
transform       = matrix-transform / translate-transform / 
                  rotate-transform / scale-transform /
                  skewx-transform / skewy-transform

matrix-transform = "matrix(" number "," number "," number "," number "," number "," number ")"
translate-transform = "t(" number [ "," number ] ")"
rotate-transform   = "r(" number [ "," number "," number ] ")"
scale-transform    = "s(" number [ "," number ] ")"
skewx-transform    = "skewX(" number ")"
skewy-transform    = "skewY(" number ")"

number          = [ "-" ] 1*DIGIT [ "." 1*DIGIT ]
```

---

## 5. Response Codes

| Code    | Meaning | Description                                     |
| :------ | :------ | :---------------------------------------------- |
| `OK`    | Success | Command completed successfully                  |
| `ERROR` | Failure | Command failed; category and description follow |
| `BYE`   | Closing | Session termination acknowledgment              |

---

## 6. Error Categories

| Category    | Description                                   |
| :---------- | :-------------------------------------------- |
| `SYNTAX`    | Malformed command or invalid parameters       |
| `NOT_FOUND` | Referenced element, symbol, or selector does not exist |
| `DUPLICATE` | Element ID already exists                     |
| `PARAM`     | Invalid parameter name or value               |
| `INTERNAL`  | Server-side error                             |

**Image-specific errors:**
- `ERROR PARAM: Invalid Base64 data in href`
- `ERROR PARAM: Unsupported image format`
- `ERROR PARAM: Image data exceeds size limit`

**Transform-specific errors:**
- `ERROR PARAM: Invalid transform syntax`
- `ERROR PARAM: Malformed matrix values`

---

## 7. Formal ABNF Grammar

```abnf
; Basic structure
command         = cmd-name *( " " parameter )
cmd-name        = "VIEW" / "CLR" / "RST" / "BYE" /
                  "RCT" / "CIR" / "ELL" / "LIN" / "PAT" / "PGN" / "PLN" /
                  "TXT" / "IMG" / "GRP" /
                  "SYM" / "SYM-DEL" / "USE" /
                  "ATTR" / "DEL" / "GET" / "EX" /
                  "STY" / "STY-DEL" / "STY-GET" /
                  "EVENT"

parameter       = key "=" value
key             = 1*( ALPHA / "-" )
value           = *VCHAR

; Element types
element-type    = "RCT" / "CIR" / "ELL" / "LIN" /
                  "PAT" / "PGN" / "PLN" /
                  "TXT" / "IMG" / "GRP"

; Common attributes
common-attr     = "id" / "grp" / "pos" / "class" /
                  "fill" / "stroke" / "sw" / "op" / "tr"

; Type-specific attributes
rect-attr       = "x" / "y" / "w" / "h" / "rx" / "ry"
circle-attr     = "cx" / "cy" / "r"
ellipse-attr    = "cx" / "cy" / "rx" / "ry"
line-attr       = "x1" / "y1" / "x2" / "y2"
path-attr       = "d"
poly-attr       = "pts"
text-attr       = "txt" / "fs" / "ff" / "fw" / "ta"
image-attr      = "href" / "pr"
symbol-attr     = "vb" / "pr"
use-attr        = "sym" / "x" / "y" / "w" / "h""
group-attr      = "tr" / "grp" / "pos" / "class" / "fill" / "stroke" / "sw" / "op"

; Values
pos-value       = "f" / "l"
ta-value        = "s" / "m" / "e"
fw-value        = "b" / "n"
pr-value        = "none" / "meet" / "slice"
bool-value      = "t" / "f"

; Image format
href-value      = "data:image/" image-format ";base64," base64-string
image-format    = "png" / "jpeg" / "jpg" / "gif" / "webp" / "svg+xml"
base64-string   = 1*( ALPHA / DIGIT / "+" / "/" / "=" )

; Color formats
color-value     = hex-color / named-color
hex-color       = "#" 6( HEXDIG ) [ 2( HEXDIG ) ]
named-color     = "red" / "blue" / "green" / "yellow" / "black" / "white" / "none"

; Path data
path-data       = 1*( path-command )
path-command    = ( "M" / "L" / "H" / "V" / "C" / "S" / "Q" / "T" / "A" / "Z" ) [ coord-pair ]*
coord-pair      = 1*DIGIT [ "." 1*DIGIT ] *( "," 1*DIGIT [ "." 1*DIGIT ] )

; Transformations
transform       = "none" / transform-list
transform-list  = transform *( " " transform )
transform       = matrix-transform / translate-transform / 
                  rotate-transform / scale-transform /
                  skewx-transform / skewy-transform

matrix-transform = "matrix(" number "," number "," number "," number "," number "," number ")"
translate-transform = "t(" number [ "," number ] ")"
rotate-transform   = "r(" number [ "," number "," number ] ")"
scale-transform    = "s(" number [ "," number ] ")"
skewx-transform    = "skewX(" number ")"
skewy-transform    = "skewY(" number ")"

number          = [ "-" ] 1*DIGIT [ "." 1*DIGIT ]
```

---

## 8. Security Considerations

- Validate `d` attribute in paths to prevent malicious data (e.g., very long strings, recursive patterns).
- Limit `pts` attribute length for polygons/polylines (e.g., max 1000 points).
- Sanitize `vb` (viewBox) values to reasonable ranges.
- **Image decoding:** Servers SHOULD reject malformed or excessively large Base64 images (e.g., > 10 MB).
- **Image type restrictions:** Servers MAY restrict allowed image formats (e.g., only PNG and JPEG).
- **Prevent image bombs:** Decode or validate image dimensions before embedding to avoid memory exhaustion.
- **Transform validation:** Validate matrix values to prevent excessive scaling or invalid transformations.
- **Symbol recursion:** Prevent symbols from referencing themselves (directly or indirectly).

---

## 9. Complete Session Example

```
CLIENT> VIEW w=1280 h=720
SERVER> OK

# Define reusable symbols
CLIENT> SYM id=iconHeart vb="0 0 24 24"
CLIENT> PAT id=heartPath d="M12 21.35l-1.45-1.32C5.4 15.36 2 12.28 2 8.5 2 5.42 4.42 3 7.5 3c1.74 0 3.41.81 4.5 2.09C13.09 3.81 14.76 3 16.5 3 19.58 3 22 5.42 22 8.5c0 3.78-3.4 6.86-8.55 11.54L12 21.35z" fill=red
SERVER> OK

CLIENT> SYM id=button vb="0 0 100 50"
CLIENT> RCT id=btnBg x=0 y=0 w=100 h=50 rx=5 fill=blue
CLIENT> TXT id=btnText x=50 y=25 txt="Click" ta=m fill=white fs=14
SERVER> OK

# Create main scene
CLIENT> GRP id=main tr="t(50,50)"
SERVER> OK

# Use symbols
CLIENT> USE id=heart1 sym=iconHeart x=10 y=10
CLIENT> USE id=heart2 sym=iconHeart x=50 y=10 fill=pink
CLIENT> USE id=btn1 sym=button x=10 y=70
CLIENT> USE id=btn2 sym=button x=10 y=140 w=150 h=60 fill=green
SERVER> OK

# Create primitive elements
CLIENT> RCT id=bg x=0 y=0 w=1280 h=720 fill=#1a1a2e
SERVER> OK

CLIENT> CIR id=dot cx=100 cy=100 r=50 fill=yellow stroke=orange sw=3
SERVER> OK

CLIENT> PGN id=star pts="100,10 40,180 190,60 10,60 160,180" fill=gold stroke=black sw=1
SERVER> OK

CLIENT> PAT id=arrow d="M20 20 L200 20 L110 150 Z" fill=none stroke=cyan sw=4
SERVER> OK

CLIENT> TXT id=title x=640 y=50 txt="Dashboard" fill=white fs=32 ff=Arial ta=m
SERVER> OK

# Apply styles
CLIENT> STY sel=.hl fill=gold stroke=orange sw=3
SERVER> OK

CLIENT> ATTR id=star class=hl
SERVER> OK

# Transformations
CLIENT> GRP id=rotated tr="r(45,200,200)"
CLIENT> USE id=heart3 sym=iconHeart x=200 y=200
SERVER> OK

# Queries
CLIENT> EX id=star
SERVER> OK EX=t

CLIENT> GET id=star
SERVER> OK id=star type=PGN pts="100,10 40,180 190,60 10,60
```