# Proposal for v2 Recipe spec

By example:

```YAML
context:
  name: blah
  version: "1.2"
  build_num: 3
  major_ver: "{{ version.split('.')[0] }}" 

# indicates that we are using version 2 of the recipe format
recipe_format: 2

package:
  name: {{ name|lower }}
  version: "{{ version }}"  # we have to have quotes here or conda build errors

source:
  - git_url: https://github.com/blib/blah
    git_rev: master
    patches:
      - 001_blah_to_blih.patch
      - sel(win): 002_blib_to_blob.patch

build:
  number: 0
  script: "{{ python }} -m pip install . --no-deps --ignore-installed -vv"
  
requirements:
  host:
    - pip
    - python >=3.5
    # this is a selector
    # it can be as many keys as needed
    # when parsing, only one key is allowed to eval to true, if more than one does 
    # an error is raised - the value in the dict with the true key is inserted for the 
    # element
    # this construct can appear anywhere in the recipe
    - sel(win or osx): rust =1
      sel(linux): rust >=1
  run:
    - python
    - "{{ pin_subpackage('libxgboost', exact=True) }}"
   
test:
  requirements:
    - nose
  imports:
    - blah

about:
  home: "https://palletsprojects.com/p/click/"
  license: "BSD"
  license_family: "BSD"
  license_file: "LICENSE.txt"
  summary: "Composable command line interface toolkit"

extra:
  recipe-maintainers:
    - FrankSinatra
    - ElvisPresley
```



# Discussion

# Next gen conda recipe spec

Cons of the current spec:

- It combines three programming languages: YAML, Jinja and the custom selector syntax
- Jinja and selectors makes it hard to parse using standard YAML parsers from Python
- Parsing is a multi-stage process in Python that iterates until all values are filled
- Version numbers are interpreted as floating point values ("0.14") in YAML

# more ideas (convo w/ MB)

 - test matrix for noarch for different platforms / versions
 - keep all steps independent
 - more formal spec so that we can validate recipes, get inputs directly
 - 

# Specification design process

(MBargull) If we want to come up with a "more sound" specification that is easier to parse (for machine as well as humans), avoids many complications of the current format and allows future extensions/workflow integrations, we should (on a higher level, IMHO) try to discern needed structures and processes as clearly as feasible.
This means:
- For the overall build process:
  - Identify and separate distinct phases, e.g. (just as an example, not meant to be correct/complete list),
    - Gather build context defined outside the actual package build specs (e.g., pinnings),
    - Parse build specs,
    - Gather build input/environment definitions,
    - Build package contents,
    - Post-process/packaging,
    - Testing,
    - ???
  - And then clearly define inputs and outputs for such phases
    - Helps with separation of concerns,
    - Allows definition of build workflows, e.g., dispatching actual building and testing to workers,
    - Removes "unexpected" implicit behavior.
- For the build specs (assuming a declarative tree-based format) (and its serialization format):
  - Define required input information for current and possible future needs.
    I.e., support everything that is currently supported (however, maybe drop existing concepts
    if they compete with base goals of the improved format) and keep future extensions in mind.
  - For clarity, start with a conceptual spec, i.e., don't jump right into YAML vs TOML vs ???:
    - E.g., for one of the `requirements` sections define something like
      ```
      NODE[X] := X | SELECT_NODE[X]
      SELECT_NODE[X] := [CONDITION, X]
      NODE[REQUIREMENTS] := LIST[REQUIREMENT_NODE]
      NODE[REQUIREMNT] := [PACKAGE_NAME, PACKAGE_VERSION, PACKAGE_BUILD, ...]
      ```
    - This allows a specification (somewhat) independent of the serialization format,
    - Avoids introducing format-specific constrictions early on, e.g.,
      if we go with the `"sel(cond)": entry` route, we assume to go with a format that makes it
      easy (or even allows) putting arbitrary content in mapping keys.
      (e.g., `{if: cond, then: entry}` would be a more flexible serialization alternative)
    - Pure tree-like definition allows to better identify data dependencies etc.
      And to decide on where what kind of dynamic content is necessary, e.g.,
      we don't have to stick with the more complicated `TEXT -> Jinja2 -> YAML -> [OBJECTS]`
      route but can do `TEXT -> YAML -> [OBJECTS] -> [Jinja2(obj) for obj in OBJECTS]`.
      Plus, we can see that `pin_compatible`/`pin_subpackage` don't have to be processed by the
      template engine at all (hint: they have static content!).
    - Lets us identify where we can artifically restrict the format to avoid complexity.
      (And plan for lifting restrictions for possible future features.)

