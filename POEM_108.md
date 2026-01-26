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

[^function]: Likely not functional in the mathematical sense.

## Description
We want something like this:

```python
prob = om.Problem()
# Do the usual stuff to define the Problem.
x = get_desvar_vector_dict(prob)
f = get_objective_callback(prob)
objective = f(x)
g = get_constraints_callback(prob)
constraints = g(x)
dfdx = get_objective_gradient_callback(prob)
objective_x = dfdx(x)
dgdx = get_constraint_jacobian_callback(prob)
consraints_x = dgdx(x)
```

* Questions
  * How do we properly handle the non-design variable inputs?
    Those can of course change during the life of a `Problem`.
  * What about options?
    Can those change during a `Problem`?
  * What type should the inputs and outputs be?
    Vectors, dicts, some type of hybrid?
    Maybe whatever an OpenMDAO `Vector` is?
    Or should we cook up a subclass of `numpy.ndarray`?
  * Should whatever code we come up with for need to interact with `Problem`, or just `Problem.model`?
    I think it needs to know about design variables and objectives and constraints, and those appear to be defined on `Problem.model`.
    On the other hand `Problem.run_model`, `Problem.compute_totals` and `Problem.compute_jacvec_product` exist on the `Problem` level.
  * Do we need to worry about PETSC vectors or MPI?
    Are the design variables, objectives or constraints ever distributed among MPI ranks is the critical question?
