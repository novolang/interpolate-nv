# interpolate-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`. Installing this package works;
calling it panics with `not implemented`.

## What this is

`scipy.interpolate`'s notebook subset: seven ways of drawing a curve
through sorted data, as a **value built once and evaluated many times**.

Linear and nearest; the cubic spline in its three boundary conditions —
natural, clamped, not-a-knot; Akima and PCHIP for data where a classical
spline would ring or overshoot. Then everything a caller does with a
curve once they have one: derivatives to any order, antiderivatives,
definite integrals, resampling onto a new grid, and an exact range and
monotonicity check computed from the coefficients rather than by
sampling.

Beside them: bilinear, bicubic and nearest over a regular 2-D grid of
ndarray-nv values, and the B-spline basis that all of it is a special
case of.

It is for the notebook that has two time series on different sample
grids and wants to subtract them; for the plot that needs a smooth line
through twelve points; for the lookup table with a sensor calibration in
it; for the image resize.

```
novo pkg add interpolate-nv
novo pkg build
novo test
```

## The one example that will work

```novo
use ipknots
use ipspline
use ipcurve

// Two series on different grids become two on one grid, and then they
// can be subtracted.
fn onto_common_grid(xs: [Float], ys: [Float],
                    lo: Float, hi: Float, n: Int) -> Result<[Float], IpFault>
    let knots = ipknots.of_lists(xs, ys)!
    let curve = ipspline.pchip(knots, IpExtrapClamp)!
    ipcurve.resample(curve, lo, hi, n)
```

Two fallible calls and one that answers the data. `of_lists` is where
the knots are checked; `pchip` is where the method is chosen; after that
every evaluation is total.

## The layer, and why

`core` — no effects at all.

Interpolation is arithmetic over numbers the caller already holds. The
knots arrive as a list and the answers go back as one; nothing is read,
nothing is written, no clock is consulted. The budget is `[]` on every
one of the 68 public functions.

**No `@tier(embedded)` claim, and there is no
`tests/embedded_probe.nv`.** The audit's `core-embedded` row passes and
says the package makes no claim. A curve owns its knots and its
coefficients as **lists**, and a list literal is a heap allocation the
tier refuses (SPEC § 14.4).

A device form is a real thing to want — a sensor calibration table is
exactly this, and a microcontroller reading a thermistor does the
lookup on every sample — but it is a different type: a `@value` struct
with a fixed knot count and inline arrays, the way fft-nv's `fftfix`
carries a fixed transform width. **That is a new row**, and this lane's
report names it rather than this package half-claiming it.

## The load-bearing interface

```novo
pub struct IpKnots
pub fn of_lists(xs: [Float], ys: [Float]) -> Result<IpKnots, IpFault> []

pub struct IpCurve
pub fn at(c: IpCurve, x: Float) -> Float []
```

**Validation is a type, and that is what makes evaluation total.**

Every method in this package needs the same three things of its input:
the abscissae are strictly increasing, every number is finite, and the
values list is as long as the knots list. Seven methods would otherwise
check them seven times — or, worse, seven *slightly different* times.

So the check is a type. `IpKnots` is a pair of lists that has passed it,
`of_lists` is the only way to make one, and every curve constructor
takes an `IpKnots` and cannot meet an unsorted knot. What that buys is
the property the whole package is shaped around:

**`ipcurve.at` returns a `Float`, not a `Result<Float, IpFault>`.**

By the time a curve exists there is nothing left for an evaluation to
refuse. Out-of-domain is not an error either — it is a policy the caller
chose, and three of the four policies answer a number. So the inner loop
of a resample over a million points is a loop over floats with no error
path in it at all.

The one exception is `at_checked`, for a caller who chose
`IpExtrapError` — and it is a **separate function** rather than a
`Result` on the common path, precisely so the common path stays free of
it.

A validated-input type is a small idea and it is worth spelling out
because the alternative looks so reasonable: a `spline(xs, ys)` that
returns a `Result` looks like it has the same safety, and it does — for
one method. Seven methods plus a resample plus a grid is nine
validations, and the moment one of them differs the package has two
ideas of what a knot list is.

