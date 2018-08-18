Consider a simple 1-d advection equation:
$$
\dfrac{\partial \phi}{\partial t} + c \dfrac{\partial \phi}{\partial x} = 0
$$
we could have the followings by applying Lax-Wendroff scheme:
$$
\phi(x_{j},t_{n+1})=\phi_j^{n+1}=\phi_j^{n}-\dfrac{c\Delta t}{2\Delta x}(\phi_{j+1}^{n}-\phi_{j-1}^{n})+\dfrac{c^2\Delta t^2}{2\Delta x^2}(\phi_{j+1}^{n}-2\phi_{j}^{n}+\phi_{j-1}^{n})
$$
$t_n$ represents the n-th time step and $x_j$ means the j-th grid. The truncation error of this method is $O(\Delta t^2+\Delta x^2)$, meaning this method has the second-order accuracy.

After some algebra, the above numerical solution could be expressed as a two-step formula in which each individual step is centered in space and time.

The first step:
$$
\dfrac{\phi_{j+\frac{1}{2}}^{n+\frac{1}{2}}-\dfrac{1}{2}(\phi_{j+1}^n+\phi_j^n)}{\dfrac{1}{2}\Delta t}=-c(\dfrac{\phi_{j+1}^n-\phi_{j}^n}{\Delta x})
$$
and the second step:
$$
\dfrac{\phi_{j-\frac{1}{2}}^{n+\frac{1}{2}}-\dfrac{1}{2}(\phi_{j}^n+\phi_{j-1}^n)}{\dfrac{1}{2}\Delta t}=-c(\dfrac{\phi_{j}^n-\phi_{j-1}^n}{\Delta x})
$$
In the second step, $\phi_j^{n+1}$ is computed from:
$$
\dfrac{\phi_{j}^{n+1}-\phi_{j}^{n}}{\Delta t}=-c(\dfrac{\phi_{j+\frac{1}{2}}^{n+\frac{1}{2}}-\phi_{j-\frac{1}{2}}^{n+\frac{1}{2}}}{\Delta x})
$$
The Lax-Wendroff scheme could be recovered by eliminating $\phi^{n+1/2}$ and $\phi^{n-1/2}$ in the second step formula using first step formula. Also, if the wind speed $c$ is a function of the spatial coordinate , $c$ is replaced by $c_{j+1/2}$ and $c_{j-1/2}$ ,$c_j$in the first and second step respectively.

Lax-Wendroff scheme is a stable scheme and we do not need to apply artificial diffusion in time or space. 

As for shallow water equations here, we actually only need to extend the 1-d case into 2-d case. The Lax-Wendroff scheme for 2-d simple advection equation is:
$$
\phi_{i+\frac{1}{2},j}^{n+\frac{1}{2}} = \dfrac{1}{2}(\phi_{i+1,j}^n+\phi_{i,j}^n)-\dfrac{1}{2}\dfrac{c\Delta t}{\Delta x}(\phi_{i+1,j}^n-\phi_{i,j}^n) \\
\phi_{i,j+\frac{1}{2}}^{n+\frac{1}{2}} = \dfrac{1}{2}(\phi_{i,j+1}^n+\phi_{i,j}^n)-\dfrac{1}{2}\dfrac{c\Delta t}{\Delta y}(\phi_{i,j+1}^n-\phi_{i,j}^n) \\
\phi_{i,j}^{n+1}=\phi_{i,j}^n-c \Delta t \left[  \dfrac{\phi_{i+\frac{1}{2},j}^{n+\frac{1}{2}}-\phi_{i-\frac{1}{2},j}^{n+\frac{1}{2}}}{\Delta x} +   \dfrac{\phi_{i,j+\frac{1}{2}}^{n+\frac{1}{2}}-\phi_{i,j-\frac{1}{2}}^{n+\frac{1}{2}}}{\Delta y}  \right]
$$
If we have source/sink term, we just need to add it on the right hand of the third equation:
$$
\phi_{i,j}^{n+1}=\phi_{i,j}^n-c \Delta t \left[  \dfrac{\phi_{i+\frac{1}{2},j}^{n+\frac{1}{2}}-\phi_{i-\frac{1}{2},j}^{n+\frac{1}{2}}}{\Delta x} +   \dfrac{\phi_{i,j+\frac{1}{2}}^{n+\frac{1}{2}}-\phi_{i,j-\frac{1}{2}}^{n+\frac{1}{2}}}{\Delta y}  \right] + S_{i,j}^{n+\frac{1}{2}} \Delta t
$$
