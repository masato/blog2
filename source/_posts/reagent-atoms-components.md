title: "Reagent入門 - Part2: atomとcomponentとthreading macro"
date: 2015-05-22 23:49:35
tags:
 - Reagent
 - React
 - ClojureScript
description: Reactアプリは親子関係があるcomponentで構成された木構造になります。前回は静的なcomponentを作成しましたが実際のcomponentはstateとpropsを持ちます。Reagentではatomを使いReactのstateを抽象化します。stateとpropsの厳密な区別が不要で、Reactよりも複雑なモデルをcomponentで作成することができます。
---

[React](https://facebook.github.io/react/)アプリは親子関係があるcomponentで構成された木構造になります。[Part1](/2015/05/22/reagent-tutorials-static-mock/)では静的なcomponentを作成しましたが実際のcomponentはstateとpropsを持ちます。[ClojureScript](https://github.com/clojure/clojurescript)の[Reagent](http://reagent-project.github.io/)では[Clojure](http://clojure.org/)の[atom](https://clojuredocs.org/clojure.core/atom)を拡張したatomを使いReactのstateを抽象化します。stateとpropsの厳密な区別が不要で、Reactよりも複雑なモデルをcomponentで作成することができます。

<!-- more -->

## Reactのstateとprops

Reactアプリはcomopnentで構成します。子componentの共通の親componentがstateを持ち、子componentへpropsとして渡します。stateを持つだけの親componentを作る場合もあります。親が管理をして子は使う関係になります。

### stateの特徴

* 値が変化する
* stateが変更されるとcomponentは再描画される

### propsの特徴

* immutableで値が変化しない
* 親から渡される値
* stateや他のpropsから計算される値

## atom

Reagentではstateをatomで抽象化します。

* steteを定義して値の変化を監視する
* イベントハンドラが変化を受け付けて値を更新する

### Reagent独自のatom

Reagentのatom(ratom)は通常のClojureのatomと同じように動作します。atomの値が変化があると、derefしているすべてのcomponentが自動的に再描画される点が通常のatomとは異なります。

### atomの操作

副作用の関数を使ってatomの値を更新します。副作用の関数は[reset!](https://clojuredocs.org/clojure.core/reset!)や[swap!](https://clojuredocs.org/clojure.core/swap!)のように`!`でsuffixされています。

## 使い方

以下のサイトを参考にしてatomとcomponentの使い方を見ていきます。

* [BUILDING SINGLE PAGE APPS WITH REAGENT](http://yogthos.net/posts/2014-07-15-Building-Single-Page-Apps-with-Reagent.html)
* [Step 3: Identify the minimal (but complete) representation of UI state](http://facebook.github.io/react/docs/thinking-in-react.html#step-3-identify-the-minimal-but-complete-representation-of-ui-state)
* [Functional programming on frontend with React & ClojureScript](http://blog.scalac.io/2015/04/02/clojurescript-reactjs-reagent.html)


### globalなatom

次の例ではdocument全体のstateを管理するglobalなatomを定義しています。

```clj
(def state (atom {:doc {} :saved? false}))

(defn set-value! [id value]
  (swap! state assoc :saved? false)
  (swap! state assoc-in [:doc id] value))
```

atomの値を参照(deref)する場合は、`@state`のように`@`をprefixします。

```clj
(defn get-value [id]
  (get-in @state [:doc id]))
```

### home component

home componentが一番親のcomponentになります。input、list、buttonのcomponentを子に持ちます

````clj
(defn home []
  [:div
    [:div.page-header [:h1 "Reagent Form"]]

    [text-input :first-name "First name"]
    [text-input :last-name "Last name"]
    [selection-list :favorite-drinks "Favorite drinks"
     [:coffee "Coffee"]
     [:beer "Beer"]
     [:crab-juice "Crab juice"]]

   (if (:saved? @state)
     [:p "Saved"]
     [:button {:type "submit"
              :class "btn btn-default"
              :onClick save-doc}
     "Submit"])])
```


### input component

`text-input`関数は`row`関数を定義してcomponentを作成します。`row`関数は直接実行せずベクターで定義します。関数の実行はReagentが必要なときに自動的に行います。onChangeイベントが発火されると`set-value`関数が実行されてinputフィールドの新しいの値でstateを更新します。

```clj
(defn row [label input]
  [:div.row
    [:div.col-md-2 [:label label]]
    [:div.col-md-5 input]])

(defn text-input [id label]
  [row label
   [:input
     {:type "text"
       :class "form-control"
       :value (get-value id)
       :on-change #(set-value! id (-> % .-target .-value))}]])
```

### -> threading macro

`->` スレッディングマクロは左から右に連続して次の関数の関数を実行します。[What does -> do in clojure?](http://stackoverflow.com/questions/4579226/what-does-do-in-clojure)に例があります。(+ 2 3)の結果の5が次の関数の先頭に送信されます。(- 5 7)を評価するので結果は-2になります。

```clj
(-> 2 (+ 3) (- 7))
```


### list component

comopnentの中でlocalなatomをletで作成することもできます。

```clj
(defn selection-list [id label & items]
  (let [selections (->> items (map (fn [[k]] [k false])) (into {}) atom)]    
    (fn []
      [:div.row
       [:div.col-md-2 [:span label]]
       [:div.col-md-5
        [:div.row
         (for [[k v] items]
          [list-item id k v selections])]]])))
```

`list-item`関数はli componentを作成します。onClkickイベントが発火されるとatomのselectionsに新しい値をセットします。

```clj
(defn list-item [id k v selections]
  (letfn [(handle-click! []
            (swap! selections update-in [k] not)
            (set-value! id (->> @selections
                                (filter second)
                                (map first))))]
    [:li {:class (str "list-group-item"
                      (if (k @selections) " active"))
          :on-click handle-click!}
      v]))
```

### ->> threading macro

`->>` スレッディングマクロは、`->` スレッディングマクロと評価の順番が異なります。`->`は最初に`->>`は最後に挿入されます。`(-> 2 (+ 3) (- 7))`は`-2`でしたが、`(->> 2 (+ 3) (- 7))`の場合は`2`になります。`(+ 3 2)`の結果の`5`が`(- 7 5)`のように最後に入ります。

```clj
(->> @selections
     (filter second)
     (map first))
```

`(filter second @selections)`でフィルタした結果のcollectionを`(map first coll)`します。

### localのatom

atomのselectionsは`selection-list`関数内でletを使いlocalのatomとして`->>`マクロを使い作成されています。

```clj
(defn selection-list [id label & items]
  (let [selections (->> items (map (fn [[k]] [k false])) (into {}) atom)]
...
```

itemsベクターは以下のような`[キーワード シンボル]`のベクターを要素に持ちます。ClojureScript REPLを起動して確認してみます。

```clj
cljs.user=> (def items [[:coffee "Coffee"] [:beer "Beer"] [:crab-juice "Crab juice"]])
[[:coffee "Coffee"] [:beer "Beer"] [:crab-juice "Crab juice"]]
```

`->>`マクロでitemsはmap関数の後ろの引数に入ります。`[[k]]`でベクターをdestructuringして先頭のキーワードを`k`のシンボルにバインドします。map関数ではitemの要素ごとに`[キーワード false]`の新しいベクターを返します。

```clj
cljs.user=> (def items_keys (map (fn [[k]] [k false]) items))
([:coffee false] [:beer false] [:crab-juice false])
```

map関数の結果のコレクションは`->>`マクロで次のinto関数の引数の後ろに入りmapをつくります。

```clj
cljs.user=> (def items_map (into {} items_keys))
{:coffee false, :beer false, :crab-juice false}
```

最後にatom関数の引数にmapが渡りatomを作成します。

```clj
cljs.user=> (require '[reagent.core :as reagent :refer [atom]])
nil
cljs.user=> (def selections (atom items_map))
#<Atom: {:coffee false, :beer false, :crab-juice false}>
cljs.user=> @selections
{:coffee false, :beer false, :crab-juice false}
```

`:beer`のitemがクリックされてonClickイベントが発火されると、selectionsが保持するキーワードに該当するbool値を反転させます。

```clj
cljs.user=> (swap! selections update-in [:beer] not)
{:coffee false, :beer true, :crab-juice false}
```

次の`->>`マクロを実行してクリックされた`:beer`キーワードの値をlocalのatomから取得します。

```clj
cljs.user=> (->> @selections (filter second) (map first))
(:beer)
```

globalなatomのstateはドキュメント全体の`state`を保持しています。

```clj
cljs.user=> (def state (atom {:doc {} :saved? false}))
#<Atom: {:doc {}, :saved? false}>
```

list componentの中で保持しているlocalなatomをクリックイベントによって更新したあと、globalなatomのstateを更新します。選択されたitem componentの`:beer`キーワードとlist componentの`:favorite-drinks`キーワードを使いglobalのatomを更新します。

```clj
cljs.user=> (swap! state assoc :saved? false)
{:doc {}, :saved? false}
cljs.user=> (swap! state assoc-in [:doc :favorite-drinks] :beer))
{:doc {:favorite-drinks :beer}, :saved? false}
```