**`IpCurve` is the second half of that decision.** All seven methods
answer the same type, because every one of them produces a piecewise
cubic and they differ only in how the coefficients were chosen. That is
what lets `derivative`, `integral`, `resample`, `is_monotone` and
`range_of` be written **once** instead of seven times each.

## Which method, in one line each

This is the only thing a caller has to understand about this package.

| method | choose it when | minimum points |
| --- | --- | --- |
| `iplinear.nearest` | the data is a lookup table, not a function | 2 |
| `iplinear.of_knots` | you want the line a plot would draw | 2 |
| `ipspline.not_a_knot` | the data is smooth and nothing is known about the ends | 4 |
| `ipspline.natural` | as above, but the curve should leave the data straight | 3 |
| `ipspline.clamped` | the end slopes are known from physics | 3 |
| `ipspline.akima` | the data has a corner or a step in it | 5 |
| `ipspline.pchip` | overshoot would be *wrong* — a concentration, a probability, a count | 3 |

The three classical splines solve one tridiagonal system so that the
second derivative is continuous everywhere. That is what "smooth" means
and it is also what makes them **global**: moving one knot changes the
curve everywhere, and a single outlier makes the whole curve ring.

The two local methods choose each slope from a few nearby points. No
system, no global coupling, an outlier stays where it is — and the
second derivative is not continuous.

**The case that makes the difference visible** is a flat run, a step,
and another flat run — `[0, 0, 0, 1, 1, 1]`. A natural spline dips
below zero before it rises and overshoots one after. PCHIP does
neither, because it picks each slope so that every interval's cubic is
monotone wherever the data is. The guarantee is exact: **on any interval
where the data is monotone the interpolant is monotone, and the
interpolant never leaves the range of the data.** `ipspline.range_of`
and `ipspline.is_monotone` make it checkable, and the suite checks it.

## Two kinds of input, and which one each curve takes

The 1-D curves take `IpKnots` — two plain `[Float]` lists. The 2-D grid
takes an `NdFloat`. Same rule, applied twice.

**A knot list has no shape.** It is a list, its stride is 1, and an
`NdFloat` would be that list plus a shape, strides and an offset — three
numbers checked on every call and used by nothing. The evaluator's inner
loop is an index and a polynomial, and it would pay for the abstraction
on every read.

**A regular grid IS a shape.** `z[i][j]` is the value at
`(x[i], y[j])`, and the two extents have to agree with the two axis
lists. Handing that over as a flat list plus a row count would be this
package asking a caller to carry a shape beside a buffer, which is
precisely what `NdFloat` is.

The same argument fft-nv makes for its 1-D and 2-D halves, and for the
same reason.

## The reference implementation, and what is specification

`scipy.interpolate` is the reference. The distinction matters because it
decides what a test may assert.

**Specification, and binding on this package**

- **Every method passes through its knots.** `at(c, xs[i]) == ys[i]`,
  for all seven. That is what interpolation means, and it is the first
  thing a wrong implementation breaks.
- **Every method reproduces a straight line exactly**, and the cubic
  ones reproduce a cubic where their boundary condition allows it —
  not-a-knot does, natural and clamped do not, and that is a fact about
  the boundary conditions rather than about the implementation.
- **PCHIP's monotonicity guarantee**, from Fritsch and Carlson's paper.
  Exact, not approximate.
- **Akima's minimum of five points**, which is the width of its own
  formula.
- **Not-a-knot's minimum of four**, because the condition is about the
  second and second-to-last knots and with three points those are the
  same knot.
- **The B-spline basis is a partition of unity**, and only `degree + 1`
  of it is non-zero at any point. Both are theorems, and both are
  asserted.
- **A clamped B-spline knot vector starts and ends at its control
  points.** Without the end repeats a B-spline starts somewhere inside
  its control polygon, which is the first surprise everybody meets.

**scipy's own choices, which this package follows and a test may not
treat as correctness**

