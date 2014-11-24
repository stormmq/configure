# `configure`: functions module for shellfire

This module provides a simple framework for managing configuration of a [`shellfire`] application. It is deliberately separate to the [`core`] module, which provides automatic loading of configuration for the command line at run time. Instead, it lets users define configuration in both files and directories of snippets, which can be overridden, defaulted, validated and restored.

An example user is the [`swaddle`] packaging tool.

## Overview

Configuration is organised into namespaces which are _private_ to your program; specifying the same namespace in different programs will give different results.

To get going, first we need to register some settings:-

```bash
configure_register 'Value' 'Boolean' 'swaddle_web' 'use_index_name_in_directory_links' yes
configure_register 'Array' 'Any' 'swaddle_web pandoc_options'
configure_register 'Value' 'NotEmpty' 'swaddle_web' 'index_name' 'index.html'
```

Note that there's no need to register the namespace; this happens automatically. The framework supports `Value`s and `Array`s. A `Value` may have a default (for example, `index_name` above defaults to `index.html`). Arrays can not have defaults. By convention, namespaces are organised in a hierarchy using underscores, `_`, and names use lower case and underscores `_`. This makes them easy to set by administrative users. There is no need to quote the values, either; this is done above to take advantage of syntax highlighting.

Then we source our configuration:-

```bash
configure_reset 'swaddle_web'
configure_source '/path/to/configuration/folder' "myconf"
```

This will reset any configuration, then load 

## Importing

To import this module, add a git submodule to your repository. From the root of your git repository in the terminal, type:-
```bash
mkdir -p lib/shellfire
cd lib/shellfire
git submodule add "https://github.com/shellfire-dev/configure.git"
cd -
git submodule init --update
```

You may need to change the url `https://github.com/shellfire-dev/configure.git"` above if using a fork.

You will also need to add paths - include the module [`paths.d`].


## Namespace `configure`

### To use in code

If calling from another [`shellfire`] module, add to your shell code the line
```bash
core_usesIn configure
```
in the global scope (ie outside of any functions). A good convention is to put it above any function that depends on functions in this module. If using it directly in a program, put this line _inside_ the `_program()` function:-

```bash
_program()
{
	core_usesIn configure
	…
}
```

### Functions


#### `configure_register`

|Parameter|Value|Optional|
|---------|-----|--------|
|`kind`|Namespace used to differentiate configuration setting from any other|_No_|
|`configurationValidationFunction`|Use the function called `configure_validate_${configurationValidationFunction}` to validate configuration|_No_|
|`namespace`|Namespace used to differentiate configuration setting from any other|_No_|
|`configurationSettingName`|Name of configuration setting|_No_|
|`default`|When `kind` is `Value`, may be specified if a default is wanted. Not permitted when `kind` is `Array`|_Yes_|

The values of `kind` may be:-

|Value|Description|
|-----|-----------|
|`Value`|A single value|
|`Array`|An array of zero or more values|

The values of `configurationValidationFunction` are not constrained; you may define your own. Those supplied in this module are:-

|Value|Description|
|-----|-----------|
|`Any`|Any value permitted|
|`NotEmpty`|Any value apart from empty permitted|
|`Boolean`|Value must be a boolean (eg yes, true, 1, on, Y, T, etc) as per `core_variable_isBoolean`|
|`ReadableSearchableFolderPath`|Value must be a readable, searchable folder path|

See the [`swaddle`] project for more examples of validation functions.

A `namespace` must consist only of lower and upper case letters, digits and underscores, to be compatible with all POSIX shells. This is not enforced, however.

The `default` may be whitespace-separated without using quotes; all values are concatenated together with a single `SP` separator.


#### `configure_isValueValid`

|Parameter|Value|Optional|
|---------|-----|--------|
|`snippetName`|Name of a snippet|_No_|
|`configurationSettingValue`|Value that would be retrieved from `configure_getValue`|_No_|

This is a helper function to call when defining your own [`configurationValidationFunction`](#configure_register). It allows you to embed a list of values inside your code that might change. It uses the [`core`] snippet functions. For example:-
```bash
core_snippet_embed raw validate_rpm_licence
configure_validate_RpmLicence()
{
	configure_isValueValid validate_rpm_licence "$1"
}
```
where `validate_rpm_licence` is the file `lib/shellfire/${_program_name}/validate_rpm_licence.snippet`, and might contain a list such as:-
```bash
AGPLv1
Artistic 2.0
…
```

Each entry is terminated by a line feed, including the final entry. If the value specified by configuration is in this list, it's valid. If it's not, its invalid. You can use your new validation function when registering, eg
```bash
configure_register Value RpmLicence swaddle_rpm licence
```


#### `configure_getValue`

|Parameter|Value|Optional|
|---------|-----|--------|
|`namespace`|Namespace used to differentiate configuration setting from any other|_No_|
|`configurationSettingName`|Name of configuration setting|_No_|

Use this function to obtain the `Value` of a configuration setting. If nothing was configured, the default specified at registration is returned. This does not work for `Array`s. For example, to find the `repositoryName`:-
```bash
…
local repositoryName="$(configure_getValue 'swaddle_github' 'repository_name')"
…
```

#### `configure_iterateOverArrayWithDefaultsIfEmpty`

|Parameter|Value|Optional|
|---------|-----|--------|
|`namespace`|Namespace used to differentiate configuration setting from any other|_No_|
|`configurationSettingName`|Name of configuration setting|_No_|
|`callback`|Name of a function (or program on the path)|_No_|
|`...`|Zero or more arguments to use as defaults to iterate over if the configuration setting is an empty array|_Yes_|

Use this function to iterate over each value in an `Array` configuration setting, passing each value as the variable `core_variable_array_element` to the function `callback`.  If nothing was configured, then a set of defaults can be supplied as additional arguments `...`.

For example,
```bash

some_callback()
{
	echo "architecture: $core_variable_array_element"
}

configure_iterateOverArrayWithDefaultsIfEmpty 'swaddle_apt' 'architectures' some_callback 'amd64' 'i386'
```
will iterate over whatever is in `architectures`, or, if, empty, the values `amd64` and `i386`.


#### `configure_callFunctionWithDefaultsIfEmpty`

|Parameter|Value|Optional|
|---------|-----|--------|
|`namespace`|Namespace used to differentiate configuration setting from any other|_No_|
|`configurationSettingName`|Name of configuration setting|_No_|
|`callback`|Name of a function (or program on the path)|_No_|
|`...`|Zero or more arguments to use as defaults to iterate over if the configuration setting is an empty array|_Yes_|

Use this function to pass all the values in an `Array` configuration setting to a callback. If nothing was configured, then a set of defaults can be supplied as additional arguments `...`.

For example,
```bash

some_callback()
{
	echo "number of elements in array: $#"
	echo "Values: $@"
}

configure_iterateOverArrayWithDefaultsIfEmpty 'swaddle_apt' 'architectures' some_callback 'amd64' 'i386'
```
will pass whatever is in `architectures`, or, if, empty, the values `amd64` and `i386` (and so `$#` is 2).




[shellfire]: https://github.com/shellfire-dev "shellfire homepage"
[core]: https://github.com/shellfire-dev/core "shellfire core module homepage"
[swaddle]: https://github.com/raphaelcohn/swaddle "Swaddle homepage"
[paths.d]: https://github.com/shellfire-dev/paths.d "paths.d shellfire module homepage"
