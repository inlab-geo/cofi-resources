# Continuous integration for Python Projects on GitHub

These instructions assume that a python package has been created that:
- can successfully be installed using `pip install .` from its main directory
- is available on Github
- contains unit tests using the `pytest` framework

We will be using [https://github.com/inlab-geo/pyfm2d](https://github.com/inlab-geo/pyfm2d`) 
as an example and create two GitHub actions that is one to test the package and one to deploy it on pypi.

Github actions are `yml`/`yaml` files that need to be placed in `.github/workflows` for github 
to recognise them and it appears that they need to be created on the main branch.

The instructions provided here are based on the infomration available here
- [https://packaging.python.org/en/latest/guides/publishing-package-distribution-releases-using-github-actions-ci-cd-workflows/
](https://packaging.python.org/en/latest/guides/publishing-package-distribution-releases-using-github-actions-ci-cd-workflows/)
- [https://packaging.python.org/en/latest/tutorials/packaging-projects/](https://packaging.python.org/en/latest/tutorials/packaging-projects/)

## Unit testing
The complete github action `.github/workflows/test.yaml` that runs the tests is given below and it splits up into several parts and uses the following predefined github action.
- [https://github.com/actions/checkout](https://github.com/actions/checkout)
- [https://github.com/fortran-lang/setup-fortran](https://github.com/fortran-lang/setup-fortran)
- [https://github.com/astral-sh/setup-uv](https://github.com/astral-sh/setup-uv)

```
name: Run Tests

# Configure this workflow so that it runs on pushes to the main branch and pull requests
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

# Define the tests to be run the environments to be tested are defined by the matrix option.
# specifcially the python versions 3.9 to 3.13 are used to test the package on ubuntu 
# and mac os x

jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        python-version: ["3.9", "3.10", "3.11", "3.12", "3.13"]
        os: [ubuntu-latest, macos-latest]

    steps:
	   # this pulls the package from the repository
      - uses: actions/checkout@v4

	  # this setups the gnu fortran compiler
      - name: Setup GNU Fortran
        uses: fortran-lang/setup-fortran@v1

      # uv is used as the python package manager
      - name: Install uv and set the python version
        uses: astral-sh/setup-uv@v5
        with:
          python-version: ${{ matrix.python-version }}

	  # uv is used to install the project
      - name: Install the project
        run: uv sync --all-extras --dev
  
  	  # this simply runs the test in the test directory of the project
      - name: Run tests
        # test_fmmin2d needs to be run from the test directory
        # for access to the input files
        run: |
          cd test 
          uv run pytest 
```

The package `pyfm2d` contains  fortran code hat needs to be compiled for a package where this is not the 
case there is no need to setup a fortran compiler and this step  can be omitted.


## Publishing on PyPI

An account is needed on pypi to publish the package from the github repository using 
a github action. Care must be taken with the release number defined in `pyproject.toml` 
and it needs to match the tag that is being used to automatically trigger a release. 
That is when a tag `like v1.2.3` is pushed, the workflow is triggered.

In the approach outlined here the initial upload of the package to pypi is different from
subsequent updates from github directly. Thus for the initial upload it is recommended 
to use a version number that is not the initial release in `pyproject.toml` something 
like `0.0.1dev`

```{warning}
Version numbers on pypi are **unique** once a version has been uploaded it can not be replaced or deleted. Hence the suggestion to use `0.0.1dev` for the initial uploade and/or use [https://packaging.python.org/en/latest/guides/using-testpypi/](https://packaging.python.org/en/latest/guides/using-testpypi/) when experimenting.
```

The alternative is to follow the instructions here [https://packaging.python.org/en/latest/guides/publishing-package-distribution-releases-using-github-actions-ci-cd-workflows/](https://packaging.python.org/en/latest/guides/publishing-package-distribution-releases-using-github-actions-ci-cd-workflows/) but this requires more editing of github actions than the approach outlined below.

### Initial upload to PyPI

Install the latest version PyPA build

`python3 -m pip install --upgrade build`

Now run this command from the same directory where pyproject.toml is located:

`python3 -m build`

This will create distribution files for the platform that is being used 
in the `dist` subdirectory. These are the initial upload to PyPI.

To securely upload your project, you’ll need a PyPI API token. Create one 
at [https://pypi.org/manage/account/#api-tokens](https://pypi.org/manage/account/#api-tokens), 
setting the “Scope” to “Entire account”. Don’t close the page until you have 
copied and saved the token — you won’t see that  token again.

Now that you are registered, you can use twine to upload the distribution packages. 
You’ll need to install Twine:

`python3 -m pip install --upgrade twine`

and then upload the package

`python3 -m twine upload --repository pypi dist/*`

When prompted for an API token use the token value, including the pypi- prefix 
created on the pypi website.

This creates a project on PyPI which we now can publish to using github actions. The 
access token that has been created can now be deleted as it is no longer needed.

### Publishing from github to PyPI

The recommendation is to use trusted publishing which can be setup once the 
project exists on pypi or can be setup directly for a new project on pypi following the 
steps here [https://packaging.python.org/en/latest/guides/publishing-package-distribution-releases-using-github-actions-ci-cd-workflows/](https://packaging.python.org/en/latest/guides/publishing-package-distribution-releases-using-github-actions-ci-cd-workflows/)

Here we assume the project already exists on pypi. The next step is to select 
the project on its pypi page and then select publishing and add a new publisher 
for github by completing the fields. It is recommended to constrain the created 
trusted publisher to the github action 'deployment' environment.

This then allows to use the following github action in `.github/workflows/build_and_deploy.yaml
` to deploy the project to pypi whenever there is a new tag created and the version number in 
`pyproject.toml` updated.

```
Name: Build and Deploy

# workflow is run either manually triggerd, if there is a tag being added to a commit or in
# caase of a pull request.
on:
  workflow_dispatch:
  push:
    tags:
      - 'v*.*.*'
  pull_request:
    branches:
      - main

jobs:

# define the operating systems for which we will build binaries
  build-wheel:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: ["ubuntu-latest", "macos-latest"]
      fail-fast: false
    
    steps:
      - uses: actions/checkout@v4

      - name: Setup GNU Fortran
        uses: fortran-lang/setup-fortran@v1
        id: setup-fortran
        with:
          compiler: gcc
          version: 11
    
      - name: Build wheels
        uses: pypa/cibuildwheel@v2.22
        with:
          output-dir: wheelhouse
        env:
          CIBW_SKIP: pp* cp36* cp37* cp38* *-win32 *-manylinux_i686 *-musllinux_*
          MACOSX_DEPLOYMENT_TARGET: 14

      - name: Upload wheels as artifact
        uses: actions/upload-artifact@v4
        with:
          name: ${{ matrix.os }}-wheelhouse
          path: wheelhouse

# make the source available and build from it if there is no binary for the platform
# seeking to install the package.
  build-sdist:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repo
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Install Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.10'

      - name: Set up environment
        run: |
          python -m pip install scikit-build-core build

      - name: Build and check sdist
        run: |
          python -m build --sdist

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          path: dist/*.tar.gz
          name: sourcedist

# deploy the pacakge  to pypi
  deploy:
    runs-on: ubuntu-latest
    needs: [build-wheel, build-sdist]
    if: ${{ startsWith(github.ref, 'refs/tags/v') || github.event_name == 'workflow_dispatch' }} # <--- Here!
    environment: deployment
    permissions:
      id-token: write
    steps:
      - name: Download artifacts
        uses: actions/download-artifact@v4
        with:
          merge-multiple: true
          path: dist

      - name: Publish package to PyPI
        uses: pypa/gh-action-pypi-publish@v1.12.3
```

### Triggering an update on pypi

PyPI works by releases taht is we have to increase the version number for a release to happen. For pyfm2d this means updating the version number in the `pyproject.yml` file.

```
[build-system]
requires = ["scikit-build-core[rich,pyproject]"]
build-backend = "scikit_build_core.build"

[project]
name = "pyfm2d"
version = "0.1.4"
requires-python = ">=3.9"
dependencies = [
    "cartopy>=0.2.3",
    "matplotlib>=3.9.4",
    "numpy>=2.0.2",
    "pyrr>=0.10.3",
    "scipy>=1.13.1",
    "tqdm>=4.67.1",
]

[project.optional-dependencies]
notebooks = [
    "jupyterlab>=4.3.4",
]

[dependency-groups]
dev = [
    "pytest>=8.3.4",
]
```


### Github actions

The two github actions we defined in the two yaml files are aviailabe for the repository on its action subpage.
[https://github.com/inlab-geo/pyfm2d/actions](https://github.com/inlab-geo/pyfm2d/actions)