This should also help us identify which legacy concepts/features we can remove or replace by
more general approaches (e.g., I'd like us to remove unnecessarily domain-specific things like
`noarch: python` (should be `{arch: noarch, package_type: python}` or the like), `CONDA_*` and
`PYTHON`/`{{ python }}` env/jinja2 vars etc.) and to make implicit behavior more explicit etc.


# Design parameters

- Declarative file or script file?
  - declarative is much simpler to parse, but less flexible. What dynamic features do we need? How do these mesh with declarative syntax?
    - Is a templating engine of any kind supported? 
    - If so, what kind, and how are limits on that going to be imposed?
  - script file can lead to impossible situations, like setup.py, and the inability to parse package metadata. This is largely avoided by embedding fully-rendered (static) metadata with built packages, but evaluating the metadata (rendering) can be slow for other things, like building graphs from just recipes.
  - Could provide both, like Jenkins: https://www.edureka.co/community/54705/difference-between-declarative-pipeline-scripted-pipeliine
- What dimensions of variability are necessary?
  - Version pins from a central configuration
  - Operating system
  - CPU architecture + optimization flags
- Producing related packages
  - If a build script produces a set of files that belongs in several split packages, how is that specified?
  - If one recipe relies on another, how can the version constraints be expressed?
  - How is the dependency tree for related packages computed? Can it be done in a non-iterative way?
- What is the model for metadata?
  - Conda-build 2.1 introduced split packages, with the implicit top-level recipe being treated as an output to maintain compatibility with the existing body of recipes.
  - Forcing everything to have explicit outputs would be nice
  - Top-level metadata could then be understood to only be for build, and would remain present as a snapshot for any of the output steps.
  - How does this model get affected by the various dimensions of variability?
  - This is related to the "Alternative to `bld.bat` / `build.sh`" section below.
- What workflows are supported? How does a recipe translate into:
  - packages
  - test environments (and how do these get re-run?)
  - CI build/test/upload reports
  - For recipes with interdependent split packages, are the dependency relationships fulfilled if only building one output? In other words, to "develop" an output of a recipe, are all of its requirements built and put in place so that the build of the ouptut can proceed? 
- Can mutex metapackage structures be simplified/streamlined?
  - These have way too many facets to understand and get right. The pattern is well-established now, and recipes could include some shorthand for this. 
- Specify environment variables to be set at runtime. This is a feature that must be implemented in conda and conda-build simultaneously.
- 

# Additional questions, ideas

- Support more of the match spec syntax. At least namespace (if it is ever implemented) will be mandatory and I can see the appeal of channel support as well (Wolf)
    - `channel:namespace:pkg 0.3.1 XXXXXX`
    - Think about supporting build numbers better. With the full match spec one can write `mypackage[build_number>5]` which seems impossible with the current conda_build spec
- `bld.bat` / `build.sh` ↦ unify script names to `build.bat` and `build.sh`
- Use a shell agnostic "scripting" language?

### Extending lists with lists

(WV) The new selector syntax may tempt to write code like this:

```
requirements:
  host:
    - python
    - "sel(win)":
      - curl
      - flask
      "sel(linux)":
      - wget
      - bash
```

It would require some code to make this work as expected. I wonder if we should support it or not.

MRB: No!

### Old School env vars?

At some point in time variants could be set with `CONDA_PY`, `CONDA_R`... environment vars. This apparently leads to conda-build having to check and regex the build scripts if these env vars are used. I think it's time to get rid of them and not include them with v2.

### Dave Hirschfeld: Optional Dependencies

It would be great if there was a better story for *optional* dependencies.

The special-casing of test requirements is inconsistent with the requirements for host/build/run. The fact that you can't independently install the test dependencies is a huge usability wart and suggests there should be a better way of supporting them.

