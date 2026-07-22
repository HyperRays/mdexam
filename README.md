# mdexam — File Format

An **mdexam** file is a plain markdown document. Anything markdown can do —
headings, prose, emphasis, images, LaTeX math — renders as a normal document.
Interactive question widgets are placed with **inline markers** that sit
exactly where the input element should appear, so a question can be put in the
middle of a sentence.

A minimal complete exam:

```markdown
---
title: Warm-up Quiz
---

# Derivatives

Compute $\frac{d}{dx} x^3 =$ [[input 3*x^2]]{points=2}

:::solution
Power rule: bring down the exponent, $3x^{3-1} = 3x^2$.
:::
```
---

## 1. File structure

An exam file has four kinds of content:

| Piece | Syntax |
|---|---|
| Metadata | `--- … ---` at the very top | 
| Markdown | ordinary markdown + `$…$` math | 
| Question markers | `[[type payload]]{attributes}` | 
| Solution blocks | `:::solution … :::` solutions, revealed after grading| 

---

## 2. Metadata

Optional. Simple `key: value` lines between `---` fences on the first line of
the file.

```markdown
---
title: Analysis
show_solutions_after_submit: true
---
```

| Key | Default | Meaning |
|---|---|---|
| `title` | `Untitled exam` | Shown in the exam header and browser tab |
| `show_solutions_after_submit` | `true` | Set to `false` to keep `:::solution` blocks hidden even after grading |

---

## 3. Math in md

Use `$…$` for inline and `$$…$$` for display math; both are rendered with
KaTeX.

```markdown
The matrix $\begin{pmatrix}1&2\\3&4\end{pmatrix}$ is invertible, since

$$\det\begin{pmatrix}1&2\\3&4\end{pmatrix} = 1\cdot 4 - 2\cdot 3 = -2 \neq 0.$$
```

A `$` only opens math block when it is directly followed by a non-space, and only
closes when directly preceded by one — so md like "costs $5 and $ 10" is
left alone.

---

## 4. Question markers

Questions are **inline**:

```
[[type payload]]{attribute attribute …}
```

The `{…}` attribute block is optional. Five types exist:

| Marker | Widget | Checked by |
|---|---|---|
| `[[input ANSWER]]` | free-text box | SymPy, algebraically |
| `[[dropdown a \| *b \| c]]` | select menu | choice index |
| `[[tf true]]` | True / False buttons | boolean |
| `[[mc]]` | radio buttons (pick one) | task list that follows |
| `[[multi]]` | checkboxes (pick several) | task list that follows |

Markers can appear mid-sentence, and several can share a line:

```markdown
The function is [[dropdown *even | odd]] and $f(0) =$ [[input 1]].
```

### 4.1 `[[input …]]` — algebraically checked answer

The payload is the **correct answer in STACK syntax** (section 6). The
student's entry is parsed the same way and compared for algebraic
equivalence, so any equivalent form is accepted.

```markdown
Factor: $x^2 - 1 =$ [[input (x-1)*(x+1)]]{points=2}
```

Students may answer `(x+1)*(x-1)`, `x^2-1`, `(x-1)(x+1)` — all correct.

While typing, students see a live rendered preview of how their input was
understood (shown 1:1, without simplification), which catches bracket and
`*` mistakes before submitting.

**Equations** are accepted up to a nonzero constant factor:

```markdown
Tangent line at $(1,1)$: [[input y = 2*x - 1]]{points=2}
```

`(y-1)=2*(x-1)`, `2y = 4x - 2`, and `i`-scaled complex variants all pass;
`y = 2x` does not.

**Matrices, sets, lists:**

```markdown
$A^{-1} =$ [[input matrix([-2,1],[3/2,-1/2])]]{points=2}
Solutions of $x^2=1$ as a set: [[input {-1, 1}]]
First three primes, in order: [[input [2, 3, 5]]]
```

Matrices compare entrywise (equivalent entries accepted, e.g.
`-1/2*matrix([4,-2],[-3,1])`), sets ignore order, lists respect it.

**Quoting:** the marker ends at the first `]]` at square-bracket depth zero,
so `matrix(...)` and plain lists need nothing special. If an answer ever
contains a literal `]]` that would confuse the scanner, wrap it in quotes:

```markdown
[[input "[[1,2],[3,4]]"]]
```

### 4.2 `[[dropdown …]]` — inline select

Choices are separated by `|`; prefix the correct one with `*`:

```markdown
The series $\sum 1/n$ is [[dropdown convergent | *divergent]]{points=1}
```

### 4.3 `[[tf …]]` — true / false

Payload is `true` or `false` (case-insensitive):

```markdown
Every bounded sequence converges. [[tf false]]{points=1}
```

### 4.4 `[[mc]]` and `[[multi]]` — choice lists

The marker sets the location, points, and mode; the choices come from the
**next task list** after the marker's paragraph. `[x]` marks correct
choices.

