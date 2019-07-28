# emacs-tree-sitter [![Build Status](https://travis-ci.org/ubolonton/emacs-tree-sitter.svg?branch=master)](https://travis-ci.org/ubolonton/emacs-tree-sitter) [![Build Status](https://dev.azure.com/ubolonton/emacs-tree-sitter/_apis/build/status/ubolonton.emacs-tree-sitter?branchName=master)](https://dev.azure.com/ubolonton/emacs-tree-sitter/_build/latest?definitionId=2&branchName=master)

This is an Emacs Lisp binding for [tree-sitter](https://tree-sitter.github.io/tree-sitter/), an incremental parsing library.

It aims to be the foundation for a new breed of Emacs packages that understand code structurally. For examples:
- Faster, fine-grained code highlighting.
- More flexible code folding.
- Structural editing (like Paredit, or even better) for non-Lisp code.
- More informative indexing for imenu.

The author of tree-sitter articulated its merits a lot better in this [Strange Loop talk](https://www.thestrangeloop.com/2018/tree-sitter---a-new-parsing-system-for-programming-tools.html).

## Pre-requisites

- Emacs 25.1 or above, built with module support. This can be checked with `(functionp 'module-load)`.
- [Rust toolchain](https://rustup.rs/), to build the dynamic module.
- [tree-sitter CLI tool](https://tree-sitter.github.io/tree-sitter/creating-parsers#installation), to build loadable language files from grammar repos.

## Building and Installation

- Clone this repo.
- Build:
    ```bash
    ./bin/build
    ```
- Add this repo's directory to `load-path`.

## Getting Language Support
This package is currently not bundled with any language. Language support is to be loaded from shared dynamic libraries.

One way to get these shared libraries is to use the `tree-sitter` CLI tool.

For example, to get Rust support:

- Get the language's grammar:
    ```bash
    git clone https://github.com/tree-sitter/tree-sitter-rust
    cd tree-sitter-rust
    ```
- Build the shared lib from the grammar:
    ```bash
    # The library should be created at ~/.tree-sitter/bin/rust.so
    tree-sitter test
    ```
- Load it with `tree-sitter`:
    ```emacs-lisp
    (ts-load-language "rust")
    ```

## Basic Usage

```emacs-lisp
(require 'tree-sitter)

;;; Assuming ~/.tree-sitter/bin/rust.so was generated by tree-sitter CLI tool.
(setq rust (ts-load-language "rust"))
(setq parser (ts-make-parser))
(ts-set-language parser rust)

;;; Parse a simple string.
(ts-parse-string parser "fn foo() {}")

(with-temp-buffer
  (insert-file-contents "src/types.rs")
  (let* ((tree)
         (initial (benchmark-run (setq tree (ts-parse parser #'ts-buffer-input nil))))
         (reparse (benchmark-run (ts-parse parser #'ts-buffer-input tree))))
    ;; Second parse should be much faster than the initial parse, especially as code size grows.
    (message "initial %s" initial)
    (message "reparse %s" reparse)))
```

## APIs

The [tree-sitter doc](https://tree-sitter.github.io/tree-sitter/using-parsers) is a good read to understand its concepts, and how to use the parsers in general.

Functions in this package are named differently, to be more Lisp-idiomatic. The overall parsing flow stays the same.

Documentation for individual functions can be viewed with `C-h f` (`describe-function`), as usual.

A `symbol` in the C API is actually the ID of a type, so it's called `type-id` in this package.

### Types

This package exposes the following types:

- `point`: a vector in the form of `[row column]`, where `row` and `column` are zero-based. Note that this is different from Emacs's concept of "point".
- `range`: a vector in the form of `[start-point end-point]`.
- `language`, `parser`, `tree`, `node`, `cursor`: corresponding tree-sitter types, embedded in `user-ptr` objects.

Each of the above types has a corresponding type-checking predicate `ts-<type->-p`, which is useful for debugging.

Note that these types are understood only by this package. They are not recognized by `type-of`.

### Functions

- Parser:
    + `ts-make-parser`: create a new parser.
    + `ts-set-language`: set a parser's active language.
    + `ts-parse-string`: parse a string.
    + `ts-parse`: parse with a text-generating callback.
    + `ts-set-included-ranges`: set sub-ranges when parsing multi-language text.
- Tree:
    + `ts-changed-ranges`: compare 2 trees for changes.
    + `ts-edit-tree`: prepare a tree for incremental parsing.
    + `ts-root-node`: get the tree's root node.
    + `ts-clone-tree`: make a copy of the tree.
    + `ts-tree-to-sexp`: debug utility.
- Cursor:
    + `ts-goto-` functions: move to a different node.
    + `ts-current-` functions: get the current field/node.
    + `ts-make-cursor`: obtain a new cursor from either a tree or a node.
- Node:
    + `ts-node-` functions: node's properties and predicates.
    + `ts-get-` functions: get related nodes (parent, siblings, children, descendants).
    + `ts-count-` functions: count child nodes.
    + `ts-mapc-children`: loops through child nodes.
    + `ts-node-to-sexp`: debug utility.

## Development
- Make sure necessary languages are installed for tests:
    ```bash
    ./bin/ensure-lang rust
    ```
- Testing:
    ```bash
    ./bin/test
    ```
- Continuous testing (requires [cargo-watch](https://github.com/passcod/cargo-watch)):
    ```shell
    ./bin/test watch
    ```

On Windows, use PowerShell to run the corresponding `.ps1` scripts.


## TODOs
- [ ] Implement a `tree-sitter-minor-mode` that exposes an always-up-to-date syntax tree, by integrating incremental parsing with Emacs's change hooks.
    ```emacs-lisp
    ;;; WIP
    (progn
      (setq tree-sitter-language (ts-load-language "rust"))
      (tree-sitter-mode +1))
    ```
- [ ] Implement syntax highlighting, maybe using [tree-sitter-highlight](https://github.com/tree-sitter/tree-sitter/tree/master/highlight).
- [ ] Support multi-language buffers (with `ts-set-included-ranges`).
- [ ] Allow parsing to be paused/cancelled by integrating tree-sitter cancellation/timeout support with `should_quit`.
