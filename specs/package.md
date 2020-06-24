


Paths data
----------

Path data for a package exists in two and a half versions: 0.9, 0.1 and 1.

The path data for version 0 is stored in multiple files:

- `info/files`
- `info/no_link`
- `info/no_softlink`
- `info/has_prefix`

The files work as follows:

#### has_prefix

The has_prefix is a plaintext text file that contains the following content. 

In version 0.0 it contains all names of files that contain the prefix (to be replaced during the installation process). The files are seperated by a new line. The prefix is the constant `"/opt/anaconda1anaconda2anaconda3"`.

In version 0.1 the has_prefix file contains 3 entries, seperated by a space character, per line:

`PLACEHOLDER␣FILE_MODE␣FILE_PATH`

The entries can be quoted (for example if the path contains a space).

The placeholder is the prefix placeholder (e.g. `/some/path/host_placeholder...`), the file mode is either the string `"text"` or "`"binary"`.

File path is the path to the file in the unpacked tar ball.

The paths data for version 1 is stored in a single json file  

- `info/paths.json`

The JSON contains the following entries:

```json
{
	"paths_version": <integer>,
	"paths": [
		<PATH_ENTRIES>
	]
}
```

Where PATH_ENTRIES are objects of the following form:

```json
{
	"_path": <string>,
	"path_type": string["softlink" | "hardlink"],
	"sha256": string | null,
	"size_in_bytes": int,
	"prefix_placeholder": optional[string],

}
```

Question: can "directory" be another option for `path_type`?

Softlinks have a `sha256: null` entry.