```markdown
Which is an antiderivative of $2x$? [[mc]]{points=1 shuffle}

- [x] $x^2 + 7$
- [ ] $2$
- [ ] $2x^2$
```

`[[mc]]` renders radio buttons and expects exactly one `[x]`.
`[[multi]]` renders checkboxes; mark any number of `[x]` and grading is
all-or-nothing (the selected set must match exactly):

```markdown
Which are vector spaces over $\mathbb{R}$? [[multi]]{points=2}

- [x] $\mathbb{R}^3$
- [x] the set of polynomials of degree $\le 2$
- [ ] the set of invertible $2\times 2$ matrices
```

Choice text *may* contain `$…$` math (unlike dropdowns). Add `shuffle` to
randomize choice order per load.

---

## 5. Attributes

Space-separated inside `{…}`. Values with spaces need double quotes. All are
optional.

| Attribute | Example | Meaning |
|---|---|---|
| `points=N` | `points=2.5` | Weight of the question (default `1`) |
| `check=MODE` | `check=cas-equal` | Checking mode for `input` (default `alg-equiv`, see §7) |
| `tol=T` | `tol=0.001` | Tolerance for `check=num` |
| `vars=…` | `vars=x,y` | Declare variables (only needed for the cases in §5.1) |
| `assume="…"` | `assume="x:positive"` | Attach assumptions to variables (§7.1) |
| `wrong="…"` | `wrong="Recheck the sign."` | Feedback shown under the widget when the answer is wrong |
| `shuffle` | `shuffle` | Randomize mc/multi choice order |
| `#id` | `#tangent` | Override the auto id (`q1`, `q2`, …) |

Example using most of them (a real marker must stay on one line):

```markdown
[[input 2*n]]{#even points=1 vars=n assume="n:integer+positive" wrong="An even number is twice an integer."}
```

### 5.1 When you actually need `vars`

Variables are created automatically from whatever names appear in the
expressions, so `vars` is usually unnecessary. Declare it only to:

- attach assumptions: `vars=x assume="x:real"`;
- use `e`, `i`, `j`, or `pi` as a **variable** rather than the constant —
  `vars=i` makes `i` an ordinary symbol (e.g. an index) instead of
  $\sqrt{-1}$.

---

## 6. STACK input syntax (answers and student entries)

