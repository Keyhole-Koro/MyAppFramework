# MyAppFramework — MyOS アプリケーションフレームワーク

アプリがリンクする **SDK**。アプリは 1 本の実行形式（`.mbin`、プロセス）で、リンクするのは
この SDK と MyStdLib だけ：

import する側で 3 つに分かれる：`src/*.mln` はアプリが import する面、`src/protocol/` は OS と
共有する契約（MyOS が import するのはここだけ）、`src/os/` と `src/runtime/` は SDK の内側。

| ファイル | 役割 |
| --- | --- |
| `src/annotations.mln` | `@app` / `@timer` / `@key` / `@open` / `@on_close` の**宣言** |
| `src/protocol/ui.mln` | UI プロトコル：`UiMsg` / `UiEvent`、要求とイベントの表、運び手の宣言 |
| `src/protocol/services.mln` | `OS_CALL` のサービス番号（`OsService`）とエラーコード |
| `src/os/syscall.masm`, `src/os/uiproto.mln` | syscall スタブと、その上の運び手の実装：`request` / `poll` / `idle` / `exit` |
| `src/ui.mln` | アプリが呼べる UI API（1 関数 = 1 要求） |
| `src/elements.mln` | markup の語彙とデフォルト（`<Window>` … = CREATE_\* 要求） |
| `src/runtime/runtime.mln` | ハンドラ表、`start()`（`@app` の view を呼び、`@timer` / `@key` / `@open` / `@on_close` を登録）、`run()`（イベントループ）、`log()` |
| `src/runtime/app_main.mln` | プロセスのエントリ：この image の `@app` 行を見つけて `start` → `run` |
| `src/fs.mln`, `src/console.mln` | ファイルと他プロセス（`OS_CALL` の `FS_*` / `PROC_*`） |

UI サーバ（`MyOS/src/ui/ui_server.mln`, `elements_server.mln`）が要求に答え、`@app` の意味
（インストール・起動・ウィンドウ管理・終了）はシェル（`MyOS/src/shell/app.mln`）。
コンパイラが残したメタデータ表の読み手は MyStdLib（`meta/annotations.mln`）：シェルは
ディスク上の各 `.mbin` のヘッダから `@app` 行を読んでランチャーに載せ、アプリのプロセスは
自分の image の行から `@timer` 等を登録する。層の全体像は `docs/design/os-app-boundaries.md`、
メッセージ表は `docs/design/ui-protocol.md`。
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
- **アプリはディスク上の実行形式。** `qa/runners/build_user_apps.py` が各 `.dom.mln` を SDK と
  リンクして `.mbin` にし、mkfs が `/apps` に置く。シェルは起動時に `/apps` の各ファイルの
  ヘッダから `@app` 行を読む。framework も image もアプリを知らない。将来はアプリを
  MFS 上の .mbin にしてローダが表を読む形にする予定（main.mln の TODO）。
- **レシーバはポインタか参照** (`Counter *c` / `ref mut Counter c`)。framework はインスタンスの
  アドレスを第一引数に渡すので、値レシーバ（move）は使えない（コンパイルエラー）。
- **`ref={c->label}`** はそのノードの id をフィールドに書く。木を歩いて id を
  取り直すコードは要らない。
- **テキストはアプリのバッファへ**：`ui.text_copy(id, out, cap)`。サーバの中を指す
  ポインタは返ってこない（`char g_buf[...]` をモジュール変数に置く）。
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

- テキスト: `set_text`, `text_copy(id, out, cap)`, `set_text_fmt(id, "%d / %s", a, b)`
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
  `dom.emit` が `UiEvent` としてそのプロセスのチャネルに積み、アプリ側の `runtime.run()` が
  `poll` で取って表を引きメソッドを呼ぶ。要求は `OS_CALL` syscall でカーネルへ、カーネルの
  `os_calls` がユーザメモリをコピーして UI サーバのタスクのチャネルへ渡し、返事が来るまで
  アプリは `sys_yield` で待つ。DOM を触るのはサーバのタスクだけ（automation は `dom.lock`
  を取って読む）。`@key` は「キーを widget に渡すか」を即決する必要があるので、アプリが
  起動時に CLAIM_KEY で組み合わせを申告し、シェルが照合して KEY イベントにする。
