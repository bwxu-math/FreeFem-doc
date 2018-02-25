# Pure Convection : The Rotating Hill

**Summary :** _Here we will present two methods for upwinding for the simplest
convection problem. We will learn about Characteristics-Galerkin
and Discontinuous-Galerkin Finite Element Methods._

Let $\Omega$ be the unit disk centered at 0; consider the rotation vector field
$$ {u} = [u1,u2], \qquad u_1 = y,\quad u_2 = -x.$$
Pure convection by $u$ is

$$
    \p_t c  + {u}.\nabla c  = 0~\hbox{~in~}~~\Omega\times(0,T)
    ~~~ c (t=0) =  c ^0~\hbox{~in~}~~\Omega.
$$

The exact solution $c(x_t,t)$ at time $t$ en point $x_t$ is given by

$$
 c(x_t,t)=c^0(x,0)
$$

where $x_t$ is the particle path in the flow starting at point $x$ at time $0$.
So $x_t$ are solutions of

** CAREFUL MISTAKE ON THE FOLLOWING FORMULA $\codered$**

\[
    \dot{x_t} = u(x_t),~~~\vec  , \quad\ x_{t=0} =x , \quad\mbox{where}\quad  \dot{x_t} = \frac{\d ( t \mapsto x_t)}{\d t}
\]

The ODE are reversible and we want the solution at point $x$ at time $t$ ( not at point $x_t$)
the initial point is $x_{-t}$, and we have
$$
 c(x,t)=c^0(x_{-t},0)
$$
The game consists in solving the equation until $T=2\pi$, that is for
a full revolution and to compare the final solution with the initial one;
they should be equal.

**Solution by a Characteristics-Galerkin Method**

In FreeFem++ there is an operator called `convect([u1,u2],dt,c)` which compute  
$ c\circ X$ with $X$ is the convect field defined by
$ X(x)= x_{dt}$ and where  $x_\tau$ is particule path in the steady state velocity field $\bm{u}=[u1,u2]$
starting at point $x$ at time $\tau=0$, so $x_\tau$ is solution of the following ODE:
%exactly the equation for ${\vec \chi}:=(X(\tau),Y(\tau))$
\[
    \dot{x}_\tau = u(x_\tau),~~~\vec x_{\tau=0}=x.
\]

When $\bm{u}$ is piecewise constant; this is possible because
$x_\tau$ is then a polygonal curve which can be computed exactly and the solution exists always when
$u$ is divergence free; convect returns  $c(x_{df})=C\circ X$.

```freefem
// file convects.edp

border C(t=0, 2*pi) { x=cos(t);  y=sin(t); };
mesh Th = buildmesh(C(100));
fespace Uh(Th,P1);
Uh cold, c = exp(-10*((x-0.3)^2 +(y-0.3)^2));

real dt = 0.17,t=0;
Uh u1 = y, u2 = -x;
for (int m=0; m<2*pi/dt ; m++) {
    t += dt;     cold=c;
    c=convect([u1,u2],-dt,cold);
    plot(c,cmm=" t="+t + ", min=" + c[].min + ", max=" +  c[].max);
}
```

!!! info
	3D plots can be done by adding the qualifyer "dim=3" to the plot instruction.


The method is very powerful but has two limitations: a/ it is not conservative, b/ it may diverge
in rare cases when $|u|$ is too small due to quadrature error.

**Solution by Discontinuous-Galerkin FEM**

Discontinuous Galerkin methods take advantage of the discontinuities of $c$ at the edges to build
upwinding.  There are may formulations possible. We shall implement here the so-called dual-$P_1^{DC}$
formulation (see Ern\cite{ern}$\codered$):

\[
    \int_\Omega(\frac{c^{n+1}-c^n}{\delta t} +u\cdot\n c)w
    +\int_E(\alpha|n\cdot u|-\frac 12 n\cdot u)[c]w
    =\int_{E_\Gamma^-}|n\cdot u| cw~~~\forall w
\]

where $E$ is the set of inner edges and $E_\Gamma^-$ is the set of boundary edges where $u\cdot n<0$
(in our case there is no such edges). Finally $[c]$ is the jump of $c$ across an edge with the convention
that $c^+$ refers to the value on the right of the oriented edge.

```freefem
// file convects.edp
...
fespace Vh(Th,P1dc);

Vh w, ccold, v1 = y, v2 = -x, cc = exp(-10*((x-0.3)^2 +(y-0.3)^2));
real u, al=0.5;  dt = 0.05;

macro n() (N.x*v1+N.y*v2) // Macro without parameter
problem  Adual(cc,w) =
int2d(Th)((cc/dt+(v1*dx(cc)+v2*dy(cc)))*w)
  + intalledges(Th)((1-nTonEdge)*w*(al*abs(n)-n/2)*jump(cc))
//  - int1d(Th,C)((n<0)*abs(n)*cc*w) // Unused because cc=0 on $\p\Omega^-$
  - int2d(Th)(ccold*w/dt);

for ( t=0; t< 2*pi ; t+=dt)
{
  ccold=cc; Adual;
  plot(cc,fill=1,cmm="t="+t + ", min=" + cc[].min + ", max=" +  cc[].max);
};
real [int] viso=[-0.2,-0.1,0,0.1,0.2,0.3,0.4,0.5,0.6,0.7,0.8,0.9,1,1.1];
plot(c,wait=1,fill=1,ps="convectCG.eps",viso=viso);
plot(c,wait=1,fill=1,ps="convectDG.eps",viso=viso);
```

