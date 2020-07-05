# eql-style-guide

> CONTRIBUTING: Feel free to open a issue or PR ;)

Common patterns to develop [EQL](https://edn-query-language.org/) API's.

This guide will cover both `EQL` as a abstract API and common libraries usage, like [fulcro](https://github.com/fulcrologic/fulcro) and [pathom](https://github.com/wilkerlucio/pathom)

# EQL

## Keep it flat

If you have `1..1` relationship, it should be accessible without a `join`

```clojure
;; good: a session has a account, so I can access account attributes directly
[:app.current-session/token
 :app.account/display-name]

;; good: it's OK to have a explicit relationship attribute between them
[:app.current-session/token
 :app.account/display-name
 {:app.current-session/account [:app.account/display-name]}]
```

## Global namespaces

Avoid use the same name both in "global" and "contextual"

```clojure
;; good: there is `:app.authed-account/email` for global and `:app.account/email` to local
[:app.authed-account/email
 {:app.authed-account/account [:app.account/email]}]

;; avoid: `:app.session/email` is used both for global and local context.
[:app.account/email
 {:app.authed-account/account [:app.account/email]}]
```

# Pathom

## Group by output

Prefer to group your resolvers by output

```clojure
;; good: Group by output
(ns ...authed-account)

(pc/defresolver email [env input]
 {::pc/input  #{:app.current-session/id}
  ::pc/output [:app.authed-account/email]}
  ...)

;; avoid: Group by input
(ns ...current-session)

(pc/defresolver authed-account-email [env input]
 {::pc/input  #{:app.current-session/id}
  ::pc/output [:app.authed-account/email]}
  ...)
```

## Resolve it flat

If it's a `1..1` relationship, keep it flat. See [Keep it flat](#keep-it-flat) from EQL

```clojure
;; good
(pc/defresolver main-address [env input]
 {::pc/input  #{:app.account/id}
  ::pc/output [:app.address/street]}
  ...)
```

Once there is `placeholders`, clients can as `[:app.account/id {:>/main-address [:app.address/street]}]` if it needs

## Trust in qualified keys

With [spec](https://clojure.org/about/spec) in mind, you should trust your input

```clojure
;; good
(pc/defresolver main-address [env {:app.account/keys [id]}]
 {::pc/input  #{:app.account/id}
  ::pc/output [:app.address/street]}
  (db/main-address env id))
  
;; avoid
(pc/defresolver main-address [env {:app.account/keys [id]}]
 {::pc/input  #{:app.account/id}
  ::pc/output [:app.address/street]}
  (db/main-address env (UUID/fromString (str id)))
```

## Explicit transformations

If client send a thing and you need to transform, name it!

```clojure
;; good :app.account/id-as-str will always be a string, :app.account/id will always be a uuid
(pc/defresolver id-as-ast->id [env {:app.account/keys [id-as-str]}]
 {::pc/input  #{:app.account/id-as-str}
  ::pc/output [:app.address/id]}
  {:app.address/id (UUID/fromString id-as-str)})
    
;; avoid: `:app.account/id` can both be both a `string?` or a `uuid?`
(pc/defresolver main-address [env {:app.account/keys [id]}]
 {::pc/input  #{:app.account/id}
  ::pc/output [:app.address/street]}
  (db/main-address env (UUID/fromString (str id)))
```

## DRY (Don't Repeat Yourself)

If your resolvers follow a similar process to resolve the data, make sure that you factor it away in a separate function that will exist only once in your codebase. You can use the [::pc/transform key](https://wilkerlucio.github.io/pathom/#connect-transform) for this purpose.

## Security

> Please join [this issue](https://github.com/souenzzo/eql-style-guide/issues/4) and share your problems/solutions with us.

Be careful using attribute-based authentication with `p/ident-reader` and `p/open-ident-reader`. Consider the following case:

```clojure
(pc/defresolver authed-account [env _]
  {::pc/input  #{}
   ::pc/output [:app.authed-account/email]}
  (when-let [email (some-auth-logic env)]
    {:app.authed-account/email email}))

(pc/defresolver sensitive-data [{:keys [parser] :as env}
                                {:app.authed-account/keys [email]}]
  {;; (1) do NOT require input for auth
   ::pc/input  #{:app.authed-account/email}
   ::pc/output [:sensitive-data]}
  (when-let [{:app.authed-account/keys [email]}
             ;; (2) do NOT call parser with current env for auth attr
             (seq (parser env [:app.authed-account/email]))]
    {:sensitive-data [,,,]}))
```

In this case, sensitive data can be retrieved by providing context in the query:
```clojure
[{[:app.authed-account/email "admin@email.com"]
  [:sensitive-data]}]
```

Instead, prefer either calling parser without entity:
```clojure
(let [entity-key (or (::p/entity-key env) ::p/entity)]
  (parser (dissoc env entity-key) [:app.authed-account/email]))
```

Or add a parser, closured against the initial value of the env, to the env:
```clojure
(defn make-parser []
  (let [parser (p/parser {,,,})]
    (fn wrapped-parser [env tx]
      (parser
        (assoc env :parser* (partial wrapped-parser env))
        tx))))
;; then inside resolvers
(let [{:keys [parser*]} env]
  (parser* [:app.authed-account/email]))
```

# Fulcro

## Use flat keys

See [Keep it flat](#keep-it-flat) from EQL

```clojure
;; good
(defsc MainView [this {:account/keys [name]
                       :session/keys [valid? account] :as props}]
       {:ident (fn [] [:component/id :session])
        :query [:session/valid?
                :account/name
                {:session/account (comp/get-query AccountProjectList)}]
        :initial-state {}}
       (div
         (div (str "Hello, " name "!"))
         (ui-account-project-list account)))

;; bad: the MainView depends on AccountProjectList query/data
(defsc MainView [this {:session/keys [valid? account] :as props}]
       {:ident (fn [] [:component/id :session])
        :query [:session/valid?
                {:session/account (conj (comp/get-query AccountProjectList) :account/name)}]
        :initial-state {}}
       (div
         (div (str "Hello, " (:account/name account) "!"))
         (ui-account-project-list account)))
```

