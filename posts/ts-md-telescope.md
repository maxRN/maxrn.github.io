# Writing a Markdown extension for Telescope that uses treesitter queries

- Use treesitter queries to determine headings etc.
- Use Telescope to display + navigate structure of document
- Show document headings in proper order and proper intendation

Inspired by [telescope-heading](https://github.com/crispgm/telescope-heading.nvim)

## How to write a Treesitter query

- [Syntax](https://tree-sitter.github.io/tree-sitter/using-parsers#query-syntax) based on LISP
- Need to find predicates? directives? using vim treesitter API
    - `vim.treesitter.query.list_directives()`
    - `vim.treesitter.query.list_predicates()`
- What is a *directive*? What is a *predicate*?
- How does the query look like?
        - e.g. `(atx_heading) @capture` shows all headings
- Can we just filter for headings?
- How can I see the entire treesitter tree?
    - install [treesitter playground](https://github.com/nvim-treesitter/playground)
    - run `:TSPlaygroundToggle` to see AST of document
        - while in this tree view, hit `o` to start query editor
        - e.g. `(atx_heading) @capture` shows all headings
    - I think `tstree:root()` and `tsnode:iter_children()` or `tsnode:child_count()` would be interesting
    - how to get tree? I tried `lua vim.treesitter.get_parser(0):parse()` however that only returns { <userdata1> }
    - `vim.treesitter.get_parser(0)` actually returns a parser but I'm not sure if that is the correct one
    

## How to write a Telescope plugin
