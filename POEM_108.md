POEM ID:  108
Title:  Functional interface for driver-level callbacks 
authors: @dingraha (Daniel Ingraham)  
Competing POEMs:   
Related POEMs:   
Associated implementation PR: 

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
input_vars = ["x1", "x2", "x3"]
output_vars = ["y1", "y2"]
x0, y0, f = prob.get_callback(form="f", input_vars=input_vars, output_vars=output_vars)
x0, dydx0, dfdx = prob.get_callback(form="dfdx", input_vars=input_vars, output_vars=output_vars)
x0, y0, dydx0, fdfdx = prob.get_callback(form="fdfdx", input_vars=input_vars, output_vars=output_vars)
```

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
