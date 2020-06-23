# eql-style-guide

> CONTRIBUTING: Feel free to open a issue or PR ;)

Common patterns to develop [EQL](https://edn-query-language.org/) API's.

This guide will cover both `EQL` as a abstract API and common libraries usage, like [fulcro](https://github.com/fulcrologic/fulcro) and [pathom](https://github.com/wilkerlucio/pathom)

# EQL

## Keep it flat

If you have 1..1 relationship, it should be accessible without a `join`

```clojure
;; good: a session has a account, so I can access account attributes directly
[:app.current-session/token
 :app.account/display-name]

;; good: it's OK to have a explicit relationship attribute between them
[:app.current-session/token
 :app.account/display-name
 {:app.current-session/account [:app.account/display-name]}]
```

# Pathom

# Fulcro