Notice the new keywords, `intalledges` to integrate on all edges of all triangles

\begin{equation}
\mathtt{intalledges}(\mathtt{Th}) \equiv \sum_{T\in\mathtt{Th}}\int_{\p T }
\end{equation}

(so all internal edges are see two times), nTonEdge which is one
if the triangle has a boundary edge and two otherwise, `jump` to implement $[c]$.
Results of both methods are shown on Fig. 3.6 with identical levels for the level line;
this is done with the plot-modifier viso.

Notice also the macro where the parameter $u$ is not used (but
the syntax needs one) and which ends with a //; it simply replaces
the name `n` by `(N.x*v1+N.y*v2)`. As easily guessed `N.x,N.y` is the normal to the edge.

|Fig. 3.6:The rotated hill after one revolution with Characteristics-Galerkin|and with Discontinuous $P_1$ Galerkin FEM.|
|:----|:----|
|![Convect CG](images/convectCG.svg)|![convectDG](images/convectDG.svg)|

Now if you think that DG is too slow try this :

```freefem
// The same DG very much faster
varf aadual(cc,w) = int2d(Th)((cc/dt+(v1*dx(cc)+v2*dy(cc)))*w)
        + intalledges(Th)((1-nTonEdge)*w*(al*abs(n)-n/2)*jump(cc));
varf bbdual(ccold,w) =  - int2d(Th)(ccold*w/dt);
matrix  AA= aadual(Vh,Vh);
matrix BB = bbdual(Vh,Vh);
set (AA,init=t,solver=sparsesolver);
Vh rhs=0;
for ( t=0; t< 2*pi ; t+=dt)
{
  ccold=cc;
  rhs[] = BB* ccold[];
  cc[] = AA^-1*rhs[];
  plot(cc,fill=0,cmm="t="+t + ", min=" + cc[].min + ", max=" +  cc[].max);
};
```

Notice the new keyword `set` to specify a solver in this framework; the modifier `init` is used to tell the solver that the matrix has not changed (init=true), and the name parameter are the same that in problem definition (see. \ref{def problem})

**Finite Volume Methods** can also be handled with FreeFem++ but it requires programming.
For instance the $P_0-P_1$ Finite Volume Method of Dervieux et al associates to each $P_0$
function $c^1$ a $P_0$ function $c^0$ with constant value around each vertex $q^i$ equal to $c^1(q^i)$
on the cell $\sigma_i$ made by all the medians of all triangles having $q^i$ as vertex.
Then upwinding is done by taking left or right values at the median:

\[
    \int_{\sigma_i}\frac 1{\delta t}({c^1}^{n+1}-{c^1}^n) + \int_{\p\sigma_i}u\cdot n c^-=0
    ~~~\forall i
\]

It can be programmed as :

```freefem
load "mat_dervieux"; // External module in C++ must be loaded
border a(t=0, 2*pi){ x = cos(t); y = sin(t);  }
mesh th = buildmesh(a(100));
fespace Vh(th,P1);

Vh vh,vold,u1 = y, u2 = -x;
Vh v = exp(-10*((x-0.3)^2 +(y-0.3)^2)), vWall=0, rhs =0;

real dt = 0.025;
// qf1pTlump means mass lumping is used
problem  FVM(v,vh) = int2d(th,qft=qf1pTlump)(v*vh/dt)
                  - int2d(th,qft=qf1pTlump)(vold*vh/dt)
      + int1d(th,a)(((u1*N.x+u2*N.y)<0)*(u1*N.x+u2*N.y)*vWall*vh)
+ rhs[] ;

matrix A;
MatUpWind0(A,th,vold,[u1,u2]);

for ( int t=0; t< 2*pi ; t+=dt){
  vold=v;
  rhs[] = A * vold[] ; FVM;
  plot(v,wait=0);
};
```

the `mass lumping` parameter forces a quadrature formula with Gauss points at the vertices
so as to make the mass matrix diagonal; the linear system solved by a conjugate gradient method for instance will then converge in one or two iterations.

The right hand side `rhs` is computed by an external C++ function `MatUpWind0(...)`
which is programmed as :

```freefem
// Computes matrix a on a triangle for the Dervieux FVM
int   fvmP1P0(double q[3][2], // the 3 vertices of a triangle T
              double u[2],   // convection velocity on T
              double c[3],   // the P1 function on T
              double a[3][3],// output matrix
              double where[3] ) // where>0 means we're on the boundary
{
  for(int i=0;i<3;i++) for(int j=0;j<3;j++) a[i][j]=0;

    for(int i=0;i<3;i++){
        int ip = (i+1)%3, ipp =(ip+1)%3;
        double unL =-((q[ip][1]+q[i][1]-2*q[ipp][1])*u[0]
                    -(q[ip][0]+q[i][0]-2*q[ipp][0])*u[1])/6;
        if(unL>0) { a[i][i] += unL; a[ip][i]-=unL;}
            else{ a[i][ip] += unL; a[ip][ip]-=unL;}
        if(where[i]&&where[ip]){        // this is a boundary edge
            unL=((q[ip][1]-q[i][1])*u[0] -(q[ip][0]-q[i][0])*u[1])/2;
            if(unL>0) { a[i][i]+=unL; a[ip][ip]+=unL;}
        }
    }
  return 1;
}
```

It must be inserted into a larger .cpp file, shown in Appendix A,
which is the load module linked to FreeFem++.