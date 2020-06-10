# repodata.json

The repodata file contains all information of a repository.

The metadata for the repository is under the `info` key.

The packages in the repository are in the `packages` key. New-style `.conda` packages are under the `conda.packages` key.
The `packages` and `conda.packages` key point to a dictionary which has the file name as key.

A given package can appear under `packages` (as `.tar.bz2` package) as well as under the `conda.packages` key. Generally, the `.conda` package will be preferred, but a setting can override that behavior.

### Subdirs:

Currently, the potential subdirs are:

- `linux-{32, 64}`
- `osx-{32, 64}`
- `win-{32, 64}`
- `linux-aarch64`
- `linux-ppc64le`
- `noarch`


### Keys in repodata.json

|      key       |            type            |                                    meaning                                     |
+----------------+----------------------------+--------------------------------------------------------------------------------+
| info           | [dict]                     | metadata, usually one key (subdir)                                             |
| packages       | [dict[string => packages]] | dict with all old-style packages (tar.bz2), dictionary key is package filename |
| conda.packages | [dict[string => packages]] | dict with all new-style packages (conda)                                       |
| removed        | [list[string]]             | list of package file names that are removed                                    |


### Package data

The metadata for each package consists of the following fields:

|      key       |        type        |                              meaning                               |
+----------------+--------------------+--------------------------------------------------------------------+
| name           | [string]           | name of the package                                                |
| version        | [string]           | version of the package                                             |
| build          | [string]           | build hash of the package                                          |
| build_number   | [int]              | build number                                                       |
| license        | [string]           |                                                                    |
| license_family | [string]           |                                                                    |
| timestamp      | [int]              | timestamp of the package _build time_                              |
| size           | [int]              | size of the package file                                           |
| subdir         | [string]           |                                                                    |
| sha256         | [string, OPTIONAL] | SHA256 sum of the package, only available on CDN provided channels |
| md5            | [string]           | MD5 sum of the package file                                        |
| depends        | [list[string]]     | List of dependency strings, in ocnda-build format                  |
| constrains     | [list[string]]     | List of constrain strings, in conda-build format                   |

### Removed packages

Packages that were previously part of repodata.json, but are broken for some reason are placed in the `removed` list. This means, the solver will never attempt to install them, but if environment files where those files are explictly listed continue to work.

### Example

```
{
  "info": {
    "subdir": "linux-64"
  },
  "packages": {
    "2dfatmic-1.0-he991be0_0.tar.bz2": {
      "build": "he991be0_0",
      "build_number": 0,
      "depends": [
        "libgcc-ng >=7.3.0",
        "libgfortran-ng >=7,<8.0a0"
      ],
      "license": "Public Domain",
      "license_family": "OTHER",
      "md5": "a8519cbd6eaead777674c5c2e0e3ebe5",
      "name": "2dfatmic",
      "sha256": "a946804083b6ea9aa2fdf60da9eb173c4262d2dd2823e9d4de0f4a9ef903af29",
      "size": 172560,
      "subdir": "linux-64",
      "timestamp": 1581652603532,
      "version": "1.0"
    },
    ...
  },
  "conda.packages": {
    "2dfatmic-1.0-he991be0_0.conda": {
      "build": "he991be0_0",
      "build_number": 0,
      "depends": [
        "libgcc-ng >=7.3.0",
        "libgfortran-ng >=7,<8.0a0"
      ],
      "license": "Public Domain",
      "license_family": "OTHER",
      "md5": "a8519cbd6eaead777674c5c2e0e3ebe5",
      "name": "2dfatmic",
      "sha256": "a946804083b6ea9aa2fdf60da9eb173c4262d2dd2823e9d4de0f4a9ef903af29",
      "size": 172560,
      "subdir": "linux-64",
      "timestamp": 1581652603532,
      "version": "1.0"
    },
    ...
  },
  "removed": [
  	"2dfatmic-0.1-he991be0_0.conda"
  ]
}
```

## current_repodata.json

Conda generates a subset of repository data, called the `current_repodata.json`. The smaller `current_repodata.json` has benefits: faster download speeds for repository data and fewer packages which leads to faster solve times. Conda automatically falls back to the full repodata if a package cannot be found or installed from `current_repodata.json`.

The current_repodata.json also contains only one of the two packages (`.conda` or `.tar.bz2`).

The `current_repodata.json` generation is implemented in `conda-build`.