I think that rather than special-casing test dependencies, that `test` should simply be another sub-category under `requirements` alongside host/build/run. Taking this idea further, a user could define *any* sub-category they wanted under `requirements` - e.g. `dev`, `docs`, etc...

In the case of `dask` they would define sub-categories `array`, `bag`, `dataframe`, `distributed`, `diagnostics`, `delayed`:
https://github.com/dask/dask/blob/master/setup.py#L10-L28

As it is, conda forces you to take an all or nothing approach - e.g.
https://github.com/conda-forge/dask-feedstock/blob/master/recipe/meta.yaml

Even if all you want to use is `dask.array` you're forced to install the dependencies for *everything*. The answer often given is to use outputs, and that's what we do ourselves - every package builds `pkg-test`, `pkg-docs` and `pkg-dev` metapackages along with `pkg` itself. Whilst this can be made to work, IMHO it's a poor substitute for proper support for optional dependencies. Ideally a recipe maintainer could specify whatever sub-category under `requriements` they wanted and conda would allow you to install that sub-category independently.

e.g. using pip syntax it would be

```
conda install dask[array,bag]
```

It has been argued that the pip syntax conflicts with existing conda syntax which is a fair argument, but there could be other ways to specify optional dependencies - e.g.

```
conda install dask --include-deps array --include-deps bag
```

`conda install dask` could then be used as an alias for `conda install dask --include-deps run`

This proposal requires a change to the yaml spec and also support from conda/mamba. The yaml changes could be made backwards compatible by allowing test requirements to be specified either under `test/requirements` or `requriements/test` with a suitable deprecation period.

MRB: we should not further complicate recipes with optional deps or arbitrary dep sections. the gain here is marginal and the costs are big 


### Output handling

Outputs can currently be two things:

- they can split a "super"-build up into multiple packages by selecting files specific to e.g. the python bindings of a C++ library
- they can act as standalone build with their own build scripts, define dependencies ...

These two functions and the (implicit?) super-build are confusing and not very intuitive. 
One suggestion is to have explicit "transient" builds / outputs but the rules for this are also not clear.

MRB: what do you mean the rules are not clear? The rules are

1. any transient output cannot be a run or test dependence
2. its artifact is always deleted at the end
3. it must always be pinned exactly in any host or build section

### Jinja functions

conda-build defines the following jinja functions:

- `load_setup_py_data` (used by 0 packages on cf)
- `load_setuptools` (used by 1 package on cf, conda_smithy)
- `load_npm` (used by 0 packages on cf)
- `load_file_regex` (used by 0 packages)
- `installed`
- `pin_compatible` (IMPORTANT)
- `pin_subpackage` (IMPORTANT)
- `compiler` (IMPORTANT)
- `cdt` (IMPORTANT)
- `resolved_packages`
- `time` -> this might not be great for reproducible builds? (MRB: we have to keep these, they are very important for builds that need to autoincrement the version)
- `datetime` -> this might not be great for reproducible builds? (MRB: we have to keep these, they are very important for builds that need to autoincrement the version)
- `environ` (IMPORTANT)


Maybe pin_compatible and pin_subpackage should _not_ be implemented as Jinja functions but rather as a preprocessing step in the build / host environment configuration? The syntax would stay the same but the `{{ & }}` brackets would be removed. The reasoning is that the output of those two functions depends on the solve and hashing process which is already a part of the processing that conda build does (vs. string manipulation and templating that Jinja does). A clean seperation could lead to code that becomes easier to digest.

The syntax to continue to use variables would change to `pin_subpackage({{ name }}, max_pin='x.x')`

MRB: The change of pin_subpackage and pin_compatible to non-jinja2 is pretty invasive and not needed. We can rearange the parsing to explicitly use the graph of deps and proceed in topological order to avoid the difficulties here.

WV: Yes, I agree we can keep the Jinja syntax, but internally the jinja might just modify the dependency to some other machine readable output, e..g 

`{{ pin_subpackage(name, max_pin='x.x') }} => PIN_SUBPACKAGE[somelib, max_pin=x.x, exact=False, ...]` which we can parse in a later step. This would remove the need of Jinja and solving to be intermangled.

MRB: FWIW, jinja2 gives you access to their AST. They break the text up in a way that has nothing to do with the YAML AFAICT.

