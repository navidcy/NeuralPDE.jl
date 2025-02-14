# 1D Wave Equation with Dirichlet boundary conditions

Let's solve this 1-dimensional wave equation:

```math
\begin{align*}
∂^2_t u(x, t) = c^2 ∂^2_x u(x, t) \quad & \textsf{for all } 0 < x < 1 \text{ and } t > 0 \, , \\
u(0, t) = u(1, t) = 0 \quad & \textsf{for all } t > 0 \, , \\
u(x, 0) = x (1-x)     \quad & \textsf{for all } 0 < x < 1 \, , \\
∂_t u(x, 0) = 0       \quad & \textsf{for all } 0 < x < 1 \, , \\
\end{align*}
```

with grid discretization `dx = 0.1` and physics-informed neural networks.

Further, the solution of this equation with the given boundary conditions is presented.

```julia
using NeuralPDE, Flux, ModelingToolkit, GalacticOptim, Optim, DiffEqFlux
import ModelingToolkit: Interval, infimum, supremum

@parameters t, x
@variables u(..)
Dxx = Differential(x)^2
Dtt = Differential(t)^2
Dt = Differential(t)

#2D PDE
C=1
eq  = Dtt(u(t,x)) ~ C^2*Dxx(u(t,x))

# Initial and boundary conditions
bcs = [u(t,0) ~ 0.,# for all t > 0
       u(t,1) ~ 0.,# for all t > 0
       u(0,x) ~ x*(1. - x), #for all 0 < x < 1
       Dt(u(0,x)) ~ 0. ] #for all  0 < x < 1]

# Space and time domains
domains = [t ∈ Interval(0.0,1.0),
           x ∈ Interval(0.0,1.0)]
# Discretization
dx = 0.1

# Neural network
chain = FastChain(FastDense(2,16,Flux.σ),FastDense(16,16,Flux.σ),FastDense(16,1))
initθ = Float64.(DiffEqFlux.initial_params(chain))
discretization = PhysicsInformedNN(chain, GridTraining(dx); init_params = initθ)

pde_system = PDESystem(eq,bcs,domains,[t,x],[u])
prob = discretize(pde_system,discretization)

cb = function (p,l)
    println("Current loss is: $l")
    return false
end

# optimizer
opt = Optim.BFGS()
res = GalacticOptim.solve(prob,opt; cb = cb, maxiters=1200)
phi = discretization.phi
```

We can plot the predicted solution of the PDE and compare it with the analytical solution in order to plot the relative error.

