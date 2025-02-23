# Jump process
Simon Frost (@sdwfrost), 2020-04-27

## Introduction

This implementation defines the model as a combination of two jump processes, infection and recovery, simulated using the [Doob-Gillespie algorithm](https://en.wikipedia.org/wiki/Gillespie_algorithm).

## Libraries

```julia
using DifferentialEquations
using SimpleDiffEq
using Random
using DataFrames
using StatsPlots
using BenchmarkTools
```




## Transitions

For each process, we define the rate at which it occurs, and how the state variables change at each jump. Note that these are total rates, not *per capita*, and that the change in state variables occurs in-place.

```julia
function infection_rate(u,p,t)
    (S,I,R) = u
    (β,c,γ) = p
    N = S+I+R
    β*c*I/N*S
end
function infection!(integrator)
  integrator.u[1] -= 1
  integrator.u[2] += 1
end
infection_jump = ConstantRateJump(infection_rate,infection!);
```


```julia
function recovery_rate(u,p,t)
    (S,I,R) = u
    (β,c,γ) = p
    γ*I
end
function recovery!(integrator)
  integrator.u[2] -= 1
  integrator.u[3] += 1
end
recovery_jump = ConstantRateJump(recovery_rate,recovery!);
```





## Time domain

```julia
tmax = 40.0
tspan = (0.0,tmax);
```




For plotting, we can also define a separate time series.

```julia
δt = 0.1
t = 0:δt:tmax;
```




## Initial conditions

```julia
u0 = [990,10,0]; # S,I,R
```




## Parameter values

```julia
p = [0.05,10.0,0.25]; # β,c,γ
```




## Random number seed

We set a random number seed for reproducibility.

```julia
Random.seed!(1234);
```




## Running the model

Running this model involves:

- Setting up the problem as a `DiscreteProblem`;
- Adding the jumps and setting the algorithm using `JumpProblem`; and
- Running the model, specifying `SSAStepper()`

```julia
prob = DiscreteProblem(u0,tspan,p);
```


```julia
prob_jump = JumpProblem(prob,Direct(),infection_jump,recovery_jump);
```


```julia
sol_jump = solve(prob_jump,SSAStepper());
```




## Post-processing

In order to get output comparable across implementations, we output the model at a fixed set of times.

```julia
out_jump = sol_jump(t);
```




We can convert to a dataframe for convenience.

```julia
df_jump = DataFrame(out_jump')
df_jump[!,:t] = out_jump.t;
```




## Plotting

We can now plot the results.

```julia
@df df_jump plot(:t,
    [:x1 :x2 :x3],
    label=["S" "I" "R"],
    xlabel="Time",
    ylabel="Number")
```

![](figures/jump_process_14_1.png)



## Benchmarking

```julia
@benchmark solve(prob_jump,FunctionMap())
```

```
BenchmarkTools.Trial: 
  memory estimate:  12.94 KiB
  allocs estimate:  107
  --------------
  minimum time:     9.817 μs (0.00% GC)
  median time:      400.432 μs (0.00% GC)
  mean time:        447.690 μs (5.05% GC)
  maximum time:     7.199 ms (77.92% GC)
  --------------
  samples:          10000
  evals/sample:     1
```


