# Configuration

The browser tab in which your PyScript based web page is displayed is a very
secure sandboxed computing environment for running your Python code.

This is also the case for web workers running Python. Despite being associated
with a single web page, workers are completely separate from each other
(except for some very limited and clearly defined means of interacting, which
PyScript looks after for you).

We need to tell PyScript how we want such Python environments to be configured.
This works in the same way for both the main thread and for web workers. Such
configuration ensures we get the expected resources ready before our Python
code is evaluated (resources such as arbitrary data files, third party Python
packages and PyScript plugins).

## TOML or JSON

Configuration can be expressed in two formats:

* [TOML](https://toml.io/en/) is the configuration file format preferred by
  folks in the Python community.
* [JSON](https://www.json.org/json-en.html) is a data format most often used
  by folks in the web community.

Since PyScript is the marriage of Python and the web, and we respect the
traditions of both technical cultures, we support both formats.

However, because JSON is built into all browsers by default and TOML requires
an additional download of a specialist parser before PyScript can work, **the
use of JSON is more efficient from a performance point of view**.

The following two configurations are equivalent, and simply tell PyScript to
ensure the packages [arrr](https://arrr.readthedocs.io/en/latest/) and
[numberwang](https://numberwang.readthedocs.io/en/latest/) are installed from
PyPI (the [Python Packaging Index](https://pypi.org/)):

```TOML title="Configuration via TOML"
packages = ["arrr", "numberwang" ]
```

```JSON title="Configuration via JSON"
{
    "packages": ["arrr", "numberwang"]
}
```

## File or inline

The recommended way to write configuration is via a separate file and then
reference it from the tag used to specify the Python code:

```HTML title="Reference a configuration file"
<script type="py" src="main.py" config="pyscript.toml"></script>
```

If you use JSON, you can make it the value of the `config` attribute:

```HTML title="JSON as the value of the config attribute"
<script type="mpy" src="main.py" config='{"packages":["arrr", "numberwang"]}'></script>
```

For historical and convenience reasons we still support the inline
specification of configuration information via a _single_ `<py-config>` or
`<mpy-config>` tag in your HTML document:

```HTML title="Inline configuration via the &lt;py-config&gt; tag"
<py-config>
{
    "packages": ["arrr", "numberwang" ]
}
</py-config>
```

!!! warning

    Should you use `<py-config>` or `<mpy-config>`, **there must be only one of
    these tags on the page per interpreter**.

## Options

There are four core options ([`interpreter`](#interpreter), [`files`](#files),
[`packages`](#packages) and [`plugins`](#plugins)). The user is free to define
arbitrary additional configuration options that plugins or an app may require
for their own reasons.

### Interpreter

The `interpreter` option pins the Python interpreter to the version of the
specified value. This is useful for testing (does my code work on a specific
version of Pyodide?), or to ensure the precise combination of PyScript version
and interpreter version are pinned to known values.

The value of the `interpreter` option should be a valid version number
for the Python interpreter you are configuring, or a fully qualified URL to
a custom version of the interpreter.

The following two examples are equivalent:

```TOML title="Specify the interpreter version in TOML"
interpreter = "0.23.4"
```

```JSON title="Specify the interpreter version in JSON"
{
    "interpreter": "0.23.4"
}
```

The following JSON fragment uses a fully qualified URL to point to the same
version of Pyodide as specified in the previous examples:

```JSON title="Specify the interpreter via a fully qualified URL"
{
    "interpreter": "https://cdn.jsdelivr.net/pyodide/v0.23.4/full/pyodide.mjs"
}
```

### Files

The `files` option fetches arbitrary content from URLs onto the filesystem
available to Python, and emulated by the browser. Just map a valid URL to a
destination filesystem path.

The following JSON and TOML are equivalent:

```json title="Fetch files onto the filesystem with JSON"
{
  "files": {
    "https://example.com/data.csv": "./data.csv",
    "/code.py": "./subdir/code.py"
  }
}
```

```toml title="Fetch files onto the filesystem with TOML"
[files]
"https://example.com/data.csv" = "./data.csv"
"/code.py" = "./subdir/code.py"
```

If you make the target an empty string, the final "filename" part of the source
URL becomes the destination filename, in the root of the filesystem, to which
the content is copied. As a result, the `data.csv` entry from the previous
examples could be equivalently re-written as:

```json title="JSON implied filename in the root directory"
{
  "files": {
    "https://example.com/data.csv": "",
    ... etc ...
  }
}
```

```toml title="TOML implied filename in the root directory"
[files]
"https://example.com/data.csv" = ""
... etc ...
```


!!! warning

    **PyScript expects all file destinations to be unique.**

    If there is a duplication PyScript will raise an exception to help you find
    the problem.

!!! tip

    **For most people, most of the time, the simple URL to filename mapping,
    described above, will be sufficient.**

    Yet certain situations may require more flexibility. In which case, read
    on.

Sometimes many resources are needed to be fetched from a single location and
copied into the same directory on the file system. To aid readability and
reduce repetition, the `files` option comes with a mini
[templating language](https://en.wikipedia.org/wiki/Template_processor)
that allows re-usable placeholders to be defined between curly brackets (`{`
and `}`). When these placeholders are encountered in the `files` configuration,
their name is replaced with their associated value.

!!! Attention

    Valid placeholder names are always enclosed between curly brackets
    (`{` and `}`), like this: `{FROM}`, `{TO}` and `{DATA SOURCE}`
    (capitalized names help identify placeholders
    when reading code ~ although this isn't strictly necessary).

    Any number of placeholders can be defined and used anywhere within URLs and
    paths that map source to destination.

The following JSON and TOML are equivalent:

```json title="Using the template language in JSON"
{
  "files": {
    "{DOMAIN}": "https://my-server.com",
    "{PATH}": "a/path",
    "{VERSION}": "1.2.3",
    "{FROM}": "{DOMAIN}/{PATH}/{VERSION}",
    "{TO}": "./my_module",
    "{FROM}/__init__.py": "{TO}/__init__.py",
    "{FROM}/foo.py": "{TO}/foo.py",
    "{FROM}/bar.py": "{TO}/bar.py",
    "{FROM}/baz.py": "{TO}/baz.py",
  }
}
```

```toml title="Using the template language in TOML"
[files]
"{DOMAIN}" = "https://my-server.com"
"{PATH}" = "a/path"
"{VERSION}" = "1.2.3"
"{FROM}" = "{DOMAIN}/{PATH}/{VERSION}"
"{TO}" = "./my_module"
"{FROM}/__init__.py" = "{TO}/__init__.py"
"{FROM}/foo.py" = "{TO}/foo.py"
"{FROM}/bar.py" = "{TO}/bar.py"
"{FROM}/baz.py" = "{TO}/baz.py"
```

The `{DOMAIN}`, `{PATH}`, and `{VERSION}` placeholders are
used to create a further `{FROM}` placeholder. The `{TO}` placeholder is also
defined to point to a common sub-directory on the file system. The final four
entries use `{FROM}` and `{TO}` to copy over four files (`__init__.py`,
`foo.py`, `bar.py` and `baz.py`) from the same source to a common destination
directory.

For convenience, if the destination is just a directory (it ends with `/`)
then PyScript automatically uses the filename part of the source URL as the
filename in the destination directory.

For example, the end of the previous config file could be:

```toml
"{TO}" = "./my_module/"
"{FROM}/__init__.py" = "{TO}"
"{FROM}/foo.py" = "{TO}"
"{FROM}/bar.py" = "{TO}"
"{FROM}/baz.py" = "{TO}"
```

### Packages

The `packages` option defines a list of Python `packages` to be installed from
[PyPI](https://pypi.org/) onto the filesystem by Pyodide's 
[micropip](https://micropip.pyodide.org/en/stable/index.html) package
installer.

!!! warning

    Because `micropip` is a Pyodide-only feature, and MicroPython doesn't
    support code packaged on PyPI, **the `packages` option is only available
    for use with Pyodide**.

    If you need **Python modules for MicroPython**, use the
    [files](#files) option to manually copy the source code onto the
    file system.

The following two examples are equivalent:

```TOML title="A packages list in TOML"
packages = ["arrr", "numberwang", "snowballstemmer>=2.2.0" ]
```

```JSON title="A packages list in JSON"
{
    "packages": ["arrr", "numberwang", "snowballstemmer>=2.2.0" ]
}
```

The names in the list of `packages` can be any of the following valid forms:

* A name of a package on PyPI: `"snowballstemmer"`
* A name for a package on PyPI with additional constraints:
  `"snowballstemmer>=2.2.0"`
* An arbitrary URL to a Python package: `"https://.../package.whl"`
* A file copied onto the browser based file system: `"emfs://.../package.whl"`

### Plugins

The `plugins` option lists plugins enabled by PyScript to add extra
functionality to the platform.

Each plugin should be included on the web page, as described in the
[plugins](#plugins_1) section below. Then the plugin's name should be listed.

```TOML title="A list of plugins in TOML"
plugins = ["custom_plugin", "!error"]
```

```JSON title="A list of plugins in JSON"
{
    "plugins": ["custom_plugin", "!error"]
}
```

!!! info

    The `"!error"` syntax is a way to turn off a plugin built into PyScript
    that is enabled by default.

    Currently, the only built-in plugin is the `error` plugin to display a
    stack trace and error messages in the DOM. More may be added at a later
    date.

### Custom 

Sometimes plugins or apps need bespoke configuration options.

So long as you don't cause a name collision with the built-in option names then
you are free to use any valid data structure that works with both TOML and JSON
to express your configuration needs.

**TODO: explain how to programmatically get access to an object representing
the config.**

