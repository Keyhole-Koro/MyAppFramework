# MyAppFramework — MyOS アプリケーションフレームワーク

アプリがリンクする **SDK**。`src/annotations.mln`（`@app` 等の**宣言**）、`src/protocol.mln`
（UI プロトコルのメッセージ表）、`src/ui.mln`（アプリが呼べる UI API）、`src/elements.mln`
（markup の語彙とデフォルト）、`src/runtime.mln`（ハンドラ表とイベントループ）。`ui` と
`elements` の 1 関数は 1 要求メッセージで、UI サーバ（`MyOS/src/ui/ui_server.mln`,
`elements_server.mln`）がそれに答える。`@app` の意味（起動・ウィンドウ管理）はシェル
（`MyOS/src/shell/app.mln`）、`@timer` / `@key` / `@open` / `@on_close` の意味は `runtime.mln`。
コンパイラが残したメタデータ表の読み手は MyStdLib（`meta/annotations.mln`）。
層の全体像は `docs/design/os-app-boundaries.md`、メッセージ表は `docs/design/ui-protocol.md`。
コンパイラは「`@a(x)` を宣言と照合して、モジュールの表に 1 行記録する」ことしか知らない
（`toolchain/MyLangCompiler/docs/grammar.md` "Attributes and annotations"）。`@app` の
**意味**と、いつ処理するか（ライフサイクル）は全部このリポジトリにある。

## アプリの書き方

```mylang
package counter;

import elements from "../../../MyAppFramework/src/elements.mln";        // markup の要素語彙
import ui from "../../../MyAppFramework/src/ui.mln";                    // アプリが呼べる UI API
import { app } from "../../../MyAppFramework/src/annotations.mln";      // 使うアノテーション

struct Counter {
    i32 clicks;          // インスタンスの状態。mount 時にゼロ埋めされる
    i32 step;
    i32 label;           // ref= で受け取るノード id
};

@app                                                     // view に付ける = この型はアプリ
i32 (Counter *c) view() {
    c->step = 1;                                         // 0 以外の初期値はここで
    return <Window title="Counter" x={96} y={72} w={400} h={272}>
        <Label ref={c->label} text="clicks: 0" bold={1} testId="counter" />
        <PrimaryButton text="Click me" w={120} h={36} onClick={c->click} />
    </Window>;
}

void (Counter *c) click(i32 id) {
    c->clicks = c->clicks + c->step;
    ui.set_text_fmt(c->label, "clicks: %d", c->clicks, 0);
}
```

- **アノテーションは import する宣言。** `@app` / `@timer` / `@key` / `@open` /
  `@on_close` は `annotations.mln` のプロトタイプ `export void app(i32 view, char *type, i32 size, ...);`
  等（Java の `@interface` 相当。呼ばれない）。コンパイラは使用箇所を宣言と照合して
  メタデータ表に記録し、`app.install()` が起動時に表を読んで意味を与える。
  import していない名前・引数の形が違う使い方はコンパイルエラー。
- **アプリは `boot/main.mln` が import する。** import されたモジュールがプログラムに入り、
  各モジュールのメタデータ行（`annotations` 束ねセクション）を**リンカ**が 1 本の表に
  連結する。framework はアプリを知らない。将来はアプリを
  MFS 上の .mbin にしてローダが表を読む形にする予定（main.mln の TODO）。
- **レシーバはポインタか参照** (`Counter *c` / `ref mut Counter c`)。framework はインスタンスの
  アドレスを第一引数に渡すので、値レシーバ（move）は使えない（コンパイルエラー）。
- **`ref={c->label}`** はそのノードの id をフィールドに書く。木を歩いて id を
  取り直すコードは要らない。
- **省略できるプロパティ**: `x`/`y`（親原点）、`w`/`h`、`color`、`bold`、
  `gap`、`padding`、`onClick` 等は `elements.mln` のデフォルトが入る。
  `testId="..."` は automation 用の名前。