- **owner はプロセス**: 要求の `owner` はカーネルが pid に書き換える。`Node.owner` には
  それが入るので、アプリが途中で作ったダイアログやタイマも同じ owner になり、ウィンドウを
  閉じる → CLOSE → アプリが `@on_close` を走らせて EXIT → シェルが `dom.remove_owned(pid)`
  で一括回収し、プロセスは終了する。
- **レジストリ** (`MyOS/src/shell/app.mln`): ディスク上の実行形式ごとに path / type /
  single / name / `@open` の有無（起動時に全 `.mbin` のヘッダから）。`"Ctrl+S"` の解釈は
  `key_spec_matches`（claim 表を照合）。タイマ・キー・open・on_close のメソッドはシェルには
  なく、アプリの `runtime.start()` が自分の行から登録する。
- **メタデータ表** (`toolchain/MyStdLib/meta/annotations.mln`): 1 行 8 ワード（名前・関数・
  型名・サイズ・引数数・引数×3）。`section.as_slice<AnnotationRow>("annotations")` で表を
  スライスとして取り、`Annotations` カーソルが名前／型でフィルタしながら歩く。
- **表の集約はリンカ**（`.section annotations` の束ね、`__sections` ディレクトリ）。`main.mln`
  に目印は無い。アプリを fs 上の .mbin にする段階では、同じディレクトリを MBIN ヘッダに載せて
  ローダが読む。

## 検証

```
make build                                        # アプリは build/user/*.mbin → disk.img
python3 system/MyOS/tests/app_framework_test.py   # launcher / single / @key / @open / close / dialog（make framework-test）
python3 system/MyOS/tests/dom_click_test.py       # Counter と Notes の操作
python3 system/MyOS/tests/apps_e2e_test.py        # プロセス / エディタ / ファイラ
```

## 既知の制限

- 1 アプリ（プロセス）が持てるノードは 96 まで（`dom.NODES_PER_OWNER`）。超えると要素は
  id 0 で返る（描かれない）。id は閉じたウィンドウのものから再利用される。
- `f->items[0]` のように、ポインタ経由の配列フィールドは添字できない。
  バッファはモジュール変数か heap に置く（`files.dom.mln` 参照）。
- `Result<Option<i32>, E>` を値の case で受ける (`Ok(v) -> v`) と struct が
  コピーされない。文レベルの 2 段 case で読む（`files.dom.mln` の `refresh`）。
- `@task` は無くした（`scheduler.spawn_task` がタスク引数を取れるようになったら関数を 1 つ足すだけ）。
- struct のフィールド初期化子 (`i32 step = 1;`) は効かない（生成コードが無い）。view() で入れる。
- グローバルの**ポインタ配列**への実行時代入 (`char *g[16]; g[0] = "x";`) は誤コンパイル
  される（要素ストライドの既知バグ）。シェルの `app.mln` は `i32` 配列 + キャストで持っている。
  静的初期化 (`char *g[] = {"a", "b"};`) は `.word` で正しく出る。
- 1 プロセス 1 インスタンス（`app_main.mln` が `sizeof` の分だけ `sbrk` で取る）。`@app(single)` は
  シェルが 2 回目の起動を既存プロセスへ向けることで実現する。
- プロセスのスタックは 16 KiB、ヒープ（`sbrk`）は SDK からは使っていない。大きなバッファは
  モジュール変数に置く（editor の `g_save_text[4096]`）。
- 文字列の上限：要求の s0 は 1024 バイト（`list_set_items`）、s1 は 128、TEXT_COPY の
  戻りは 4096、イベントの文字列（OPEN のパス）は 64。
