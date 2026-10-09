# Fortran User Materials for the UVC Material Model

This readme file outlines the usage of the UVC material model in Abaqus (previously RESSForLab).

## Installation

### Prerequisites

- Compatible version of Abaqus linked with the Fortran compiler
- Local copy of the user subroutine file of interest (i.e., the files included in this repository)

Note: all testing for the UMATs have been done with Abaqus v6.14, the Intel Fortran compiler (ifort), and Visual Studio Community 2013.
The user is responsible for correctly linking Abaqus with a compatible Fortran compiler.

### Obtaining the UMAT files 

The user material subroutine (UMAT) files simply need to be downloaded and made available to the Abaqus model of interest.
All the relevant steps are described in the Usage section.
The UMAT files can be downloaded individually from github, or the entire repository can be cloned using git using the command line.

The git command to clone the repository is:
```
git clone https://github.com/ahartloper/UVC_MatMod.git
```
Alternatively ssh can be used instead of https when cloning the repository.
Use of the UMAT files are described in the next section.

## Usage

This section outlines the usage with Abaqus CAE.
Usage directly with an input file is similar and the reader is instructed to \*USER MATERIAL of the Abaqus Keyword Reference Guide for more information.
Usage is divided into parts depending on the UMAT file used, generally the steps are: 1) collect the model parameters, 2) set-up the user material, 3) optionally specify the transverse shear stiffness and the hourglass control, 4) make the UMAT file available to Abaqus.
The number of parameters to be specified depends on whether you are using the uniaxial version, or the plane-stress/multiaxial version.
Enhanced hourglass control is recommended for reduced integration elements, and the transverse shear stiffness needs to be calculated and specified if shell elements are used.
Details of how to calculate and specify the transverse shear stiffness for shell elements are summarized under the UVCplanestress section.

