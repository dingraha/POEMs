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
J2 = dfdx.create_jacobian_matrix()
dfdx(x, J=J2)

# Create combined callback function that will return both outputs and Jacobian.
fdfdx = prob.get_callback(form="fdfdx", input_vars=input_vars, output_vars=output_vars)
x = fdfdx.create_input_vector()
y, J = fdfdx(x)

# Again, can set things in-place.
y3 = f.create_output_vector()
J3 = fdfdx.create_jacobian_matrix()
fdfdx(x, y=y3, J=J3)
```

### TODOs
* High priority items
  * [ ] Handle linear constraints properly
    * derivatives of linear constraints aren't returned by default by the `_TotalJacInfo.compute_totals` method.
      Should fix that.
  * [x] ignore scaling
  * [x] do indices myself in the functional interface class
  * [x] Advice from Bret: make sure to cache the `_TotalJacInfo` object in the functional interface wrapper class (expensive to create).
  * [x] Ignore the possibility of sparse total Jacobians
    * Claude and I think that `Problem.compute_totals` will always return plain old `numpy.ndarray`s, so ignoring that for now.
  * [ ] Check that I'm handling indices properly.
    * [ ] `flat_indices = False` case
    * [ ] `flat_indices = True` case
    * [ ] Check behavior of design variables/objectives/constraints with user-declared indices (e.g passing an `indices` or `flat_indices` argument to `Problem.add_design_var`).
      * At the moment the user can specify metadata for input and output vars, but if they don't pass anything then I use what the `_TotalJacInfo` object comes up with for two things: 
          * which variables they want
          * the indices for each variable
        But I need to make sure that things don't conflict: what if a user sets indices for e.g. a design variable, then wants different indices for the functional interface?
* [ ] Add a warning when a user specifies units explaining that it's not fully supported.
* [x] Provide a way for users to get back part of the input or output vectors?
  * Done via `_FunctionalCallback.get_input_val(var_name)` and `_FunctionalCallback.get_output_val(var_name)`

* Low priority items
  * Add units support
    * [x] Easy for input and output vectors, since we can just use `Problem.get_val` and `Problem.set_val` with the `units` argument.
    * [ ] ATM would need to manually scale the Jacobian we get from `Problem.compute_totals`.
    * [ ] Best way to do that would be to add units support to `_TotalJacInfo`.
    * [ ] Should add a warning when a user specifies units explaining that it's not fully supported.
  * [ ] Allow constraints to be expressed as a violation (instead of raw value).
    Maybe I should provide a way for the user to specify what they'd prefer?
    Maybe `driver_scaling` should mean the user wants a violation, otherwise raw value.
  * [ ] Add `driver_scaling` support
    * This should conflict with including units, I think.
  * [ ] Determine how to best deal with sparse total Jacobians
    * Seems that `_TotalJacInfo` doesn't do anything ATM.
  * [ ] `compute_jacvec_product` support

#### `Problem.compute_totals` wishlist/concerns/questions
* How should constraint values be expressed?
  A user might not care about the constraint value themselves, but instead want them in terms of a violation aka `g = y - y_target`.
  Should be an option IMHO, perhaps set via the eventual `driver_scaling` flag.

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
