# Editors

VS Code support ships in this repository. It is a client of about five hundred lines of plain JavaScript, plus `vyrn-lsp`. The server answers every question by calling the compiler front end `vyrn check` calls, so an error in your editor is the error your build prints.

It is not on a marketplace yet. Every release publishes the extension as a single `.vsix`, one file for every platform. The server ships in the platform archive beside `vyrn`.

## 18 features

Each row names the file that implements it.

- **Errors while you type** — The compiler's own diagnostics, on the line and the token they name. The squiggle and `vyrn check` come from the same front end, so they never disagree. Implemented by `compiler/vyrn-lsp/src/main.rs` (`analyze_doc / publish`).
- **Across the files you import** — Imports are resolved and analyzed. A name from another module is not an unknown one, and an error inside an imported file is reported against the file you have open. The editor reads a remote module from the vendor directory or the cache, and never fetches. Implemented by `compiler/vyrn-lsp/src/main.rs` (`load_context`).
- **Hover** — The type of a binding, the signature of a function, the arms of an enum, and the doc comment above any of them. For a CSS class, the rule out of your own stylesheet. Implemented by `compiler/vyrn-lsp/src/main.rs` (`handle_hover`).
- **Go to definition** — F12 on a name opens where it is declared, including across a module boundary; on a component tag in a template it opens that component's own file. Implemented by `compiler/vyrn-lsp/src/main.rs` (`handle_definition`).
- **Completion** — Top-level names, locals and parameters, and `.` members off the receiver's type. That covers built-in methods, record fields, and the methods a `protocol` gives a type. Implemented by `compiler/vyrn-lsp/src/main.rs` (`handle_completion`).
- **Outline and breadcrumbs** — Ctrl-Shift-O and the breadcrumb bar list this document's own functions, methods, types, variants and `test` blocks. An imported name is not in the list. Implemented by `compiler/vyrn-lsp/src/main.rs` (`handle_document_symbol`).
- **Highlight the binding, not the word** — The cursor on a name marks its real references, with reads and writes told apart. A same-named binding in another scope stays unmarked, and so does the word inside a comment. Implemented by `compiler/vyrn-lsp/src/main.rs` (`handle_document_highlight`).
- **Rename, including what a generator derived** — Renaming a procedure rewrites the call sites your source never names: the client stub a page calls, the dispatch name, the re-export a projection made. A spelling it cannot derive refuses the whole rename rather than doing half of it. Implemented by `compiler/vyrn-lsp/src/rename.rs` (`rename_at`).
- **Quick fix on a contract** — A page declaring an export its contract does not name gets the did-you-mean as an edit you can apply: `laod` to `data`. Implemented by `compiler/vyrn-lsp/src/main.rs` (`handle_code_action`).
- **Format, and format on save** — Shift+Alt+F, or VS Code's own format-on-save with no extra setting, runs the canonical formatter over the whole document. A buffer that does not lex is returned unchanged rather than corrupted mid-edit. Implemented by `compiler/vyrn-lsp/src/main.rs` (`handle_formatting`).
- **Inlay hints** — The type of a binding whose line does not say it, and a label at each point a value is moved. `editor.inlayHints.enabled` turns them off. Implemented by `compiler/vyrn-lsp/src/main.rs` (`handle_inlay_hint`).
- **Semantic colour** — Every identifier is classified from the analysis: namespace, type, variant, parameter, property, function, method. A grammar cannot do that, because a grammar sees only text. Implemented by `compiler/vyrn-lsp/src/main.rs` (`handle_semantic_tokens_full`).
- **Templates are the same language** — A file a generator reads, such as a `.vyx` template, gets the same diagnostics, hover, definition, highlight, hints and completion, mapped back into the coordinates you wrote. The generator's own module adds the completions of its grammar; for `.vyx` they are tag names, sibling components, props and `@events`, and class tokens come from the theme. Implemented by `compiler/vyrn-lsp/src/main.rs` (`generator_completion`).
- **Syntax highlighting** — A TextMate grammar derived from the lexer's own token set. It colours `.vyrn`, `.von` and `.vyx` with no server running. Implemented by `editor/vscode/vyrn.tmLanguage.json` (`source.vyrn`).
- **Snippets** — Eleven, for the shapes with a body to fill in: `fn`, `main`, record and enum `type`, `protocol`, `impl`, `match`, `import`, the `logging` block, `test` and `bench`. Implemented by `editor/vscode/snippets/vyrn.json` (`vyrn.json`).
- **Run it from the file** — A **▶ Run** above `fn main`, a **▶ Run test** above each `test` block and the same pair for `bench`. Each one runs the real command in the integrated terminal. Implemented by `editor/vscode/extension.js` (`provideCodeLenses`).
- **▶ Run dev server, where it means something** — The lens appears only on a file that is a dev root: one importing `std/rpc` with an `rpcServer` call site. A second press restarts the server rather than stacking another on the port. Implemented by `compiler/vyrn-lsp/src/main.rs` (`handle_is_dev_entry`).
- **The wire path above the procedure** — Over each procedure a generator mounts, the method and path it is reachable at. Nothing in the file states it: the path comes from the file's location and the export's name. Implemented by `compiler/vyrn-lsp/src/main.rs` (`handle_route_lenses`).

The link opens the file on GitHub.

What it does not do. A `.von` file is coloured and not analyzed, and the run lenses are on `.vyrn` files only. The server has no plugin API and no setting beyond where to find the two binaries.

## Install

The extension lives in `editor/vscode` and the server in `compiler/vyrn-lsp`. Both are published: the server inside every platform archive, the extension as one `.vsix` asset. Build from source only to work on the extension.

### Install the toolchain, which carries the server

```sh
curl -fsSL https://vyrn-lang.github.io/install.sh | sh
```

The archive holds `vyrn-lsp` beside `vyrn`, and the installer puts both in `~/.vyrn/bin`. On Windows the line is `powershell -c "irm https://vyrn-lang.github.io/install.ps1|iex"`. If you have run either, you have the server.

### Download the .vsix from the newest release and install it

```sh
code --install-extension vyrn-vscode-<version>.vsix
```

One file for every platform. The extension finds the server on `PATH`, or beside the `vyrn` that is on `PATH`. `vyrn.serverPath` overrides both.

### Or work on the extension itself

```sh
cargo build --manifest-path compiler/vyrn-lsp/Cargo.toml
```

A **debug** build, which is the one the F5 development host looks for. Then open the repository in VS Code and press F5. The launch does not rebuild the server, so re-run this line whenever you change it. `npm run package` in `editor/vscode` builds the same `.vsix` the release does.

Two settings, both optional. `vyrn.serverPath` points at a server binary somewhere else. `vyrn.path` points at the compiler the run lenses invoke. Left empty, each one searches `PATH` first and your checkout after.

Modules resolve from `vyrn.json`. The editor reads the cache of pinned imports and never writes to it, so nothing in your editor fetches code. [Projects and dependencies](https://vyrn-lang.github.io/tooling/projects.html) explains the pins.

## Other editors

`vyrn-lsp` is an ordinary language server: it speaks JSON-RPC over stdin and stdout and knows nothing about VS Code. Any editor with a generic LSP client gets the whole table above except the lenses, which the client draws. There is no socket mode, so a client that cannot spawn a process cannot use it.

The rest is data any editor can read: the grammar is TextMate JSON, and the snippets are one flat file.

---

From https://vyrn-lang.github.io/tooling/editors.html — generated by `site/export.vyrn` from the Vyrn repository.