See [here](https://github.com/ahartloper/UVC_MatMod/blob/master/README.md#abaqus-2019-and-later-with-intel-oneapi) for using more recent version of Abaqus and the Intel oneAPI Fortran compiler.

### UVCuniaxial

This UMAT (UVCuniaxial.for) should be used in elements with uniaxial stress states (e.g., beam elements).
An alternative uniaxial version, `UVCuniaxial_IS.for`, is also provided.
This alternative UMAT implements the Young (1972) residual stress distribution for I-shaped sections.

#### Parameters
- E: Young's modulus of the material
- sy0: Initial yield stress
- QInf: Maximum increase in yield stress due to cyclic hardening at model saturation 
- b: Saturation rate of QInf
- DInf: Maximum initial reduction in yield surface for materials with discontinuous yielding (set to 0.0 to neglect this effect)
- a: Saturation rate of DInf, this parameter should be non-zero
- C1: Increase in stress due to kinematic hardening at saturation for backstress 1
- gamma1: Rate term for backstress 1
- [C2 gamma2 C3 gamma3 ... CN gammaN] Additional backstress parameters, if CK is specified then the corresponding gammaK must also be specified. 
N is the total number of backstress in the model.

#### (Optional) Additional parameters required when using UVCuniaxial_IS
The following values need to be specified in addition to the material parameters above only when the UVCuniaxial_IS.for UMAT is used:
- dcl: flange-to-flange centerline depth of the section (d - tf)
- bf: flange width
- tf: flange thickness
- tw: web thickness
- npt_flange: number of integration points on each flange (Abaqus default is 5)
- npt_web: number of integration points along the web (Abaqus default is 5)

#### Set-up the user defined material

- Create a user defined material, specify that the type is "Mechanical":
```
General > User Material
```
- Set the number of parameters to be 8 + 2(N - 1).
For example, there should be 10 parameters for 2 backstresses.
When using UVCuniaxial_IS, the number of parameters will be 8 + (2N - 1) + 6.
- Enter the parameters following the order listed in this Parameters section
- Assign space for the internal variables:
```
General > Depvar
```
Set the "Number of solution-dependent state variables" to 1 + N.
For example, for two backstresses the value should be 3.

Now the user material can be assigned to sections as any material included with Abaqus.

#### Transverse shear stiffness

The transverse shear stiffness needs to be assigned to beam elements when UMATs are used for these sections.
- Calculate the transverse shear stiffness value according to
```
K13 = K23 = k * (A * G)
```
where A is the total area of the beam section, G = E / (2(1 + nu)) is the shear modulus and k is a correction factor.
For I shaped sections the correction factor is k = 0.44. Factors for other cross-sections are listed in Section 29.3.3 of the Abaqus Analysis User's Guide.
- Set the transverse shear stiffness as follows:
```
Sections > Edit Section > Stiffness tab > check Specify Transverse Shear
```
- Set the values in the boxes according to the calculated values

#### Make the UMAT available to Abaqus

Now that the user material and model is properly set-up, we just need to make the UMAT file available to Abaqus.
- Specify the UMAT file
```
Double-click on the associated Job > General tab > Locate the User subroutine file
```

#### Order of the state variables

The equivalent plastic strain (PEEQ), plastic strain, and backstress vector can be output using `SDV, Solution dependent state variables` field output.
The number of SDVs depends on the model considered (e.g., UVCuniaxial or UVCplanestress) and the number of backstresses.
For UVCuniaxial:
- SDV(1) = equivalent plastic strain
- SDV(1+k) = value of backstress component k

### UVCplanestress

This UMAT (UVCplanestress.for) should be used in elements with plane-stress stress states (e.g., shell elements).

#### Parameters
- E: Young's modulus of the material
- nu: Poisson's ratio
- sy0: Initial yield stress
- QInf: Maximum increase in yield stress due to cyclic hardening at model saturation 
- b: Saturation rate of QInf
- DInf: Maximum initial reduction in yield surface for materials with discontinuous yielding (set to 0.0 to neglect this effect)
- a: Saturation rate of DInf, this parameter should be non-zero
- C1: Increase in stress due to kinematic hardening at saturation for backstress 1
- gamma1: Rate term for backstress 1
- [C2 gamma2 C3 gamma3 ... CN gammaN] Optional, additional backstress parameters, if CK is specified then the corresponding gammaK must also be specified.
N is the total number of backstress in the model.

#### Set-up the user defined material

- Create a user defined material, specify that the type is "Mechanical":
```
General > User Material
```
- Set the number of parameters to be 9 + 2(N - 1).
For example, there should be 11 parameters for 2 backstresses.
- Enter the parameters following the order listed in this Parameters section
- Assign space for the internal variables:
```
General > Depvar
```
Set the "Number of solution-dependent state variables" to 4 + 3N.
For example, for two backstresses the value should be 10.

Now the user material can be assigned to sections as any material included with Abaqus.
For shell elements, the transverse shear stiffness needs to be assigned to the section as described in the next section.

#### Transverse shear stiffness

The transverse shear stiffness needs to be assigned to shell elements when UMATs are used for these sections.
- Calculate the transverse shear stiffness value according to
```
K11 = K22 = 5/6 G * t
```
where G = E / (2(1 + nu)) is the shear modulus and t is the shell section thickness, and
``` 
K12 = 0
```
- Set the transverse shear stiffness as follows:
```
Sections > Edit Section > Advanced tab > check Specify Values under Transverse Shear Stiffnesses
```
- Set the values in the boxes according to the calculated values

#### Specify enhanced hourglass control

We recommend that enhanced hourglass control is specified for reduced integration elements (e.g., S4R, C3D8R).
If you are not using reduced integration elements this section can be ignored.

- Specify enhanced hourglass control for all elements that will be assigned the UMAT
```
Go to Element Type > choose Hourglass Control > check Enhanced
```

#### Make the UMAT available to Abaqus

Follow the directions under Make the UMAT available to Abaqus section under UVCuniaxial.


#### Order of the state variables

The equivalent plastic strain (PEEQ), plastic strain, and backstress vector can be output using `SDV, Solution dependent state variables` field output.
The number of SDVs depends on the model considered (e.g., UVCuniaxial or UVCplanestress) and the number of backstresses.
For UVCplanestress:
- SDV(1) = equivalent plastic strain
- SDV(2) to SDV(4) = plastic strains PE11, PE22, PE12
- SDV(5+[3k-3])/(6+[3k-3])/(7+[3k-3]) = backstress components in directions 11/22/12 for component k


### UVCmultiaxial

This UMAT (UVCmultiaxial.for) should be used with solid elements.

#### Parameters

Follow the instructions under the Parameters section under UVCplanestress.

#### Set-up the user defined material

- Create a user defined material, specify that the type is "Mechanical":
```
General > User Material
```
- Set the number of parameters to be 9 + 2(N - 1).
For example, there should be 11 parameters for 2 backstresses.
- Enter the parameters following the order listed in this Parameters section
- Assign space for the internal variables:
```
General > Depvar
```
Set the "Number of solution-dependent state variables" to 7 + 6N.
For example, for two backstresses the value should be 19.

Now the user material can be assigned to sections as any material included with Abaqus.


#### Specify enhanced hourglass control
 
Follow the instructions under the Specify enhanced hourglass control section under UVCplanestress.


#### Make the UMAT available to Abaqus

Follow the directions under Make the UMAT available to Abaqus section under UVCuniaxial.


#### Order of the state variables

The equivalent plastic strain (PEEQ), plastic strain, and backstress vector can be output using `SDV, Solution dependent state variables` field output.
The number of SDVs depends on the model considered and the number of backstresses.
For UVCmultiaxial:
- SDV(1) = equivalent plastic strain
- SDV(2) to SDV(7) = plastic strains PE11, PE22, PE33, PE12, PE13, PE23
- SDV(8+[3k-3])/(9+[3k-3])/(10+[3k-3])/(11+[3k-3])/(12+[3k-3])/(13+[3k-3]) = backstress components in directions 11/22/33/12/13/23 for component k


# 	The **U**pdate **V**oce-**C**haboche material model with **D**amping effect (UVC-D)
This repository contains the ABAQUS subroutines for the UVC-D material model, and this README file outlines the usage of the UVC and UVC-D material models. Validation of the UVC-D model is provided in the accompanying PDF included in this repository.

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

- $ \sigma_m $: Updated stress passed to ABAQUS for equilibrium check
- $ \sigma $: Constitutive stress from UVC model
- $ \beta_R $: Rayleigh damping coefficient
- $ D^{el} $: Elastic stiffness matrix
- $ \dot{\varepsilon} $: Strain rate of the increment

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

![Fig1](Figs\Fig1.jpg)

Figure 1 Validation of the UVC-D subroutine (for more details please refer to the validation slides)

## 3. Usage instructions on the UVC/UVC-D subroutine

The detailed usage instructions on the UVC subroutine can be found in [the official UVC Material Model GitHub repository](https://github.com/ahartloper/UVC_MatMod/tree/master/Abaqus). In brief, the user just need to assign the UVC material parameters in the User Material section in a given order as shown in **Figure 2**, and then assign the **"Number of solution-dependent state variables"** in the Depvar section (as shown in **Figure 2**) based on the number of backstresses, N:

- \(1 + N\) for **uniaxial** cases  

- \(4 + 3N\) for **plane stress** cases  

- \(7 + 6N\) for general **multiaxial** cases 

For example, if two backstresses are used (N = 2), the value for the multiaxial case is 19 as shown in **Figure 2**.


![Fig1](Figs\Fig2.jpg)

Figure 2 Input parameters of the UVC subroutine

To use the UVC-D subroutine, in the ABAQUS User Material section, aside from the original UVC material parameters (see details in [the UVC Material Model GitHub repository](https://github.com/ahartloper/UVC_MatMod/tree/master/Abaqus) and **Figure 2**), one has to additionally input four parameters, $\beta_R^1$, $t^1$, $\beta_R^2$, $t^2$,  as shown in the left panel of **Figure 3**. The two $\beta_R$ values denote respectively,

- $\beta_R^1$: Stiffness-proportional damping coefficient, activated between simulation time $t^1$ and $t^2$
- $\beta_R^2$: Stiffness-proportional damping coefficient, activated after simulation time $t^2$

This design aligns with the typical modeling workflow in **ABAQUS** dynamic analyses.  For example, **Figure 4** illustrates the time history of the story drift ratio of an MRF under ground motion. A static gravity loading step with a duration of 1 second (the time here is unreal) usually precedes the dynamic loading phase; therefore,  $t^1$ can be set to 1 (as shown in **Figure 3**) to ensure that damping is inactive during the gravity step. After the ground motion ends at $t^2$, a large value of $\beta^2_R$ can be specified to rapidly suppress free vibrations for residual drift, as demonstrated in **Figures 3** and **4**.


![Fig1](Figs\Fig3.jpg)

Figure 3 Input parameters of the UVC-D subroutine

<img src="Figs\Fig4.jpg" alt="Slide0" width="550"/>

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