MRB: My goal here is to make sure that all of these changes can be easily put into conda-build. Thus keeping them minimal is very important. The parsing and solving being intermingled is OK once we have a faster solver around.

WV: I think having clean seperations between the stages is much preferred though. Right now it's very difficult to follow the conda build code. The different stages that (theoretically) should exist are probably:

- Parse YAML
- Evaluate Jinja
- Sort outputs & recipes (this requires inspecting the dependencies ...)
- For each output / recipe, in order
    - create all variants
    - solve dependencies
    - Insert pin_subpackage & pin_compatible
    - build recipe & add output to local package cache


MRB: So all I am saying is that in step 3 above (sorting outputs and getting deps), we render with stub jinja2 functions for pin_subpackage and pin_compatible that just return the name. Then in stage 4, we use the information from the previous build. In other words, we can support the parsing + rendering + solving + building steps above with the jinja2 just fine.

WV: no, I don't think so because e.g. for sorting we _need_ the right package name inserted. Imagine an output using `{{ pin_compatible(name) }}` then we would (for sorting) have to evaluate the jinja etc. It would be much easier (implementation wise) to evaluate this to `mypackage PIN_COMPATIBLE[max_pin='x.x', ...]` and we still have a way to figure out the correct sorting order without going back to Jinja. Jinja should execute only once.

MRB: This is a minor implementation detail. Executing jinja2 once to build a special string that we then parse again is no different from having the second step of parsing the string again be done in jinja2 as well. At the end of the day, you have to turn `PIN_COMPATIBLE[max_pin='x.x', ...]` into an actual package requirement string.

WV: sure but you are missing that I was using `name` as a variable. We can throw away the jinja context and all that overhead after the first parsing step is done.

MRB: No I was not. The context is one extra dictionary that is around. Seems like a small thing. 

WV: it seems small, but it complicates the architecture. Clean seperation of the stages is very important to me. But it's an implementation detail, I agree with that and one is free to chose to do it differently. The key is that we want to engineer something that will be maintainable for a long time.

MRB: (edit:) implementing `PIN_COMPATIBLE[max_pin='x.x', ...]` in conda build as it is now is a huge change.

WV: ew don't want to support PIN_COMPATIBLE. The end user still writes `{{ pin_compatible( ... ) }}` but _internally_ boa / conda-build might choose to change it to an internal representation and do as it pleases.

MRB: I mean implementing the internal rep, not supporting it for users. Sorry! :D

WV: for reference, that's how I am planning to do it with boa, and then generate the conda-build MetaData class which I hopefully can feed to conda-build to create packages :)

### Alternative to `bld.bat` / `build.sh`

