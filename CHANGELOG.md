# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## 0.0.1 — 2026-09-11

The **interface**, before anyone implements it.  Every signature, every
type and every effect row is published; every body is `todo()`, and the
release is stamped `NOT IMPLEMENTED — interface only`.  Adding this
package works and calling it panics.

- Seven modules.  `ipknots` is the validated input; `ipcurve` is the
  curve and everything done to one; `iplinear` and `ipspline` are the
  seven methods; `ipgrid` is the 2-D regular grid; `ipbspline` is the
  basis underneath; `ipfault` is every refusal.
- **Validation is a TYPE, and that is the load-bearing interface.**
  `IpKnots` is a pair of lists that has passed the sortedness,
  finiteness and length checks, `of_lists` is the only way to make one,
  and every constructor takes one.  What that buys is the property the
  whole package is shaped around: `ipcurve.at` answers a `Float` and
  not a `Result<Float, IpFault>`, because by the time a curve exists
  there is nothing left for an evaluation to refuse.  A resample of a
  million points is a loop over floats with no error path in it.
- **`at_checked` is a separate function rather than a `Result` on the
  common path**, so that the caller who chose `IpExtrapError` gets the
  refusal and the hot path keeps no error handling it does not use.
- **One curve type for all seven methods.**  Every method here produces
  a piecewise cubic and they differ only in how the coefficients were
  chosen, so `derivative`, `integral`, `resample`, `is_monotone` and
  `range_of` are written once instead of seven times.
- **There is no extrapolation default.**  scipy's `interp1d` raises,
  `numpy.interp` clamps and `CubicSpline` extrapolates the end
  polynomial — three libraries, three answers, each right for its own
  callers — so `IpExtrap` names four and every constructor takes one.
- **The monotonicity guarantee is checkable rather than asserted.**
  `ipspline.range_of` and `ipspline.is_monotone` compute a piecewise
  cubic's extrema exactly from its coefficients — the knots and the
  roots of each interval's quadratic derivative — rather than by
  sampling, so "did my interpolation overshoot" is a call and not a
  plot.
- **Coefficients are LOCAL to their interval**, `a + b t + c t² + d t³`
  in `t = x - xs[i]`.  With a global `x` the cubic term of an interval
  at `x = 1e6` is `1e18` and the coefficients cancel catastrophically;
  scipy's `PPoly` does the same thing for the same reason.
- **The antiderivative has its own type.**  Integration raises the
  degree and an `IpCurve`'s intervals are cubic, so `IpQuartic` exists
  rather than every cubic carrying a fifth zero coefficient and every
  evaluation doing one more multiply.
- **The 1-D curves take plain lists and the 2-D grid takes an
  `NdFloat`**, and the README argues why that is one rule applied
  twice: a knot list has no shape, and a regular grid IS one.
- **Two knot validators, because the two things differ.**  An
  interpolation knot must be distinct, because every method divides by
  an interval width; a B-spline knot repeats on purpose, and the
  repeat at each end is what makes a clamped curve start at its first
  control point.
- **No `@tier(embedded)` claim and no probe.**  A curve owns its knots
  and coefficients as lists, and a list literal is a heap allocation
  the tier refuses.  A fixed-knot device form — a sensor calibration
  table — is a real thing to want and is a different type; this lane's
  report names it as a row rather than half-claiming it here.
- Every vector is a case with a closed-form answer, a property a method
  promises, or a published rule.  Nothing is a transcription of
  scipy's output at an arbitrary point.
