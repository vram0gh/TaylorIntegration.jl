# [Optimizing: `@taylorize`](@id taylorize)

Here, we describe the use of the macro [`@taylorize`](@ref), which
parses the functions containing the ODEs to be integrated, allowing
[`taylorinteg`](@ref) and [`lyap_taylorinteg`](@ref) to be sped up.

!!! warning
    The macro [`@taylorize`](@ref) is still in an experimental state;
    be cautious of the resulting integration, which has to be tested
    carefully.

## [Some context and the idea](@id idea)

The way in which [`taylorinteg`](@ref) works by default is repeatedly calling
the function where the ODEs of the problem are defined, in
order to compute the recurrence relations that are used to construct
the Taylor expansion of the solution. This is done for each order of
the series in [`TaylorIntegration.jetcoeffs!`](@ref). These computations are
not optimized: they waste memory due to repeated allocations of some temporary
arrays, and perform some operations whose result has been previously
computed.

Here we describe one way to optimize this: The idea is to replace the
default method of [`TaylorIntegration.jetcoeffs!`](@ref) by another
method (same function name) which is called by dispatch, and that in principle
performs better.
The new method is constructed specifically for the specific function
defining the equations of motion by parsing its expression. This new
method performs in principle *exactly* the same operations, but avoids
the extra allocations; to do the latter, the macro also creates an *internal*
function `TaylorIntegration._allocate_jetcoeffs!`, which allocates all temporary
`Taylor1` objects as well as the declared `Array{Taylor1,1}`s.


## An example

In order to explain how the macro works, we shall use as an example the
[mathematical pendulum](@ref pendulum). First, we carry out the integration
using the default method, as described [before](@ref pendulum).

```@example taylorize
using TaylorIntegration

function pendulum!(dx, x, p, t)
    dx[1] = x[2]
    dx[2] = -sin(x[1])
    return dx
end

# Initial time (t0), final time (tf) and initial condition (q0)
t0 = 0.0
tf = 10000.0
q0 = [1.3, 0.0]

# The actual integration
t1, x1 = taylorinteg(pendulum!, q0, t0, tf, 25, 1e-20, maxsteps=1500); # warm-up run
e1 = @elapsed taylorinteg(pendulum!, q0, t0, tf, 25, 1e-20, maxsteps=1500);
all1 = @allocated taylorinteg(pendulum!, q0, t0, tf, 25, 1e-20, maxsteps=1500);
e1, all1
```

We note that the initial number of methods defined for
`TaylorIntegration.jetcoeffs!` is 2.
```@example taylorize
length(methods(TaylorIntegration.jetcoeffs!)) == 2 # initial value
```
Similarly, the number of methods for `TaylorIntegration._allocate_jetcoeffs!` is
also 2.
```@example taylorize
length(methods(TaylorIntegration._allocate_jetcoeffs!)) == 2 # initial value
```
Using `@taylorize` will increase this number by creating a new method for these
two functions.

The macro [`@taylorize`](@ref) is intended to be used in front of the function
that implements the equations of motion. The macro does the following: it
first parses the function as it is, so the integration can be computed
using [`taylorinteg`](@ref) as above, by explicitly using the keyword
argument `parse_eqs=false`; this also declares the function of the ODEs, whose name
is used for parsing. It then creates and evaluates a new method of
[`TaylorIntegration.jetcoeffs!`](@ref), which is the specialized method
(through `Val`) on the specific function passed to the macro as well as a specialized
`TaylorIntegration._allocate_jetcoeffs!`.

```@example taylorize
@taylorize function pendulum!(dx, x, p, t)
    dx[1] = x[2]
    dx[2] = -sin(x[1])
    return dx
end

println(methods(pendulum!))
```

```@example taylorize
println(methods(TaylorIntegration.jetcoeffs!))
```

We see that there is only one method of `pendulum!`, and
there is a *new* method (three in total) of `TaylorIntegration.jetcoeffs!`,
whose signature appears
in this documentation as `Val{Main.ex-taylorize.pendulum!}`. It is
a specialized version for the function `pendulum!` (with some extra information
about the module where the function was created). This method
is selected internally if it exists (default), exploiting dispatch, when
calling [`taylorinteg`](@ref) or [`lyap_taylorinteg`](@ref); to integrate
using the hard-coded (default) method of [`TaylorIntegration.jetcoeffs!`](@ref) of the
integration above, the keyword argument `parse_eqs` has to be set to `false`.
Similarly, one can check that there exists a new method of
`TaylorIntegration._allocate_jetcoeffs!`

Now we carry out the integration using the specialized method; note that we
use the same instruction as above; the default value for the keyword argument `parse_eqs`
is `true`, so we may omit it.

```@example taylorize
t2, x2 = taylorinteg(pendulum!, q0, t0, tf, 25, 1e-20, maxsteps=1500); # warm-up run
e2 = @elapsed taylorinteg(pendulum!, q0, t0, tf, 25, 1e-20, maxsteps=1500);
all2 = @allocated taylorinteg(pendulum!, q0, t0, tf, 25, 1e-20, maxsteps=1500);
e2, all2
```

We note the difference in the performance and allocations:
```@example taylorize
e1/e2, all1/all2
```

We can check that both integrations yield the same results.
```@example taylorize
t1 == t2 && x1 == x2
```

