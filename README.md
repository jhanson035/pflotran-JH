# PFLOTRAN CK1 source

Source snapshot from the local PFLOTRAN CK1 working tree, including local source modifications.

Upstream: https://bitbucket.org/pflotran/pflotran
Base revision: `ab16c8e9c577cd55e4bfdff152dea245d5d8c2b6`

## Contents

The `src/` directory retains its original structure. License and copyright notices are included in `LICENSE` and `COPYRIGHT`.

## Build

A compatible PETSc installation and Fortran build toolchain are required. Set `PETSC_DIR` and `PETSC_ARCH`, then run:

```sh
cd src/pflotran
make pflotran
```

This source-only snapshot omits the top-level databases, regression tests, tools, and other supporting directories. Features and test targets that depend on them require those files from the full PFLOTRAN checkout. This snapshot has not been build-tested separately.