- **ハンドラはメソッドを直接渡す** (`onClick={c->click}`)。引数は
  `()`, `(i32 id)`, `(i32 id, i32 arg)` のどれでもよい。関数ポインタはアプリの外へ出ない：
  `runtime.mln` の表に (ノード id, 種別) → メソッドとして残り、サーバからの CLICK / CHANGE
  イベントで呼ばれる。

### メソッド属性

| 属性 | 意味 | 形 |
| --- | --- | --- |
| `@app` / `@app(single)` / `@app(name = "...")` | この view を持つ型をアプリに。`single` は 2 回目の起動で既存ウィンドウを前面に | `i32 (T *self) view()` |
| `@open` | 他アプリの `ui.open(path)` を受ける。`single` なら既存インスタンスへ | `(char *path)` |
| `@on_close` | ユーザーがウィンドウを閉じた。後始末のあと framework がインスタンスを解放 | `()` / `(i32 id)` |
| `@timer(ms)` | mount 中、周期的に呼ばれる。インスタンスごとに 1 本 | `()` / `(i32 id)` |
| `@key("Ctrl+S")` | そのウィンドウがアクティブな間のショートカット | `()` / `(i32 id)` / `(id, arg)` |

`src/apps/editor.dom.mln`（`single` + `@open` + `@key`）、`terminal.dom.mln`
（`@timer` + `@key("Enter")` + `@on_close`）、`files.dom.mln`（ダイアログ）が実例。

## アプリが触れるもの

アプリが import するのは `elements.mln`（markup の解決先。コードからは呼ばない）
と `ui.mln` だけで、どちらも MyAppFramework の中。`ui` の引数・戻り値は **i32 と char\* のみ**
（無しはインデックスなら -1、id/ポインタなら 0）で、struct・Option・Node\* は跨がない。
ユーザープロセスでアプリを動かすとき、この面をそのままメッセージ／syscall にするため
（MYOS-019）。

- テキスト: `set_text`, `text_of`, `set_text_fmt(id, "%d / %s", a, b)`
  （ラベルはポインタを保持するので、書式結果はラベルごとのバッファに置かれる）
- ウィジェット: `is_checked`, `set_checked`, `input_set`, `area_append`,
  `area_clear`, `list_set_items`, `list_selected` (-1), `list_item`
- ウィンドウ: `focus`, `focus_window`, `show(win)`（自作ダイアログを載せる）,
  `close(win)`, `window_of`, `window_x/y`
- 他アプリ: `open(path)` → `@open` を持つアプリへ

## 裏側

### `@app` はメタデータ

```mylang
// annotations.mln — 宣言だけ
export void app(i32 view, char *type, i32 size, bool single = false, char *name = "");
export void timer(i32 fn, char *type, i32 size, i32 ms);
```

`@timer(100)` を `Terminal` の `poll` に付けると、コンパイラは terminal.dom.mln の
`annotations` セクションに 1 行 `["timer", Terminal__poll, "Terminal", sizeof(Terminal), 1, 100, 0, 0]`
を静的データとして出す。リンカが全モジュールの行を 1 本の `annotations` セクションに連結し、
`app.install()` が MyStdLib のイテレータで名前ごとに読む：

```mylang
Annotations apps = annotations.named("app");
while (apps.next()) {
    register_app(apps.type(), apps.size(), apps.fn(), apps.arg(0), apps.text(1));
}
```

`"app"` / `"timer"` / … を型名キーのレジストリ（`register_*`）に振り分ける。順序・検証・
いつ読むかはシェル（`MyOS/src/shell/app.mln`）が決める。他の framework が自分のアノテーションを同じ表に混ぜても、
頼んでいない名前の行は見えない。仕様は `docs/design/toolchain-collected-sections.md`。