Chris Burr: This might be out of scope for this proposal but it would be nice for the scripts used for building to become more automatic. [Nix](https://nixos.org/nixpkgs/manual/#sec-stdenv-phases), [portage](https://devmanual.gentoo.org/ebuild-writing/functions/index.html) and many other package managers have a mechanism by which builds are split into "phases". Packages can modify how other stages behave, for example depending on `cmake` causes the configure stage to run something like `cmake "$source_dir" "${cmake_flags[@]}"`. The `cmake_flags` array can then be set to a sensible default. This makes it easier to write correct recipes without knowing the details of how to use every build system and also makes it easier to modify how all recipes are built. Both Nix and Portage ultimately implement this using bash scripting. I'll include a suggestion of how this might look but I'm not advocating for it to be exactly this way. I'm especially not sure how to map this over to Windows builds.

If the  `pip` package is defined like:

```yaml
build:
    export:  # Tells dependent recipes to change how they run some stages
        install: python -m pip install "${src_dir}"
```

and `cmake` is defined like:

```yaml
build:
    export_env:  # All exported environment changes are sourced before building any recipes
        declare -a cmake_flags
        cmake_flags+=("-DCMAKE_INSTALL_LIBDIR=lib")
        cmake_flags+=("-DCMAKE_INSTALL_PREFIX=${PREFIX}")
    export_phases:
        configure: |
            cmake \
                "${cmake_flags[@]}" \
                 "${src_dir}"
        build: cmake --build . -j${CPU_COUNT}
        install: cmake --build . --target install -j${CPU_COUNT}
```


* Pure python without using `pip`

```yaml
build:
    phases:
        build: python setup.py build
        install: python setup.py install
        check: |
            my_installed_comand --help
            my_installed_comand --version
        imports:
            - my_package
```

* Python using pip

```yaml
build:
    phases:  # Install phase is done automatically
        check: |
            my_installed_comand --help
            my_installed_comand --version
        imports:
            - my_package
```

* Package that uses CMake but that should have a non-standard feature (`SOME_FLAG`) enabled

```yaml
build:
    env: |
        cmake_flags+=("-DSOME_FLAG=ON")
    # No need to define any phases, everything is done automatically
```

### Providing environment changes

Currently if a package needs to provide an environment variable it has to be done by writing two bash+fish+csh+bat+... scripts that will be sourced during activation/deactivation. Many packages don't do this for all shells making them broken in some shells. It would be nicer to do this declaratively in the recipe metadata.


### Licensing

Currently some packages that provide static libraries or are header-only require downstream packages to include their licenses.
This is difficult to do correctly in practice therefore it might be useful to have a `license_exports` field.
See [this comment](https://github.com/conda-forge/conda-forge.github.io/issues/1052#issuecomment-640041318).

### Development environments

A common request is to be able to use the same recipe file local development or to provide nightly builds.
Tools such as [`conda-devenv`](https://github.com/ESSS/conda-devenv) have been made to ease this use case but this duplicates the maintanence work.

### Version constraints

Currently when using specifying a version constraint such as `numpy >=1.15` it conflicts with the global conda-forge pinning and causes the latest version to be used. It would be nice to be able to minimise versions in `host` to maximise compatibility later on. Or at least to be able to combine the global pins with the recipe ones, i.e. in this case where the global pin is `1.14` on x86 and `1.16` on ARM/POWER, `1.15` should be chosen on x86 and `1.16` should be chosen on alternative architectures.

# Proposal v2

This one combines some of the great ideas from v1, but tries to make more minimal changes to the 
spec.

rules:

 - always generate valid yaml
 - never embed functional information in comments
 - three-step parsing: 1) context constants, 2) generate context vars, 3) the rest of the recipe
 - no jinja2 control-flow elements
 - versions always have quotes
 - no implicit outputs (either every output is in outputs or not)
 - test requirements are always listed in `test.requirements`
 - each output can only have sections that are the same as the top-level ones (except the context block)
 - top-level sections are replaced by output level ones on a per key basis down one level (i.e., dict.update in python)
 - selectors are implemented as dictionaries whose keys are `eval`ed in python - see the comments below on how they are 
   interpreted

```YAML
context:
  name: blah
  version: "1.2"
  build_num: 3
  major_ver: "{{ version.split('.')[0] }}" 

# indicates that we are using version 2 of the recipe format
recipe_format: 2

package:
  name: {{ name|lower }}
  version: "{{ version }}"  # we have to have quotes here or conda build errors

source:
  - git_url: https://github.com/blib/blah
    git_rev: master
    patches:
      - 001_blah_to_blih.patch
      - sel(win): 002_blib_to_blob.patch

build:
  number: 0
  script: "{{ python }} -m pip install . --no-deps --ignore-installed -vv"
  
requirements:
  host:
    - pip
    - python >=3.5
    # this is a selector
    # it can be as many keys as needed
    # when parsing, only one key is allowed to eval to true, if more than one does 
    # an error is raised - the value in the dict with the true key is inserted for the 
    # element
    # this construct can appear anywhere in the recipe
    - sel(win or osx): rust =1
      sel(linux): rust >=1
  run:
    - python
    - "{{ pin_subpackage('libxgboost', exact=True) }}"
   
test:
  requirements:
    - nose
  imports:
    - blah

about:
  home: "https://palletsprojects.com/p/click/"
  license: "BSD"
  license_family: "BSD"
  license_file: "LICENSE.txt"
  summary: "Composable command line interface toolkit"

extra:
  recipe-maintainers:
    - FrankSinatra
    - ElvisPresley
```

Here is one with outputs:

