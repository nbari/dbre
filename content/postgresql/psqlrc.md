+++
title = ".psqlrc"
weight = 5
+++

`psql` can be customized by using a `.psqlrc` file, for example

```config
\set QUIET 1
\set COMP_KEYWORD_CASE upper
\set HISTFILE ~/.psql_history- :DBNAME
\set HISTCONTROL ignoredups
\set PROMPT2 '... > '
\pset pager off
\pset null '<null>'
\timing
\x auto
```