As stated above, in order to allow opting out of using the specialized method
created by [`@taylorize`](@ref), [`taylorinteg`](@ref) and
[`lyap_taylorinteg`](@ref) recognize the keyword argument `parse_eqs`;
setting it to `false` causes the standard method to be used.
```@example taylorize
taylorinteg(pendulum!, q0, t0, tf, 25, 1e-20, maxsteps=1500, parse_eqs=false); # warm-up run

e3 = @elapsed taylorinteg(pendulum!, q0, t0, tf, 25, 1e-20, maxsteps=1500, parse_eqs=false);
all3 = @allocated taylorinteg(pendulum!, q0, t0, tf, 25, 1e-20, maxsteps=1500, parse_eqs=false);
e1/e3, all1/all3
```

We now illustrate the possibility of exploiting the macro
when using `TaylorIntegration.jl` from `DifferentialEquations.jl`.
```@example taylorize
using DiffEqBase

prob = ODEProblem(pendulum!, q0, (t0, tf), nothing) # no parameters
solT = solve(prob, TaylorMethod(25), abstol=1e-20, parse_eqs=true); # warm-up run
e4 = @elapsed solve(prob, TaylorMethod(25), abstol=1e-20, parse_eqs=true);

e1/e4
```
Note that there is an additional marginal cost to using `solve` in comparison
with `taylorinteg`.

The speed-up obtained comes from the design of the new (specialized) method of
`TaylorIntegration.jetcoeffs!` as described [above](@ref idea): it avoids some
allocations and some repeated computations. This is achieved by knowing the
specific AST of the function of the ODEs integrated, which is walked
through and *translated* into the actual implementation, where some
required auxiliary arrays are created, and the low-level functions defined in
`TaylorSeries.jl` are used.
For this, we heavily rely on [`Espresso.jl`](https://github.com/dfdx/Espresso.jl) and
some metaprogramming; we thank Andrei Zhabinski for his help and comments.

The new `TaylorIntegration.jetcoeffs!` and `TaylorIntegration._allocate_jetcoeffs!` method can be inspected by
constructing the expression corresponding to the function, and using
[`TaylorIntegration._make_parsed_jetcoeffs`](@ref):

```@example taylorize
ex = :(function pendulum!(dx, x, p, t)
    dx[1] = x[2]
    dx[2] = -sin(x[1])
    return dx
end)

new_ex1, new_ex2 = TaylorIntegration._make_parsed_jetcoeffs(ex)
```

The first function has a similar structure as the hard-coded method of
`TaylorIntegration.jetcoeffs!`, but uses low-level functions in `TaylorSeries`
(e.g., `sincos!` above); all temporary `Taylor1` objects as well as declared
arrays are allocated once by `TaylorIntegration._allocate_jetcoeffs!`.
More complex functions quickly become very difficult to read. Note that,
if necessary, one can further optimize `new_ex` manually.


## Limitations and some advice

The construction of the internal function obtained by using
[`@taylorize`](@ref) is somewhat complicated and limited. Here we
list some limitations and provide some advice.

- It is best to have expressions which involve two arguments at most, which
  requires using parentheses appropriately: For example, `res = a+b+c` should
  be written as `res = (a+b)+c`.

- Updating operators such as `+=`, `*=`, etc., are not supported. For
  example, the expression `x += y` is not recognized by `@taylorize`. Likewise,
  expressions such as `x = x+y` are not supported by `@taylorize` and should be
  replaced by equivalent expressions in which variables appear only on one side
  of the assignment; e.g. `z = x+y; x = z`.

- The macro allows the use of array declarations through `Array`, but other ways
  (e.g. `similar`) are not yet implemented.

- Avoid using variables prefixed by an underscore, in particular `_T`, `_S` and
  `_N`; using them may lead to name collisions with some internal variables.

- Broadcasting is not recognized by `@taylorize`.

- The macro may be used in combination with the [common interface with
  `DifferentialEquations.jl`](@ref diffeqinterface), for functions using the
  `(du, u, p, t)` in-place form, as we showed above. Other extensions allowed by
  `DifferentialEquations` may not be able to exploit it.

- `if-else` blocks are recognized in their long form, but short-circuit
  conditional operators (`&&` and `||`) are not.  When comparing to a
  Taylor expansion, use operators such as `iszero` for `if-else` tests
  rather than comparing against numeric literals.

- Input and output lengths should be determined at the time of `@taylorize`
  application, not at runtime.  Do not use the length of the input as an
  implicit indicator of whether to write all elements of the output.  If
  conditional output of auxiliary equations is desired, use explicit methods,
  such as through parameters or by setting auxiliary `t0` vector elements
  to zero, and assigning unwanted auxiliary outputs zero.

- Expressions which correspond to function calls (so the `head` field is
  `:call`) which are not recognized by the parser are simply copied. The
  heuristics used, especially for vectors, may not work for all cases.

- Use `local` for internal parameters (simple constant values); this improves
  performance. Do not use it if the variable is Taylor expanded during the integration.

- `@taylorize` supports multi-threading via `Threads.@threads`. **WARNING**:
  this feature is experimental. Since thread-safety depends on the definition
  of each ODE, we cannot guarantee the resulting code to be thread-safe in
  advance. The user should check the resulting code to ensure that it is indeed
  thread-safe. For more information about multi-threading, the reader is
  referred to the [Julia documentation](https://docs.julialang.org/en/v1/manual/multi-threading/#man-multithreading).

It is recommended to skim `test/taylorize.jl`, which implements different
cases and highlights cases where the macro doesn't work and how to solve the problem.

Please report any problems you may encounter.