Both the `[[input …]]` payload and what students type follow the
[STACK answer-input conventions](https://docs.stack-assessment.org/en/Students/Answer_input/):

| To enter | Type | Notes |
|---|---|---|
| $3x$ | `3*x` or `3x` | implicit multiplication is accepted |
| $x^2$ | `x^2` | negative/fractional powers need brackets: `x^(-2)` |
| $\frac{a+b}{c+d}$ | `(a+b)/(c+d)` | brackets matter |
| $\tfrac14$ | `1/4` | prefer fractions over `0.25` |
| $\pi$, $e$, $i$ | `pi`, `e`, `i` | also `%pi`, `%e`, `%i`, engineer's `j` |
| $1000$ | `1E+3` or `1e3` | scientific notation |
| $\sqrt{x}$, $\sqrt[3]{x}$ | `sqrt(x)`, `root(x,3)` | |
| $\|x\|$ | `abs(x)` | modulus for complex values too |
| $\sin^2 x$ | `sin(x)^2` | |
| $\arcsin x$ | `asin(x)` | likewise `atan`, `acos` |
| $\ln x$, $\log_{10} x$, $\log_a x$ | `ln(x)`, `lg(x)`, `lg(x,a)` | |
| $e^{x}$ | `exp(x)` or `e^x` | |
| $a_b$ | `a_b` | subscripts |
| $y = 2x-1$ | `y=2*x-1` | equations |
| $x \le 5$, $x \ne 1$ | `x<=5`, `x#1` | also `<`, `>`, `>=` |
| $\begin{pmatrix}1&2\\3&4\end{pmatrix}$ | `matrix([1,2],[3,4])` | one list per row |
| $\{1,2,3\}$ | `{1,2,3}` | sets, order-insensitive |
| list $1,2,2$ | `[1,2,2]` | order-sensitive |
| $\bar z$, $\operatorname{Re} z$, $\operatorname{Im} z$ | `conjugate(z)`, `re(z)`, `im(z)` | Maxima's `realpart`/`imagpart` also work |
| $\infty$ | `inf` | |
| Greek letters | `alpha`, `beta`, … | English names |

**Not supported (yet):** compound inequalities with `and`/`or`
(`1<x and x<5`), scientific units, and free-text/string answers.

---

## 7. Checking modes

### `check=alg-equiv` (default)

True algebraic equivalence, decided by SymPy: the student's expression must
equal the reference for all values of the variables.

- `x/2 - sin(x)*cos(x)/2` ≡ `x/2 - sin(2*x)/4` ✓
- Equations are equivalent up to a **nonzero constant factor**
  (`y=x^2-2x+1` ≡ `2y=2x^2-4x+2`), which matches "same equation";
  note this is stricter than "same solution set."
- Inequalities and `#` relations must match in **type** and have equivalent
  sides — `x<5` ≢ `x<=5`, and `2x<10` ≢ `x<5` (sides are compared
  individually, no rescaling).
- Complex-aware: `(1+i)^2` ≡ `2i`, `e^(i*pi/4)` ≡ `sqrt(2)/2+i*sqrt(2)/2`,
  `conjugate(z)*z` ≡ `abs(z)^2`.

Symbols are **complex-valued by default**, which keeps grading honest:
`sqrt(x^2)` ≢ `x` and `conjugate(z)` ≢ `z` unless you opt in via
assumptions.

#### 7.1 Assumptions

`assume="VAR:key"` with keys `positive`, `negative`, `real`, `integer`,
`rational`, `nonzero`, `nonnegative`. Combine keys with `+`, separate
variables with `,`:

```markdown
Simplify $\sqrt{x^2}$ for positive $x$: [[input x]]{vars=x assume="x:positive"}
For real $x$: $\overline{x} =$ [[input x]]{vars=x assume="x:real"}
```

Without the assumption, both of these would (correctly) reject the answer.

### `check=cas-equal`

Structural equality: the answer must be in the **same form**, not merely
equivalent. Use when the form is the point of the question:

```markdown
Write $x^2+2x+1$ as a square: [[input (x+1)^2]]{check=cas-equal}
```

Here `x^2+2*x+1` is rejected even though it is algebraically equal.

### `check=num` with `tol=…`

Numeric comparison: $|\,\text{student} - \text{answer}\,| < \text{tol}$
(magnitude, so complex values work). For questions where a decimal
approximation is expected:

```markdown
$\pi$ to 3 decimals: [[input 3.1416]]{check=num tol=0.001}
```

---

## 8. Worked solutions and feedback

A `:::solution` container is revealed once the exam is graded (unless the metadata disables
this). It holds arbitrary markdown, including math:

```markdown
Compute $\int x e^x\,dx$ (omit the constant): [[input (x-1)*e^x]]{points=3}

:::solution
Integrate by parts with $u = x$, $dv = e^x dx$:
$\int x e^x\,dx = x e^x - \int e^x\,dx = (x-1)e^x$.
:::
```

For short, immediate hints use the `wrong` attribute instead — it appears
inline under the widget only when the answer is incorrect:

```markdown
[[input -2*x*e^(-x^2)]]{wrong="Chain rule: differentiate the outer exp first."}
```

Unanswered questions show "No answer given."; a student entry that fails to
parse shows the parse error alongside your `wrong` feedback.

---

## 9. Grading, points, and ids

- Each question is worth `points` (default 1; fractions allowed). The header
  shows the total; the bottom bar shows the score after **Check answers**.
- `multi` is all-or-nothing; there is no partial credit within a question.
- Questions are auto-numbered `q1`, `q2`, … in document order; `#name`
  overrides. Ids currently matter only for readability and future tooling.
- **Reset** restores the current exam to its unanswered state.

---

## 10. Complete example

```markdown
---
title: Calculus & Complex Numbers — Practice
---

# Part 1 · Real analysis

Let $f(x) = e^{-x^2}$.

$f$ is [[dropdown *even | odd | neither]]{points=1} with maximum
$f(0) =$ [[input 1]]{points=1}, and $f'(x) =$
[[input -2*x*e^(-x^2)]]{points=2 wrong="Chain rule."}

:::solution
$f' = e^{-x^2}\cdot(-2x)$ by the chain rule.
:::

Every continuous function on $[0,1]$ attains a maximum. [[tf true]]{points=1}

Which are true for all differentiable $g$? [[multi]]{points=2}

- [x] $g$ is continuous
- [ ] $g'$ is continuous
- [x] $g$ satisfies the mean value theorem on any $[a,b]$

# Part 2 · Complex numbers

Write $\dfrac{1}{1+i}$ in the form $a+bi$:
[[input 1/2 - i/2]]{points=2}

:::solution
Multiply by the conjugate: $\frac{1}{1+i}\cdot\frac{1-i}{1-i}
= \frac{1-i}{2}$.
:::

For all $z \in \mathbb{C}$: $z\bar z =$
[[input abs(z)^2]]{vars=z points=1}

Complete the square (form matters):
$x^2 - 6x + 9 =$ [[input (x-3)^2]]{check=cas-equal points=1}
```

---

## 11. Practical notes

- **Marker on one line.** A marker plus its `{…}` attributes cannot wrap
  across lines; the surrounding prose can.
- **Floats vs fractions.** Prefer `1/4` over `0.25` in reference answers;
  exact rationals make `alg-equiv` behave predictably. Use `check=num` when
  a decimal approximation is genuinely the expected answer.
