---
title: Easy Makefile self-documentation
categories: article
tags: dev makefile
---


Here's an easy way to generate makefile help command from its comments that I saw some time ago somewhere.

```
.DEFAULT_GOAL := help

help:
	@grep -E '^[a-zA-Z0-9_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-15s\033[0m %s\n", $$1, $$2}'

command1: command2 ## Alias of command2
	echo 'Done!'

command2: ## run command2
	echo 'Running command2'

command3: # Unlisted command
	@echo "I'm private"
```

## Explanation

1. `.DEFAULT_GOAL` will run provided make command if none is specified, so by doing `make` it will execute `make help`
2. `@grep -E '^[a-zA-Z0-9_-]+:.*?## .*$$' $(MAKEFILE_LIST)`:
  - `@` will suppress output because by default make will print out the command it's executing
  - `grep ...` will look for lines that have commands with comments separated by `##`
  - `'^[a-zA-Z0-9_-]+:.*?## .*$$'` means search from start of line, any alphanumeric, `_` and `-` command names followed by `:` and any character until `##` to end of line (since `$` has special meaning in make, we have to escape it so end of line becomes `$$`)
  - [`MAKEFILE_LIST`](https://ftp.gnu.org/old-gnu/Manuals/make-3.80/html_node/make_17.html) is make variable that refers to the makefile name in this case
3. `sort` to get alphabetically sorted output
4. `awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-15s\033[0m %s\n", $$1, $$2}'`:
  - pipe the lines through `awk` and set the [`FS`](https://www.gnu.org/software/gawk/manual/html_node/Field-Separators.html) field separator to be whatever between the command and help text (modify `##` if you want different separator)
  - for each match print with [color](https://en.wikipedia.org/wiki/ANSI_escape_code#Colors): `\033[36m%-15s\033[0m %s\n` so we will get command, [padded](https://www.gnu.org/software/gawk/manual/html_node/Format-Modifiers.html) with 15 spaces, followed by the help text
  - again since we're inside make escape `$1` & `$2` for column number with `$$1` & `$$2`
5. For commands you don't want to be listed as `help` output, simply don't comment or use single `#` like `command3` example


## Output

```sh
$ make
command1        Alias of command2
command2        run command2
```