- **That not-a-knot is the natural default for a cubic spline.**
  `CubicSpline`'s default; MATLAB's `spline` agrees; it is a
  convention, not a theorem. This package has no default at all — the
  constructor names the condition.
- **Where the extrapolation default is.** scipy's `interp1d` raises,
  `numpy.interp` clamps, `CubicSpline` extrapolates the end
  polynomial. Three libraries, three answers, each right for its own
  callers — so `IpExtrap` names four and **every constructor takes
  one**.
- **Bicubic's boundary handling.** This package **reflects** the grid
  where there are not sixteen points, which is `map_coordinates`'s
  `mode="reflect"`; clamping is equally defensible and makes the
  surface flat at the edges.
- **The nearest tie rule.** Half-way rounds **up** — toward the right
  knot — which is what `kind="nearest"` does. A caller sampling a step
  function on a grid will land on every midpoint, so the rule being
  *stated* matters more than which one it is.

So the correctness condition is **the interpolant passes through the
data, reproduces what its method is exact on, and keeps the property its
method promises.** A test that hard-coded `scipy.interpolate`'s output
at an arbitrary point would be asserting against a transcription.

## Accuracy, and what the tests hold

`1e-12` on an evaluation — a handful of flops above the
double-precision floor. `1e-10` on anything that came through the
tridiagonal solve a classical spline needs.

Both are enormous compared with the error a wrong method produces: an
interpolation that is wrong is wrong by O(1) — it misses a knot, it
overshoots, it has the wrong slope — not by `1e-9`.

The suite is built to make that so, by asserting **properties rather
than transcribed outputs**: through the knots, exact on a line, PCHIP
inside the data's range, the B-spline basis summing to one, the
antiderivative of a constant being a ramp, the integral of a reversed
interval being negated.

## Deliberately not here

- **Scattered-data interpolation** — `griddata`, natural neighbour,
  radial basis functions. They need a triangulation, which is a bigger
  subject than everything above it put together. `ipgrid` is
  `RegularGridInterpolator`, not `griddata`. **A missing row.**
- **Smoothing splines** — `UnivariateSpline`, `splrep` with `s > 0`.
  They need a penalty parameter and a cross-validation rule to choose
  it, which is a statistics question rather than an interpolation one.
  **A missing row.**
- **B-spline FITTING.** `ipbspline.collocation` builds the matrix a
  least-squares fit runs against, and stops there: the `lstsq` is
  linalg-nv's. What is missing is the banded solver that would exploit
  the matrix's structure — `collocation` answers a dense matrix because
  linalg-nv is dense. **A missing row.**
- **`kind="previous"`, `kind="next"`, `kind="nearest-up"`** — scipy's
  three other step variants. `iplinear.nearest` is the one a lookup
  table wants; the others are a line each and can be added without a
  design.
- **Higher-dimensional grids.** `ipgrid` is 2-D, because the notebook
  cases are images and rasters. An N-D regular grid is the same
  algorithm with a recursive reduction, and it is a different type.
- **Interpolation in more than one output dimension** — a parametric
  curve through points in the plane. It is two curves over a common
  parameter, which a caller builds in two lines from what is here.
- **Complex-valued interpolation.** novo-lang has no complex type; the
  real and imaginary parts are two curves.

## Status

Every function is `todo()`. `novo test` runs the API suite, and every
assertion in it reaches `not implemented: interpolate-nv.<module>.<fn>`
— which is the expected result until the bodies land, and is what makes
the suite a description of the interface rather than of nothing.
`novo test --isolate` is the readable form: one verdict per test, naming
the function it stopped at.

| module | public types | functions | constants | implemented |
| --- | --- | --- | --- | --- |
| `ipbspline` | 3 | 12 | 1 | no |
| `ipcurve` | 4 | 16 | 0 | no |
| `ipfault` | 1 | 2 | 0 | no |
| `ipgrid` | 1 | 13 | 0 | no |
| `ipknots` | 1 | 11 | 0 | no |
| `iplinear` | 0 | 4 | 0 | no |
| `ipspline` | 1 | 10 | 0 | no |
| **total** | **11** | **68** | **1** | **no** |
