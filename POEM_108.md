POEM ID:  108
Title:  Functional interface for driver-level callbacks 
authors: @dingraha (Daniel Ingraham)  
Competing POEMs:   
Related POEMs:   
Associated implementation PR: [#3690](https://github.com/OpenMDAO/OpenMDAO/pull/3690)

Status:

- [x] Active
- [ ] Requesting decision
- [ ] Accepted
- [ ] Rejected
- [ ] Integrated


## Motivation
OpenMDAO's highest-level class in the `Problem`, which is an expression of an optimization problem, and mainly consists of the combination of two classes: a `System` and `Driver`.
The `System` contains the math specific to the optimization problem the `Problem` will solve, and the `Driver` solves the optimization problem using a particular optimization algorithm/"program".
OpenMDAO [supports many `Driver`s](https://openmdao.org/newdocs/versions/latest/features/building_blocks/drivers/index.html), each itself supporting a number of optimization algorithms, giving users many choices for the optimization algorithm they'll use for their problem.
On the other hand, creating a `Driver` is a non-trivial task, making it difficult for users to 

1. solve existing OpenMDAO models with new optimizers that don't have an associated `Driver`,
2. integrate an OpenMDAO model in a different framework in a clean way.

This POEM will describe a simple "functional[^function]" interface to an OpenMDAO `Problem` that will hopefully solve these... problems.

[^function]: Likely not a function in the mathematical sense.

## Description
We want something like this:

```python
prob = om.Problem()
# Do the usual stuff to define the Problem.

# Decide which input and output variables we're interested in.
# (Not specifying `input_vars` defaults to all design variables.
# Not specifying `output_vars` defaults to the objective and all constraints.)
input_vars = ["x1", "x2", "x3"]
output_vars = ["y1", "y2"]
# Create the callable object that will compute outputs wrt inputs.
f = prob.get_callback(form="f", input_vars=input_vars, output_vars=output_vars)

# Create a new `numpy.ndarray` that's the correct length to use with `f`, and is initialized with the current values in `prob`.
x = f.create_input_vector()

# Set `x` as required.

# From inputs `x` get outputs.
y = f(x)

# Can also set outputs in-place, to reuse memory.
y2 = f.create_output_vector()
f(x, y=y2)

# Get a callback function that calculates the total derivatives.
dfdx = prob.get_callback(form="dfdx")
x = dfdx.create_input_vector()
J = dfdx(x)

# Can also set Jacobian in-place, to reuse memory.
J2 = dfdx.create_jacobian()
dfdx(x, J=J2)

# Create combined callback function that will return both outputs and Jacobian.
fdfdx = prob.get_callback(form="fdfdx", input_vars=input_vars, output_vars=output_vars)
x = fdfdx.create_input_vector()
y, J = fdfdx(x)

# Again, can set things in-place.
y3 = f.create_output_vector()
J3 = fdfdx.create_jacobian()
fdfdx(x, y=y3, J=J3)
```

### TODOs
* Finish "first try" at functional interface
  * ignore units
  * ignore scaling
  * do indices myself in the functional interface class
  * write tests, duh
  * Advice from Bret: make sure to cache the `_TotalJacInfo` object in the functional interface wrapper class (expensive to create).
  * Determine how to deal with sparse Jacobians
  * Check if constraints are being handled properly (violation vs raw value).
* Check that I'm handling indices properly.
  * `flat_indices = False` case
  * `flat_indices = True` case
* Add units support
  * Easy for input and output vectors, since we can just use `Problem.get_val` and `Problem.set_val` with the `units` argument.
  * ATM would need to manually scale the Jacobian we get from `Problem.compute_totals`.
* Add `driver_scaling` support
  * This should conflict with including units.
* Provide a way for users to get back part of the input or output vectors?
  Might be tricky when using indices.

#### `Problem.compute_totals` wishlist/concerns/questions
* How should constraint values be expressed?
  A user might not care about the constraint value themselves, but instead want them in terms of a violation aka `g = y - y_target`.
  Should that be an option?
* What about linear constraints?
  `Problem.compute_totals` appears to not return derivatives of those constraints by default.
  Maybe that's OK?
* How to properly handle indices?
  If we ask for derivatives wrt design variables, objectives, constraints, `compute_totals` will use the indices given for the `add_design_var`, `add_constraint`, `add_objective`.
  But if we ask for derivatives wrt other stuff we always get the "whole thing," and there doesn't appear to be a way for the user to specify that with `compute_totals`.
  This appears to be a limitation of `compute_totals` that we'd need to fix.
* What I wish `compute_totals` could do:
  * Allow users to specify units for the `compute_totals` derivatives.
    Use an interface similar to `ExecComp`, aka `kwargs` mapping a variable name to a `dict` with `'units'`.
    In the meantime I can do the units conversion myself inside the functional interface class.
  * Allow users to specify indices for the `compute_totals` derivatives.
    Use an interface similar to `ExecComp`, aka `kwargs` mapping a variable name to a `dict` with `'indices'`.
    In the meantime I can keep track of indices inside the functional interface class.
  * Users can already ask for scaled derivatives via the `driver_scaling` argument.
    This should conflict with any `'units'` entries in the variables.

### Questions
* How do we properly handle the non-design variable inputs?
  Those can of course change during the life of a `Problem`.
    * Answer: should be able to create a callback function that takes as input any input variable.
      Same for outputs too, I think.
* What about options?
  Can those change?
  Should they be able to?
    * Answer: I think no.
      But really, as long as the callback "object" has a reference to the `Problem`, everything should just work.
* What type should the inputs and outputs be?
  Vectors, dicts, some type of hybrid?
  Maybe whatever an OpenMDAO `Vector` is?
  Or should we cook up a subclass of `numpy.ndarray`—might be the most convenient for the user.
    * Answer: Should be a vector.
* Should whatever code we come up with for need to interact with `Problem`, or just `Problem.model`?
  I think it needs to know about design variables and objectives and constraints, and those appear to be defined on `Problem.model`.
  On the other hand `Problem.run_model`, `Problem.compute_totals` and `Problem.compute_jacvec_product` exist on the `Problem` level.
    * Answer: Maybe someday it could just work with `Problem.model`, but for now it should work with `Problem`.
* Do we need to worry about PETSC vectors or MPI?
  Are the design variables, objectives or constraints ever distributed among MPI ranks is the critical question?
    * Answer: if we're allowing arbitrary inputs and outputs then of course some of them could be distributed.
      But nearly all optimizers require everything to be on one MPI rank (exception is `ParOpt`).
* Do we want to support the matrix-free stuff, aka `compute_jacvec_product`?
  Shouldn't be that hard...
* Didn't think about `indices`.
  Might need to support that too.
* What about constraint values?
  A user might not care about the constraint value themselves, but instead want them in terms of a violation aka `g = y - y_target`.
* What about scaling?
  I think developers of a new optimization algorithm would want scaled values.
  Should be an option.
* How should I properly check if the variables exist, etc.?
  `getval`/`setval` should just throw an error I guess.
* What should I do about sparse Jacobians?
  Need to understand what `compute_totals` gives me then.
* What to do about `flat_indices`?
  Need to develop a test.
