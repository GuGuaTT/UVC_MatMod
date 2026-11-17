# An Updated Voce-Chaboche Material Model

This is the repository for an updated Voce-Chaboche (UVC) material model to model structural steel materials.

The UVC material model combines the nonlinear kinematic hardening rule of Chaboche with a refined formulation of the nonlinear isotropic hardening law of Voce.
This constitutive model is intended to be used for structural steels subjected to cyclic loading.
Notably, accounting for the discontinuous yielding phenomenon is shown to influence the buckling modes of steel wide-flange columns subjected to multi-axis cyclic loading.
Material models are provided for a variety of stress states, the implementations are shown to have similar, if not better, efficiency in terms of model convergence than the built-in nonlinear isotropic/kinematic model in Abaqus.

The theory and implementations of the material model are described in detail in the references below.

All four included Abaqus UMATs, including the uniaxial residual-stress variant, support optional time-dependent stiffness-proportional damping. The four damping constants are appended at the end of the input list; for `UVCuniaxial_IS.for`, they follow the six geometry and integration-point inputs. See the [Abaqus README](Abaqus/README.md) for the input order, Depvar requirements, examples, and figures. The OpenSees implementations are retained from the reference package.

## Changelog

- 17-Nov-2026 (by Tianyu Gu, [@GuGuaTT](https://github.com/GuGuaTT)):
  - Add optional, time-dependent stiffness-proportional damping to all four Abaqus UMATs. Append `beta_R1, t1, beta_R2, t2` at the end of the material constants and add the corresponding damping contributions to stress and the material Jacobian.
  - Store the constitutive stress separately from the damping stress. In `UVCuniaxial_IS.for`, preserve the Young residual-stress distribution and initialize the stored stress with the initial residual stress; place the damping inputs after the six geometry and integration-point inputs.
  - Document the input order, Depvar requirements, state-variable output, units, and damping examples in `Abaqus/README.md`.
  - Update the documentation for the Depvar compatibility patch: allow extra SDVs without damping, require an exact Depvar count with damping, and retain the existing cyclic input decks without changes.
- 16-Nov-2026 (by Lucas Kuhn, [@LucasKuhn67](https://github.com/LucasKuhn67)):
  - Abaqus and OpenSees multiaxial: the normal to the yield surface is now recomputed inside the return-mapping loop instead of being fixed at the trial state, following [Suchocki (2022)](https://doi.org/10.1007/s00707-021-03069-3). The consistent tangent is updated accordingly. Thanks to [Claudio M. Perez (@claudioperez)](https://github.com/claudioperez) for pointing this out.
  - Abaqus plane stress, uniaxial and uniaxial_IS, and OpenSees: the backstress update uses the plastic strain increment directly, fixing a rounding issue in the equivalent plastic strain.
  - OpenSees UVCuniaxial: constructor fix.
- 02-Jun-2023 (by Alex Hartloper, [@ahartloper](https://github.com/ahartloper); integrated into the lab repository on 16-Nov-2026):
  - Fix updating of sigma for UVCuniaxial_IS regression from [PR#14](https://github.com/ahartloper/UVC_MatMod/pull/14)
  - Make backstress arrays Real(8)
- 15-Sep-2022:
  - Add to the README.md regarding using the UMATS with Intel oneAPI and more recent version of Abaqus.
- 06-Jul-2022:
  - Improve local convergence of UVCuniaxial (see [PR#14](https://github.com/ahartloper/UVC_MatMod/pull/14))
  - Add additional Abaqus files for testing

## Installation

Instructions for installation are specific to the choice of simulation platform.
Detailed instructions are provided in the [Abaqus](Abaqus/README.md) and [OpenSees](OpenSees/README.md) directories. 

## Usage

Specific instructions for use are provided in each of the Abaqus and OpenSees directories.
The examples used to validate the implementations are provided in both the `Abaqus/testing` and `OpenSees/testing` directories.

Material parameters can be found on the OpenSees wiki at https://opensees.berkeley.edu/wiki/index.php/UVCuniaxial_(Updated_Voce-Chaboche)

### Abaqus and Fortran compiler compatibility

The reference README recorded compatibility reports on 15 September 2022 concerning allocatable backstress arrays with some Abaqus/Intel oneAPI configurations. These historical reports are not a compatibility verdict for all later releases. Use the compiler/toolchain supported by your installed Abaqus release and verify the selected user subroutine locally.

The workaround documented at that time was to replace the relevant allocatable backstress arrays with fixed-size arrays and remove their corresponding `ALLOCATE` statements. For example, for exactly two backstresses, the uniaxial declarations become:

```fortran
      REAL(8), DIMENSION(2, 2) :: chab_coef
      REAL(8), DIMENSION(2) :: alpha_k, alpha_k_init
```

If that workaround is required by the local configuration, remove only the matching allocation statements and keep the input backstress count consistent with the fixed array sizes. Other subroutines require corresponding changes to their own arrays. The supplied files retain allocatable arrays; no compiler-specific modifications were made in this documentation update.

## Contributing

Bug fixes can be raised by opening a new issue in the UVC_MatMod repository.

## Authors

Code written and maintained by Alex Hartloper (alexander.hartloper@epfl.ch).

## License

This project is licensed under the MIT License - see the `LICENSE` file for details.

## Acknowledgments

- Dimitrios Lignos and Albano de Castro e Sousa for their guidance and assistance in the formulation of the model theory and return mapping algorithms.

## References
[1] Hartloper, de Castro e Sousa, and Lignos (2021). "Constitutive Modeling of Structural Steels: Nonlinear Isotropic/Kinematic Hardening Material Model and Its Calibration", Journal of Structural Engineering, https://doi.org/10.1061/(ASCE)ST.1943-541X.0002964.

[2] Hartloper, de Castro e Sousa, and Lignos (2019). "A Nonlinear Isotropic/Kinematic Hardening Model for Materials with Discontinuous Yielding". Technical Report.

[3] Hartloper, de Castro e Sousa, and Lignos (2019). "Sensitivity of Simulated Steel Column Instabilities to Plasticity Model Assumptions", Proceedings of the 12th Canadian Conference on Earthquake Engineering, Quebec City, QC, Canada