```YAML
context:
  name: blah
  version: "1.2"
  build_num: 3
  major_ver: "{{ version.split('.')[0] }}" 

package:
  version: "{{ version }}"

build:
  number: 0
  binary_relocation: False
  script: "{{ python }} -m pip install . --no-deps --ignore-installed -vv"

requirements:
  host:
   - pip
   - python >=3.5
   - "sel(win or osx)": rust =1
     "sel(linux)": rust >=1
  run:
   - python
   - "{{ pin_subpackage('libxgboost', exact=True) }}"

test:
  requirements:
   - nose
  imports:
   - blah

# all of these outputs use the same requirements as above
# along with the test requirements, but not imports
outputs
  - package:
      name: out1
      # has the same version as above
    build:
      binary_relocation: True
      script: out1_build.sh
    test:
      imports:
        - blah.out2
      commands:
        - echo "this format is nice!"
  
  - package:
      name: ou2
      version: "{{ version }}.1"
    build:
      script: out2_build.sh
    test:
      imports:
        - blah.out2

about:
  home: https://palletsprojects.com/p/click
  license: BSD
  license_family: BSD
  license_file: LICENSE.txt
  summary: Composable command line interface toolkit

extra:
  recipe-maintainers:
    - "FrankSinatra"
    - "ElvisPresley"
```

# Comments on v1 (from MRB)

This proposal is a very logical reaction to complexity. complexity arises for many reasons and i (mrb) don't fully understand where the complexity in the orignal conda recipe format came from. IIUIC selectors predate jinja2 in the recipes so part of it might simply be the accumulation of features over time. I worry that another part of the complexity is from actual needs that we have not fully enumerated or understood. Thus it seems like we should either try to make minimal changes or we need to take a census of sorts of recipes, both simple and complicated, to ensure that we can support everything that is needed. 

1. the version being a string is something we can test in conda-build itself and error otherwise. this in and of itself is not a motivation for moving to TOML or the like. 

2. the selectors here are implemented too narrowly given how our recipes use them in practice.

3. I really like the very structured context block. It solves issues around enforcing good jinja2 usage and clearly stating constants at the top. 

4. We will still need jinja2 munging of stuff in the context block. i think a three stage parsing process makes sense. 1) get the context, 2) eval munged context with jinja2, 3) render the rest of the recipe with the context. 

5. this spec doesn't address a bunch of other mildy confusing things (eg test commands w/ test scripts too, requires versus requirements in the test section, the py selector value, how build scripts are specified for outputs, ...)

6. i'd like to see all jinja2 control flow commands deprecated. 

# Proposal v1

Switch to a more restricted markup language (TOML) and get rid of some of the custom features.

Rules:

- Never produce invalid TOML
- Never use comments as functional elements
- 2-step parsing: first extract "context" variables, then render the rest with Jinja


```TOML
# initialize a variable context (has to come first)
[context]
name = "click"
version = "7.0"

# Now you can use the context variables
[package]
name = "{{ name|lower }}"
version = "{{ version }}"

[source]
url = "https://pypi.io/packages/source/{{ name[0] }}/{{ name }}/{{ name }}-{{ version }}.tar.gz"
sha256 = "5b94b49521f6456670fdb30cd82a4eca9412788a93fa6dd6df72c94d5a8ff2d7"

[build]
number = 0
script = "{{ PYTHON }} -m pip install . --no-deps --ignore-installed -vv "

[requirements]
host = [
  "pip",
  "python >=3.5",
  "rust =1 [win or osx]",
  "rust >=1 [linux]"
]

run = [
  "python",
  "{{ pin_subpackage('libxgboost', exact=True) }}"
]

[test]
imports = [
  "click"
]

[about]
home = "https://palletsprojects.com/p/click/"
license = "BSD"
license_family = "BSD"
license_file = "LICENSE.txt"
summary = "Composable command line interface toolkit"
doc_url = "https://palletsprojects.com/p/click/"
description = """
This is a long description
of the package that is described by this 
meta.yaml file and it spans multiple lines.
"""

[extra]
recipe-maintainers = [
  "FrankSinatra", "ElvisPresley"
]
```

TODO
----

Clarify how outputs are handled -- implicit first level output, but if `[outputs]` key specified, no implicit outputs?!
    - MCS: https://docs.conda.io/projects/conda-build/en/latest/resources/define-metadata.html#implicit-metapackages and https://docs.conda.io/projects/conda-build/en/latest/resources/define-metadata.html#outputs-section