- **ハンドラ ABI は 1 種類**: `void handler(i32 owner, i32 id, i32 arg)`。
  `owner` はインスタンスのポインタ。呼び出し規約が余分な引数を無視するので、
  `onClick={c->click}` も `@timer` のメソッドも**メソッドの実体**をそのまま登録する
  （`(Counter *c, i32 id)` に `owner` と `id` が届く）。トランポリンは無い。
- **イベント** (`docs/design/ui-protocol.md`): 所有ノードへのクリック・変更・タイマは
  `dom.emit` が `UiEvent` としてリング（`ui_events`）に積み、アプリ側の `runtime.pump()` が
  表を引いてメソッドを呼ぶ。今はコンポジタが `dom.drain_events()` の末尾で in-process の
  host（`shell/host.mln`）にリングを空にさせている（同じタスク上）。別タスク・別プロセスへ
  移すのが次の段（MYOS-020）で、その時に DOM ロックが要る。`@key` は「キーを widget に
  渡すか」を即決する必要があるので、アプリが起動時に CLAIM_KEY で組み合わせを申告し、
  シェルが照合して KEY イベントにする。
- **owner**: `Node.owner` は生成時に `dom.g_current_owner` が入る。framework は
  `view()` とハンドラの実行中それをインスタンスに設定するので、アプリが途中で
  作ったダイアログやタイマも同じ owner になり、ウィンドウを閉じると
  `dom.remove_owned(owner)` で一括回収される。
- **レジストリ** (`MyOS/src/shell/app.mln`): 型名ごとに size / view / single / name /
  `@open` の有無。`"Ctrl+S"` の解釈は `key_spec_matches`（claim 表を照合）。タイマ・キー・
  open・on_close のメソッドはシェルにはなく、アプリの `runtime.start()` が自分の行から登録する。
- **メタデータ表** (`toolchain/MyStdLib/meta/annotations.mln`): 1 行 8 ワード（名前・関数・
  型名・サイズ・引数数・引数×3）。`section.as_slice<AnnotationRow>("annotations")` で表を
  スライスとして取り、`Annotations` カーソルが名前／型でフィルタしながら歩く。
- **表の集約はリンカ**（`.section annotations` の束ね、`__sections` ディレクトリ）。`main.mln`
  に目印は無い。アプリを fs 上の .mbin にする段階では、同じディレクトリを MBIN ヘッダに載せて
  ローダが読む。

## 検証

```
make build
python3 system/MyOS/tests/app_framework_test.py   # launcher / single / @key / @open / close / dialog（make framework-test）
python3 system/MyOS/tests/dom_click_test.py       # Counter と Notes の操作
python3 system/MyOS/tests/apps_e2e_test.py        # プロセス / エディタ / ファイラ
```

## 既知の制限

- ノード id は再利用されず 256 で尽きる（`dom.mln`）。アプリの起動・終了を
  繰り返すと `dom: out of node ids` で止まる。
- `f->items[0]` のように、ポインタ経由の配列フィールドは添字できない。
  バッファはモジュール変数か heap に置く（`files.dom.mln` 参照）。
- `Result<Option<i32>, E>` を値の case で受ける (`Ok(v) -> v`) と struct が
  コピーされない。文レベルの 2 段 case で読む（`files.dom.mln` の `refresh`）。
- `@task` は無くした（`scheduler.spawn_task` がタスク引数を取れるようになったら関数を 1 つ足すだけ）。
- struct のフィールド初期化子 (`i32 step = 1;`) は効かない（生成コードが無い）。view() で入れる。
- グローバルの**ポインタ配列**への実行時代入 (`char *g[16]; g[0] = "x";`) は誤コンパイル
  される（要素ストライドの既知バグ）。シェルの `app.mln` は `i32` 配列 + キャストで持っている。
  静的初期化 (`char *g[] = {"a", "b"};`) は `.word` で正しく出る。
- アプリはまだ UI サーバと同じアドレス空間で動く（`uiproto.request` は関数呼び出し、
  イベントは 1 本のリング）。別タスク化は MYOS-020、プロセス化は MYOS-022。