```julia
using Plots

ts,xs = [infimum(d.domain):dx:supremum(d.domain) for d in domains]
analytic_sol_func(t,x) =  sum([(8/(k^3*pi^3)) * sin(k*pi*x)*cos(C*k*pi*t) for k in 1:2:50000])

u_predict = reshape([first(phi([t,x],res.minimizer)) for t in ts for x in xs],(length(ts),length(xs)))
u_real = reshape([analytic_sol_func(t,x) for t in ts for x in xs], (length(ts),length(xs)))

diff_u = abs.(u_predict .- u_real)
p1 = plot(ts, xs, u_real, linetype=:contourf,title = "analytic");
p2 =plot(ts, xs, u_predict, linetype=:contourf,title = "predict");
p3 = plot(ts, xs, diff_u,linetype=:contourf,title = "error");
plot(p1,p2,p3)
```
![waveplot](https://user-images.githubusercontent.com/12683885/101984293-74a7a380-3c91-11eb-8e78-72a50d88e3f8.png)


## 1D Damped Wave Equation with Dirichlet boundary conditions

Now let's solve the 1-dimensional wave equation with damping. 

```math
\begin{aligned}
\frac{\partial^2 u(t,x)}{\partial x^2} = \frac{1}{c^2} \frac{\partial^2 u(t,x)}{\partial t^2} + v \frac{\partial u(t,x)}{\partial t} \\
u(t, 0) = u(t, L) = 0 \\
u(0, x) = x(1-x) \\
u_t(0, x) = 1 - 2x \\
\end{aligned}
```

with grid discretization `dx = 0.1` and physics-informed neural networks.

```julia
using NeuralPDE, Flux, ModelingToolkit, GalacticOptim, Optim, DiffEqFlux
using Plots
using Quadrature,Cubature
import ModelingToolkit: Interval, infimum, supremum

@parameters t, x
@variables u(..)
Dxx = Differential(x)^2
Dtt = Differential(t)^2
Dt = Differential(t)

# Constants
c = 2
v = 3
L = 1
@assert c > 0 && c < 2π / (L * v)

# 1D damped wave
eq = Dxx(u(t, x)) ~ 1 / c^2 * Dtt(u(t, x)) + v * Dt(u(t, x))

# Initial and boundary conditions
bcs = [u(t, 0) ~ 0.,# for all t > 0
       u(t, L) ~ 0.,# for all t > 0
       u(0, x) ~ x * (1. - x), # for all 0 < x < 1
       Dt(u(0, x)) ~ 1 - 2x # for all  0 < x < 1
       ]

# Space and time domains
domains = [t ∈ Interval(0.0, 1.0),
           x ∈ Interval(0.0, 1.0)]

# Neural network
af = Flux.tanh
chain = FastChain(FastDense(2, 16, af), FastDense(16, 16, af), FastDense(16, 1))
initθ = Float64.(DiffEqFlux.initial_params(chain))

dx = 0.1
strategy = GridTraining(dx)
discretization = PhysicsInformedNN(chain, strategy; init_params=initθ)

pde_system = PDESystem(eq, bcs, domains, [t, x], [u])
prob = discretize(pde_system, discretization)

pde_inner_loss_functions = prob.f.f.loss_function.pde_loss_function.pde_loss_functions.contents
inner_loss_functions = prob.f.f.loss_function.bcs_loss_function.bc_loss_functions.contents
bcs_inner_loss_functions = inner_loss_functions

cb = function (p, l)
    println("Current loss is: $l")
    println("pde_losses: ", map(l_ -> l_(p), pde_inner_loss_functions))
    println("bcs_losses: ", map(l_ -> l_(p), bcs_inner_loss_functions))
    return false
end

# Optimizer
opt = Optim.BFGS()
res = GalacticOptim.solve(prob, opt; cb=cb, maxiters=1400)
phi = discretization.phi

# Analysis
ts, xs = [infimum(d.domain):dx/5:supremum(d.domain) for d in domains]

μ_n(k) = (v * sqrt(4 * k^2 * π^2 - c^2 * L^2 * v^2)) / (2 * L)
b_n(k) = 2 / L * -((2 * π * L - π) * k * sin(π * L * k) + ((π^2 * L - π^2 * L^2) * k^2 + 2) * cos(π * L * k) - 2) / (π^3 * k^3) # vegas((x, ϕ) -> ϕ[1] = sin(k * π * x[1]) * f(x[1])).integral[1]
a_n(k) = 2 / (L * μ_n(k)) * -(L^2 * (((2 * π * L - π) * c * k * sin(π * k) + ((π^2 - π^2 * L) * c * k^2 + 2 * L * c) * cos(π * k) - 2 * L * c) * v^2 + (8 * π * L - 2 * π) * k * sin(π * k) + ((2 * π^2 - 4 * π^2 * L) * k^2 + 8 * L) * cos(π * k) - 8 * L)) / (2 * π^3 * k^3) # vegas((x, ϕ) -> ϕ[1] = (sin((k * π * x[1]) / L) * (g(x[1]) + (v^2 * c) / 2 * f(x[1])))).integral[1]

# Line plots
analytic_sol_func(t,x) = sum([sin((k * π * x) / L) * exp(-v^2 * c * t / 2) * (a_n(k) * sin(μ_n(k) * t) + b_n(k) * cos(μ_n(k) * t)) for k in 1:2:100]) # TODO replace 10 with 500

using Plots
using Printf

anim = @animate for t ∈ ts
    @info "Time $t..."
    sol =  [analytic_sol_func(t, x) for x in xs]
    sol_p =  [first(phi([t,x], res.minimizer)) for x in xs]
    plot(sol, label="analitic", ylims=[0, 0.1])
    title = @sprintf("t = %.3f", t)
    plot!(sol_p, label="predict", ylims=[0, 0.1],title=title)
end
gif(anim, "1Dwave_damped.gif", fps=200)

# Surface plot
u_predict = reshape([first(phi([t,x], res.minimizer)) for t in ts for x in xs], (length(ts), length(xs)))
u_real = reshape([analytic_sol_func(t, x) for t in ts for x in xs], (length(ts), length(xs)))

diff_u = abs.(u_predict .- u_real)
p1 = plot(ts, xs, u_real, linetype=:contourf, title="analytic");
p2 = plot(ts, xs, u_predict, linetype=:contourf, title="predict");
p3 = plot(ts, xs, diff_u, linetype=:contourf, title="error");
plot(p1,p2,p3)
```

We can see the results here:

![1Dwave_damped](https://user-images.githubusercontent.com/12683885/126818842-446d43b7-7099-4690-9a61-412c4eccfa70.png)

Plotted as a line one can see the analytical solution and the prediction here:

![1Dwave_damped_gif](https://user-images.githubusercontent.com/12683885/126818855-c9ae78ce-b976-4ed8-9ddb-f6d6d0fe6311.gif)



