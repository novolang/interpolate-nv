# interpolate-nv

Interpolation draws a curve through a set of measured points, so that a
value can be read at an abscissa where nothing was measured. This
package brings seven methods of doing that to novo-lang, together with
the calculus and resampling a caller does afterwards. Its reference is
[`scipy.interpolate`](https://docs.scipy.org/doc/scipy/reference/interpolate.html).
Its two-dimensional grid reads matrices from
[ndarray-nv](https://novo-lang.org/packages/ndarray-nv).

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What the pieces are

A **knot** is one measured point: an abscissa and the value there. The
abscissae must be strictly increasing, and `IpKnots` is a pair of lists
that has been checked to be.

A **curve** is what a method builds from knots. Every method here
produces a **piecewise cubic**: one cubic polynomial per interval
between two knots, joined at the knots. The methods differ only in how
they choose the coefficients, so they all answer the same type,
`IpCurve`.

A **spline** is a piecewise polynomial whose pieces are joined smoothly.
The classical cubic spline makes the second derivative continuous
everywhere, which takes one equation per knot and therefore one solve of
a tridiagonal system. That makes it **global**: moving one knot changes
the curve everywhere, and one outlier makes the whole curve ripple.

A **boundary condition** is the extra rule a classical spline needs at
each end, where there is no next knot to be smooth against. **Natural**
sets the second derivative to zero, so the curve leaves the data
straight. **Clamped** sets the end slopes to values the caller supplies.
**Not-a-knot** requires the third derivative to be continuous at the
second and second-to-last knots.

A **local** method chooses each knot's slope from a few nearby points,
with no system to solve, so an outlier stays where it is. **Akima** and
**PCHIP** are the two here. Their second derivative is not continuous.

**Overshoot** is a curve leaving the range of the data between two
knots. A classical spline does it; PCHIP is built not to.

**Extrapolation** is what a curve answers outside the range of its
knots. It is a policy the caller names when the curve is built.

| Policy | Outside the knots the curve answers |
| --- | --- |
| `IpExtrapClamp` | the nearest end value |
| `IpExtrapLinear` | a straight line at the end slope |
| `IpExtrapPolynomial` | the end interval's own cubic, continued |
| `IpExtrapFill` | a value the caller chose |
| `IpExtrapError` | nothing; `at_checked` refuses |

## Install

```
novo pkg add interpolate-nv
```

## Example

```novo
use std.list
use ipknots
use ipspline
use ipcurve

fn main() [io]
    // Six measurements: a flat run, a step, another flat run.
    let xs = [0.0, 1.0, 2.0, 3.0, 4.0, 5.0]
    let ys = [0.0, 0.0, 0.0, 1.0, 1.0, 1.0]

    // `of_lists` is where the knots are checked. It is the only way to
    // build the value every curve constructor takes.
    match ipknots.of_lists(xs, ys)
        Err(e) => println(e.message())
        Ok(k) =>
            // PCHIP never leaves the range of the data, so this curve
            // does not dip below zero before the step.
            match ipspline.pchip(k, IpExtrapClamp)
                Err(e) => println(e.message())
                Ok(c) =>
                    // Evaluation cannot fail, so `at` answers a plain Float.
                    println("half way up the step: ${ipcurve.at(c, 2.5)}")

                    // The area under the curve between two abscissae.
                    println("area from 0 to 5: ${ipcurve.integral(c, 0.0, 5.0)}")

                    // Twenty-one points on one grid, for plotting or for
                    // subtracting another series from.
                    match ipcurve.resample(c, 0.0, 5.0, 21)
                        Err(e)  => println(e.message())
                        Ok(out) => println("${list.len(out)} resampled points")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: interpolate-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `ipknots` | The checked pair of lists every curve is built from, the two ways to make one, the lookups over it, and a `linspace`. |
| `iplinear` | The straight-line curve and the nearest-value curve, and two one-call forms for a caller with no curve to keep. |
| `ipspline` | The three classical splines, Akima, PCHIP, a Hermite curve from slopes the caller supplies, the two slope rules on their own, and the exact range and monotonicity checks. |
| `ipcurve` | The curve itself: evaluation, the method and extrapolation it carries, the coefficients of one interval, derivatives, the antiderivative and its quartic, definite integrals, and the two resamplings. |
| `ipgrid` | A regular two-dimensional grid of ndarray-nv values, with nearest, bilinear and bicubic evaluation and resampling, and the row and column slices as curves. |
| `ipbspline` | The B-spline basis all of it is a special case of: a knot vector, the basis at a point, evaluation and differentiation of a control polygon, sampling, and the collocation matrix a least-squares fit runs against. |
| `ipfault` | Every reason a construction refuses, as one enum with ten variants, and two questions to ask of one. |

## How to choose an entry point

Choose the method by what the data is.

| Method | Choose it when | Fewest knots |
| --- | --- | --- |
| `iplinear.nearest` | the data is a lookup table rather than a function | 2 |
| `iplinear.of_knots` | you want the line a plot would draw | 2 |
| `ipspline.not_a_knot` | the data is smooth and nothing is known about the ends | 4 |
| `ipspline.natural` | as above, and the curve should leave the data straight | 3 |
| `ipspline.clamped` | the end slopes are known from the physics | 3 |
| `ipspline.akima` | the data has a corner or a step in it | 5 |
| `ipspline.pchip` | overshoot would be wrong: a concentration, a probability, a count | 3 |

The case that shows the difference is a flat run, a step and another
flat run. A natural spline dips below the lower level before it rises
and overshoots the upper one after. PCHIP does neither.

**Build a curve when you will evaluate it more than once.**
`iplinear.interp` and `iplinear.interp_many` are the one-call forms for
a caller who will not come back.

**The one-dimensional curves take two plain lists and the
two-dimensional grid takes a shaped array.** A knot list is a list. A
regular grid is a shape: the value at row `i`, column `j` belongs to
abscissa `xs[i]` and ordinate `ys[j]`, and the two extents have to agree
with the two axis lists.

## The rules a user needs

1. **Knots are checked once, when `IpKnots` is built.** `of_lists`
   requires strictly increasing abscissae, all values finite, and the
   two lists of equal length. `sorted_by_x` sorts first. Nothing
   downstream re-checks.
2. **`ipcurve.at` answers a `Float` and cannot fail.** By the time a
   curve exists there is nothing left to refuse, and four of the five
   extrapolation policies answer a number outside the knots. A resample
   over a million points is a loop with no error path in it.
3. **`at_checked` is the call for `IpExtrapError`.** It is a separate
   function so that the common path stays free of a `Result`.
4. **Every constructor takes an extrapolation policy and none is the
   default.** scipy's `interp1d` raises, `numpy.interp` clamps, and
   scipy's `CubicSpline` continues the end polynomial. Three libraries
   give three answers, each right for its own callers.
5. **There is no default boundary condition either.** scipy's
   `CubicSpline` and MATLAB's `spline` both default to not-a-knot. Here
   the constructor names the condition.
6. **Every method passes through its knots.** Evaluating at a knot's
   abscissa answers that knot's value, for all seven.
7. **Every method reproduces a straight line exactly.** Only not-a-knot
   also reproduces a true cubic; natural and clamped do not, which is a
   fact about their boundary conditions.
8. **PCHIP's guarantee is exact.** On any interval where the data is
   monotone the curve is monotone, and the curve never leaves the range
   of the data. It comes from Fritsch and Carlson's construction.
   `ipspline.range_of` computes the extrema from the coefficients rather
   than by sampling, and `ipspline.is_monotone` answers the other half.
9. **Akima needs five knots and not-a-knot needs four.** Akima's formula
   is five points wide. Not-a-knot's condition is about the second and
   second-to-last knots, and with three knots those are the same knot.
   Too few is `IpTooFewKnots`, naming the method, the need and the
   count.
10. **A nearest-value tie rounds towards the right-hand knot.** A caller
    sampling a step function on a regular grid lands on every midpoint,
    so which rule applies matters.
11. **Bicubic evaluation reflects the grid at its edges**, where there
    are not sixteen surrounding points. This is what scipy's
    `map_coordinates` calls `mode="reflect"`.
12. **A clamped B-spline knot vector repeats its first and last knots.**
    Without the repeats the curve starts somewhere inside its control
    polygon rather than at the first control point.
    `ipbspline.uniform_clamped` builds one with the repeats.
13. **The B-spline basis sums to one at every point, and only
    `degree + 1` of its functions are non-zero there.** Both are
    theorems, and `ipbspline.basis_at` answers only the non-zero ones.
14. **An antiderivative is a quartic, not a cubic.** `ipcurve.
    antiderivative` answers an `IpQuartic`, which has its own evaluation
    function.

## What is not included

- **Scattered-data interpolation**, the equivalent of scipy's
  `griddata`, natural-neighbour interpolation and radial basis
  functions. They need a triangulation, which is a larger subject than
  everything here put together. `ipgrid` is a regular grid only.
- **Smoothing splines**, where the curve is allowed to miss the data.
  They need a penalty parameter and a rule for choosing it, which is a
  statistics question.
- **Fitting a B-spline.** `ipbspline.collocation` builds the matrix a
  least-squares fit runs against and stops there. The solve is
  [linalg-nv](https://novo-lang.org/packages/linalg-nv)'s, and the
  matrix is dense because linalg-nv is dense, so nothing here exploits
  its banded structure.
- **scipy's `previous`, `next` and `nearest-up` step rules.**
  `iplinear.nearest` is the one a lookup table wants.
- **Grids of more than two dimensions.** An N-dimensional regular grid
  is the same algorithm with a recursive reduction, and it is a
  different type.
- **A curve through points in the plane**, parameterised by something
  other than the abscissa. It is two curves over a common parameter,
  which a caller builds in two lines.
- **Complex-valued interpolation.** novo-lang has no complex type, so
  the real and imaginary parts are two curves.
- **A microcontroller build.** A curve owns its knots and its
  coefficients as lists, and a build for a device with no heap allocator
  refuses a list literal (SPEC section 14.4). A sensor calibration table
  on a device wants a value type with a fixed knot count and inline
  arrays, which is a different type. This package makes no device claim
  and ships no device probe.

## Related packages

- [ndarray-nv](https://novo-lang.org/packages/ndarray-nv) holds the
  regular grid `ipgrid` reads, and its `linspace` builds an axis.
- [linalg-nv](https://novo-lang.org/packages/linalg-nv) is where the
  least-squares solve behind a B-spline fit lives.
- [stats-nv](https://novo-lang.org/packages/stats-nv) summarises the
  series a resample puts on a common grid.
- [plot-nv](https://novo-lang.org/packages/plot-nv) draws the smooth
  line a curve produces.
- [fft-nv](https://novo-lang.org/packages/fft-nv) is the other way to
  resample a signal, through its frequencies rather than through a
  polynomial.

## Tests

```bash
novo test tests/interpolate_tests.nv     # 21 tests: the exact cases and the properties
```

Interpolation has closed-form answers more often than most numerics, so
most of the suite is exact rather than approximate. A test that recorded
`scipy.interpolate`'s output at an arbitrary point would be asserting
this implementation against a transcription.

What is asserted: that every method passes through its knots, that every
method reproduces a straight line exactly, that not-a-knot reproduces a
true cubic where natural and clamped do not, that a natural spline's
second derivative is zero at the ends, that PCHIP stays inside the range
of the data as `range_of` computes it, that a linear curve's integral is
the area of a trapezium, that the integral of a reversed interval is
negated, and that the B-spline basis sums to one.

Two tolerances are used. An evaluation is asserted to within 1e-12,
because it is a handful of operations above a floor near 1e-16. Anything
that came through the tridiagonal solve a classical spline needs is
asserted to within 1e-10. An interpolation that is wrong is wrong by a
whole unit: it misses a knot, it overshoots, or it has the wrong slope.

The tests compile today and fail at run, each on the
`not implemented: interpolate-nv.<module>.<fn>` panic that is its body.
That is the expected state of an interface release. They turn green one
at a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `ipbspline.IPBSPLINE_MAX_DEGREE` | yes (it is a constant) |
| `ipknots.IpKnots`, `ipcurve.IpCurve`, `.IpExtrap`, `.IpMethod`, `.IpQuartic` | declared |
| `ipspline.IpRange`, `ipgrid.IpGrid`, `ipfault.IpFault` | declared |
| `ipbspline.IpBSplineKnots`, `.IpSpan`, `.IpBasis` | declared |
| `ipknots.of_lists`, `.sorted_by_x`, `.count`, `.xs`, `.ys`, `.lo`, `.hi` | no |
| `ipknots.interval`, `.contains`, `.is_uniform`, `.linspace` | no |
| `iplinear.of_knots`, `.nearest`, `.interp`, `.interp_many` | no |
| `ipspline.natural`, `.clamped`, `.not_a_knot`, `.akima`, `.pchip`, `.hermite` | no |
| `ipspline.pchip_slopes`, `.akima_slopes`, `.is_monotone`, `.range_of` | no |
| `ipcurve.at`, `.at_checked`, `.at_many`, `.slope_at`, `.quartic_at`, `.integral` | no |
| `ipcurve.knots`, `.method`, `.method_name`, `.extrapolation`, `.with_extrapolation`, `.interval_coefficients` | no |
| `ipcurve.derivative`, `.antiderivative`, `.resample`, `.resample_onto` | no |
| `ipgrid.of_grid`, `.rows`, `.cols`, `.xs`, `.ys`, `.values`, `.row_knots`, `.column_knots` | no |
| `ipgrid.nearest_at`, `.bilinear_at`, `.bicubic_at`, `.resample_bilinear`, `.resample_bicubic` | no |
| `ipbspline.knots`, `.uniform_clamped`, `.basis_count`, `.degree`, `.knot_vector`, `.domain` | no |
| `ipbspline.basis_at`, `.eval_at`, `.eval_many`, `.derivative_at`, `.sample`, `.collocation` | no |
| `ipfault`'s ten variants, `.is_construction_fault`, `.knot_index` and its `Error` implementation | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
