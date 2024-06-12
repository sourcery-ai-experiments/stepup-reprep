# Inventory Files

StepUp RepRep uses `inventory.txt` to collect metadata to prepare a ZIP archive.
Inventory files can either be created manually (for an external dataset) or automatically (for datasets created within a StepUp workflow.)


## File Formats

### Inventory Files (`inventory.txt`)

The `inventory.txt` file contains one line per file to be archived.
Each line has four fields:

- the file size,
- the file mode,
- the [BLAKE2b](https://en.wikipedia.org/wiki/BLAKE_(hash_function)#BLAKE2) hash of the file contents (regular file) or link destination (symbolic link), and
- the relative path to the file, starting from the parent of the inventory file.

A fixed column width is used.
You do not create this type of file manually.
Instead, StepUp RepRep offers tools to make such files.
(See below.)


### Inventory Definitions (`inventory.def`)

The `inventory.def` format is inspired by the `MANIFEST.in` format from the [setuptools](https://setuptools.pypa.io/) project.
See [`MANIFEST.in` format documentation](https://setuptools.pypa.io/en/latest/userguide/miscellaneous.html#using-manifest-in) for details.
As of version 1.2.0, StepUp RepRep no longer relies on setuptools and has its own implementation to process inventory definitions.

Inventory definition files are processed one line at a time.
Comments start with the `#` character, just as in Python source code.
Empty lines or lines containing only comments are ignored.
A Non-empty line must contain one command starting with `include` or `exclude`.
Each command first generates a new list of paths.
An `include` command will add the newly generated paths to the inventory,
whereas an `exclude` command will remove them from the inventory (if present).
Each command operates on the file list constructed by preceding commands.
(The first command starts from an empty list.)

The following rules are supported:

- A bare `include` or `exclude` is follewed by one or more [Named Glob patterns](https://reproducible-reporting.github.io/stepup-core/reference/stepup.core.nglob/),
  which may also contain `**` wildcards to match files recursively.
  An exception is raised when no matching paths are found.
- The `include-git` and `exclude-git` use `git ls-files` to generate a list of files.
  This command takes no arguments.
- The `include-workflow` and `exclude-workflow` extract a file list
  from one or more StepUp `workflow.mpl.xz` files.
  The first argument is the state of the files to be selected.
  (`STATIC`, `BUILT` or `VOLATILE` are common. Other states exist but make less sense.)
  All subsequent arguments are (named glob patterns matching) StepUp workflow files.


## Creating `inventory.txt` Files.

### Command-line Tool `reprep-make-inventory`

One can create an inventory file on the command-line as follows:

```bash
reprep-make-inventory -i inventory.def -o inventory.txt
```

See `reprep-make-inventory --help` for more details.
This tool is suitable for creating inventory files of external datasets.

### StepUp RepRep Function `make_inventory`

One may include a [`make_inventory()`][stepup.reprep.api.make_inventory] step in `plan.py` as follows:

```python
from stepup.reprep.api import make_inventory

make_inventory(["file1.txt", "file2.txt", ...], "inventory.txt")
```

This step is mainly useful for creating an inventory file of a dataset generated by your StepUp workflow.
It can also be useful in combination with `stepup.core.glob` for building datasets from static files.

## Creating a ZIP Archive From a `inventory.txt` File

### Command-line Tool `reprep-zip-inventory`

Given an `inventory.txt` file, the corresponding ZIP file is created with:

```bash
reprep-zip-inventory inventory.txt
```

This is a command-line wrapper around the `zip_inventory` function discussed below.

### StepUp RepRep Function `zip_inventory`

The function [`zip_inventory()`][stepup.reprep.api.zip_inventory] takes a `inventory.txt` file as input and creates a ZIP file containing all the files listed in the inventory.
It differs from the conventional `zip` program in the following ways:

- File hashes are checked before adding files to the archive,
  which is mostly useful for external datasets.
- The `inventory.txt` file is included in the resulting ZIP file.
- The ZIP file is reproducible: all time stamps in the ZIP file are set to January 1st, 1980.
- By default, symbolic links are added as links, instead of the contents they point to.


### Unpacking the ZIP file

To unpack the ZIP file, use the following command:

```bash
unzip inventory.zip
```

This will correctly handle symbolic links, unlike Python's built-in
[extractall](https://docs.python.org/3/library/zipfile.html#zipfile.ZipFile.extractall) method.
See [cpython#82102](https://github.com/python/cpython/issues/82102)
for more details on Python's (lacking) support for symbolic links in ZIP files.

## Validate an Inventory File

If you just want to check the file sizes, modes and hashes of an inventory, run:

```bash
reprep-check-inventory inventory.txt
```

This can be useful in the following cases:

- When you work with a remote dataset, you can check if the files in the inventory have changed.
- When you unpack a ZIP file created `reprep-zip-inventory` or `zip_inventory()`,
  you can check if the files are not affected by bit rot or other data integrity issues.