# 	The **U**pdate **V**oce-**C**haboche material model with **D**amping effect (UVC-D)
This readme file outlines the usage of UVC and UVC-D material model subroutines in ABAQUS. The validations of the UVC-D model are provided in the accompanying PDF in this repo.

## 1. General information on different versions of UVC subroutine

Currently, there are two versions of the UVC subroutine:  
- The original UVC subroutine developed by Hartloper et al. (2021), and  
- A modified version that additionally incorporates Rayleigh stiffness-proportional damping effect for ABAQUS dynamic analysis, referred to as **UVC-D**.

The original UVC model captures combined isotropic and kinematic hardening effects and is well-suited for simulating cyclic plasticity in metals. Users are encouraged to consult [the official UVC Material Model GitHub repository](https://github.com/ahartloper/UVC_MatMod/tree/master/Abaqus) for subroutine files and detailed usage instructions; only a brief overview is provided here. The model parameters include:

- $E$: Young's modulus of the material.
- $\nu$: Poisson's ratio.
- $\sigma_{y,0}$: Initial yield stress.
- $Q_{\infty}$: Maximum increase in yield stress due to cyclic hardening at model saturation.
- $b$: Saturation rate of $Q_{\infty}$.
- $D_{\infty}$: Maximum initial reduction in yield surface for materials with discontinuous yielding (set to 0.0 to neglect this effect).
- $a$: Saturation rate of $D_{\infty}$, this parameter should be non-zero.
- $C_1$: Increase in stress due to kinematic hardening at saturation for the first backstress.
- $\gamma_1$: Rate term for the first backstress.
- [$C_2$ $\gamma_2$ $C_3$ $\gamma_3$ ... $C_M$ $\gamma_M$\] : Optional, additional backstress parameters $-$ if $C_k$ is specified then the corresponding $\gamma_k$ must also be specified. $M$ is the total number of backstress in the model.

The parameters $Q_{\infty}$, $b$, $D_{\infty}$, and $a$ govern isotropic hardening, while $C_k$ and $\gamma_k$ control kinematic hardening. For typical parameter values, please refer to Hartloper et al. (2021). This documentation primarily focuses on the the concept, implementation, and usage of the UVC-D subroutine, which are presented in detail in the sections below.

> **Note:** Both the UVC and UVC-D subroutines are **unit system dependent** — the versions provided are specifically configured for the **mm, Newton** unit system in ABAQUS. If you intend to use a different unit system, you **must adjust** the tolerance variable `TOL` in the subroutine accordingly.

The following table summarizes the recommended `TOL` values for different unit systems and subroutine types:

| Subroutine Type     | mm, Newton | cm, Newton | dm, Newton | m, Newton |
|---------------------|------------|------------|------------|-----------|
| Multiaxial          | 1D-10      | 1D-8       | 1D-6       | 1D-4      |
| Plane stress        | 1D-8       | 1D-4       | 1D0        | 1D4       |
| Uniaxial            | 1D-10      | 1D-6       | 1D-2       | 1D2       |

## 2. The concept and implementation of the UVC-D subroutine

In structural dynamic analysis, **Rayleigh damping** is introduced as a linear combination of the mass and stiffness matrices, expressed as:

$$
C = \alpha M + \beta K
$$

- $C$ – damping matrix
- $M$ – mass matrix
- $K$ – stiffness matrix
- $\alpha$ – mass-proportional damping coefficient
- $\beta$ – stiffness-proportional damping coefficient

However, in ABAQUS, the Rayleigh damping implementation depends on the material type:

1. **ABAQUS built-in material**:  
   ✅ Supports both **mass-** and **stiffness-proportional** damping.  

2. **User-defined material (e.g., UVC)**:  
   ⚠️ Only supports **mass-proportional** damping by default.

Therefore, to incorporate the $\beta$ term (stiffness-proportional damping) when using the UVC material model, the UVC subroutine must be manually modified. In this work, we extended the original subroutine to create **UVC-D**, which accepts an additional input parameter, $\beta_R$, to allow the inclusion of stiffness-proportional damping effects. This modification follows the guidelines provided in the [ABAQUS documentation]([ABAQUS Analysis User's Manual (v6.6)](https://classes.engineering.wustl.edu/2009/spring/mase5513/abaqus/docs/v6.6/books/usb/default.htm?startat=pt05ch20s01abm43.html)) (ABAQUS, 2019). Specifically, after the element stress is computed from the constitutive model, an additional damping-related stress term is introduced to account for the Rayleigh damping contribution:

$$
\sigma_m = \sigma + \beta_R D^{el} \dot{\varepsilon}
$$

- $\sigma_m$: Updated stress passed to ABAQUS for equilibrium check
- $\sigma$: Constitutive stress from UVC model
- $\beta_R$: Rayleigh damping coefficient
- $D^{el}$: Elastic stiffness matrix
- $\dot{\varepsilon}$: Strain rate of the increment

````fortran
! Modifications to the UVC subroutine in multiaxial case
strain_rate = dstran / dtime
stress = stress + beta_r * MATMUL(elasticity_matrix, strain_rate)
````

Since the stress used in the equilibrium check, $\sigma_m$, includes the additional damping term, the corresponding **tangent modulus** should also be based on this modified stress. Therefore, when force equilibrium is not satisfied, the strain increment update will be performed using the tangent of $\sigma_m$, rather than that of the original constitutive stress $\sigma$. This approach can lead to improved convergence performance.

$$
D_m^{\mathrm{tg}} = \frac{\Delta\sigma_m}{\Delta\varepsilon}
= \frac{\Delta\sigma}{\Delta\varepsilon} + \beta_R D^{el} \frac{\Delta\varepsilon}{\Delta\varepsilon\, \Delta t}
= D^{\mathrm{tg}} + \frac{\beta_R}{\Delta t} D^{el}
$$

- $D_m^{\mathrm{tg}}$ – modified tangent modulus matrix based on the modified stress–strain relationship  
- $D^{\mathrm{tg}}$ – original tangent modulus matrix based on the constitutive stress–strain relationship  

```fortran
! Modifications to the UVC subroutine in multiaxial case
ddsdde = ddsdde + beta_r * elasticity_matrix / dtime
```

These two modifications comprise the **UVC-D** subroutine. To validate its implementation, a series of tests were done in ABAQUS. For example, **Figure 1a** shows an SDOF system subjected to sine wave ground motion. **Figure 1b** compares the point mass responses obtained using the built-in elastic material model with built-in Rayleigh damping $\beta = 0.005$, and the UVC-D elastic material model with $\beta_R = 0.005$. The close agreement between the two responses confirms the correctness of the UVC-D subroutine implementation. For additional details on the validation process, please refer to the validation slides.

<img src="Figs\Fig1.jpg" alt="Slide1" width="900"/>

Figure 1 Validation of the UVC-D subroutine (for more details please refer to the validation slides)

## 3. Usage instructions on the UVC/UVC-D subroutine

The detailed usage instructions on the UVC subroutine can be found in [the official UVC Material Model GitHub repository](https://github.com/ahartloper/UVC_MatMod/tree/master/Abaqus). In brief, the user just need to assign the UVC material parameters in the User Material section in a given order as shown in **Figure 2**, and then assign the **"Number of solution-dependent state variables"** in the Depvar section (as shown in **Figure 2**) based on the number of backstresses, N:

- \(1 + N\) for **uniaxial** cases  

- \(4 + 3N\) for **plane stress** cases  

- \(7 + 6N\) for general **multiaxial** cases 

For example, if two backstresses are used (N = 2), the value for the multiaxial case is 19 as shown in **Figure 2**.


<img src="Figs\Fig2.jpg" alt="Slide2" width="900"/>

Figure 2 Input parameters of the UVC subroutine

To use the UVC-D subroutine, in the ABAQUS User Material section, aside from the original UVC material parameters (see details in [the UVC Material Model GitHub repository](https://github.com/ahartloper/UVC_MatMod/tree/master/Abaqus) and **Figure 2**), one has to additionally input four parameters, $\beta_R^1$, $t^1$, $\beta_R^2$, $t^2$,  as shown in the left panel of **Figure 3**. The two $\beta_R$ values denote respectively,

- $\beta_R^1$: Stiffness-proportional damping coefficient, activated between simulation time $t^1$ and $t^2$
- $\beta_R^2$: Stiffness-proportional damping coefficient, activated after simulation time $t^2$

This design aligns with the typical modeling workflow in **ABAQUS** dynamic analyses.  For example, **Figure 4** illustrates the time history of the story drift ratio of an MRF under ground motion. A static gravity loading step with a duration of 1 second (the time here is unreal) usually precedes the dynamic loading phase; therefore,  $t^1$ can be set to 1 (as shown in **Figure 3**) to ensure that damping is inactive during the gravity step. After the ground motion ends at $t^2$, a large value of $\beta^2_R$ can be specified to rapidly suppress free vibrations for residual drift, as demonstrated in **Figures 3** and **4**.


<img src="Figs\Fig3.jpg" alt="Slide3" width="900"/>

Figure 3 Input parameters of the UVC-D subroutine

<img src="Figs\Fig4.jpg" alt="Slide4" width="550"/>

Figure 4 Example of MRF response under dynamic ground motion for demonstration of UVC-D subroutine usage

When using UVC-D subroutine, it is also important to assign mass-proportional damping coefficient, $\alpha$, as usual in the Damping section, but assign $\beta$ as 0 as it has already been considered as $\beta_R$ in the subroutine as shown in **Figure 3**. 

In addition, similar to the UVC subroutine, the user must specify the **"Number of solution-dependent state variables"** in the Depvar section (as shown in **Figure 3**) based on the number of backstresses, N. The required number of state variables is:

- \(2 + N\) for **uniaxial** cases  
- \(7 + 3N\) for **plane stress** cases  
- \(13 + 6N\) for general **multiaxial** cases  

For example, if two backstresses are used (N = 2), the corresponding values should be 4, 13, and 25. It should be noted these values differ from those specified in the [official UVC Material Model GitHub repository](https://github.com/ahartloper/UVC_MatMod/tree/master/Abaqus) for UVC subroutine. 

The order of the **SDV (Solution-Dependent State Variables)** field output in the ABAQUS results file is generally consistent with what is reported in the [official UVC Material Model GitHub repository](https://github.com/ahartloper/UVC_MatMod/tree/master/Abaqus) for the original UVC subroutine. The only difference is that when the UVC-D subroutine is used, the SDV output includes additional entries: the original stress **without** the damping term, $\sigma$, is appended to the end of the SDV list. These additional entries correspond to  1 SDV for **uniaxial** cases, 3 SDVs for **plane stress** cases, and 6 SDVs for **multiaxial** cases.

## References

Hartloper, Alexander R., Albano de Castro e Sousa, and Dimitrios G. Lignos. *"Constitutive modeling of structural steels: nonlinear isotropic/kinematic hardening material model and its calibration."* *Journal of Structural Engineering* 147.4 (2021): 04021031.

ABAQUS. *ABAQUS 2019 Documentation.* Providence, RI, USA: Dassault Systèmes, 2019.
