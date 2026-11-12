# MyAppFramework — MyOS アプリケーション SDK

アプリがリンクする **SDK**。アプリは 1 本の実行形式（`.mbin`、プロセス）で、リンクするのは
この SDK と MyStdLib だけ：

Framework内は、`src/*.mln`（アプリがimportする面）と`src/runtime/`（SDKの内側）に分かれる。
OSと共有するdata-only契約はFrameworkの所有物ではなく、MyComputerルートの`contracts/myapp/`に置く。

| ファイル | 役割 |
| --- | --- |
| `src/annotations.mln` | `@app` / `@timer` / `@key` の**宣言** |
| `src/app.mln` | `App` トレイトと `app.run<T>()`（状態を持ち、イベントループを回す） |
| `../../contracts/myapp/message.contract.mln` | domain付き `Request`、`Event` / `EventKind`のdata contract |
| `../../contracts/myapp/ui.contract.mln` | DOM/widgetだけの `UiOp` |
| `../../contracts/myapp/lifecycle.contract.mln` | shell/lifecycleの `AppOp` |
| `src/runtime/transport.mln` | MyStdLibのMyOS syscall binding上の `request` / `poll` / `idle` / `exit` |
| `src/ui.mln` | アプリが呼べる UI API（1 関数 = 1 要求） |
| `src/elements.mln` | markup の語彙とデフォルト（`<Window>` … = CREATE_\* 要求） |
| `src/runtime/runtime.mln` | ハンドラ表、束縛した属性、部品、`start_annotations()`（`@timer` / `@key` を登録）、`next()`（イベントを 1 つ取る）、`log()` |
| `src/terminal.mln` | プロセスを`Screen`（`<Terminal>`）に載せるUI固有連携。汎用FS/process APIはMyStdLib `hosted/` |

UI protocol endpoint（`MyOS/src/ui/protocol.mln`, `protocol_elements.mln`）が要求に答える。`@app` の
schema とプロセス内の view 起動は SDK、インストール済みアプリの発見・起動・ウィンドウ管理・
reap はシェル（`MyOS/src/shell/app.mln`）の責務。
コンパイラが残したメタデータ表の読み手は MyStdLib（`meta/annotations.mln`）：シェルは
ディスク上の各 `.mbin` のヘッダから `@app` 行を読んでランチャーに載せ、アプリのプロセスは
自分の image の行から `@timer` 等を登録する。メッセージ表は `docs/design/ui-protocol.md`。
コンパイラは「`@a(x)` を宣言と照合して、モジュールの表に 1 行記録する」ことしか知らない
（`toolchain/MyLangCompiler/docs/grammar.md` "Attributes and annotations"）。MyStdLib は row を
意味付けせず返すだけで、SDK と MyOS がそれぞれ自分の責務に必要な row を解釈する。

## アプリの書き方

```mylang
package counter;

import app from "@app/app.mln";                  // app.run と App
import { App, View } from "@app/app.mln";
import elements from "@app/elements.mln";        // markup の要素語彙
import { app } from "@app/annotations.mln";      // 使うアノテーション

@app                                             // struct に付ける = この型はアプリ
struct Counter {
    i32 clicks;          // インスタンスの状態
    i32 step;
};

impl App for Counter {
    Self start() { return Counter { step: 1 }; } // 初期値はここで

    View (Self c) view() {
        return <Window title="Counter" w={400} h={272}>
            <Label text={"clicks: %d", c.clicks} bold={1} testId="counter" />
            <PrimaryButton text="Click me" w={120} h={36} onClick={c.click} />
        </Window>;
    }
}

void (borrow Counter c) click() {
    c.clicks = c.clicks + c.step;
}

i32 main() { return app.run<Counter>(); }
```

- **アノテーションは import する宣言。** `@app` / `@timer` / `@key` は `annotations.mln` の
  プロトタイプ `export void timer(i32 ms);` 等（Java の `@interface` 相当。呼ばれない）。
  コンパイラは使用箇所を宣言と照合してメタデータ表に記録し、SDK runtime と MyOS shell が
  自分の責務に必要な行を読む。import していない名前・引数の形が違う使い方はコンパイルエラー。
- **アプリはディスク上の実行形式。** `qa/runners/build_user_apps.py` が各 `.dom.mln` を SDK と
  リンクして `.mbin` にし、mkfs が `/apps` に置く。シェルは起動時に `/apps` の各ファイルの
  ヘッダから `@app` 行を読む。Framework は特定のアプリを知らず、MyOS image にアプリの
  コードをリンクもしない。MyOS リポジトリの `src/apps/` は配布する source package の置き場で、
  MyOS 実装層ではない。
- **ハンドラのレシーバは参照** (`borrow Counter c`)。framework はインスタンスのアドレスを第一引数に
  渡すので、値レシーバ（move）は使えない（コンパイルエラー）。`view` のレシーバは読むだけ。
- **ノードの id はアプリに来ない。** 値の属性（`text={"clicks: %d", c.clicks}`）は状態を読み、
  ハンドラの後で runtime が違いだけを送る。
- **DOM が持つテキストは部品で読む**：部品は `start` で作り（`ui.field()`、`ui.screen()`）、要素に渡す
  （`field={e.body}`、`screen={t.screen}`）。`TextField` の `read()` は次の `read` まで有効な `str` を
  返す。サーバの中を指すポインタは返ってこない。
- **省略できるプロパティ**: `x`/`y`（親原点）、`w`/`h`、`color`、`bold`、
  `gap`、`padding`、`onClick` 等は `elements.mln` のデフォルトが入る。`<Window>` に `x`/`y` は
  無く、シェルが置く。`testId="..."` は automation 用の名前。
- **ハンドラはメソッドを直接渡す** (`onClick={c.click}`)。受け取るのはイベントが運ぶ値か、それより
  少ない値だけで、ノードの id は渡らない：`onClick` は `()`、`TextInput` の `onChange` は
  `(str text)`、`Checkbox` の `onChange` は `(bool on)`、`List` の `onChange` は `(i32 row)`。
  関数ポインタはアプリの外へ出ない：`runtime.mln` の表に (ノード id, 種別) → メソッドとして残り、
  サーバからの CLICK / CHANGE イベントで呼ばれる。

### 属性と `App` のメソッド

| 書くもの | 意味 |
| --- | --- |
| `@app` / `@app(single)` / `@app(name = "...")` / `@app(opens)` | `struct` に付けて型をアプリに。`single` は 2 回目の起動で既存ウィンドウを前面に、`opens` は他アプリの `ui.open(path)` を受ける |
| `open(str path)` | `opens` のアプリが受けるパス。起動直後は `""` で呼ばれる |
| `can_close()` | ユーザーがウィンドウを閉じようとした。false ならウィンドウは残る（保存していない変更を先に尋ねるなど） |
| `close()` | ユーザーがウィンドウを閉じた。後始末のあと runtime が EXIT を送り、MyOS が所有物を回収する |
| `app.quit()` | ハンドラの中からアプリを終える。`close` は呼ばれない |
| `@timer(ms)` | メソッドに付ける。周期的に呼ばれる |
| `@key("Ctrl+S")` | メソッドに付ける。そのウィンドウがアクティブな間のショートカット |

`src/apps/editor.dom.mln`（`single` + `opens` + `@key` + `<Menu>` + 中に部品のある `Dialog` + `can_close`）、
`terminal.dom.mln`（`<Terminal>` + `@timer` + `app.quit`）、`files.dom.mln`（`List` と `Dialog`）が実例。

## アプリが触れるもの

アプリが import する SDK API は MyAppFramework の `src/*.mln`（`elements.mln`, `ui.mln`,
`annotations.mln`, `terminal.mln`）だけで、`src/runtime/`やwire contractは直接importしない。
filesystemや一般process操作はMyStdLib `hosted/` を直接importし、FSの共有error型は
`contracts/io/fs.contract.mln`からimportする。`ui` の引数・戻り値は
wire 上では **i32 と char\*** を使う。
通常のscalar値に加え、意味上の失敗や不在を区別するAPIはdata-only contractの
`Result` / `Option`をcopy-outする。`Node*`などserver内部のポインタは跨がない。
この面は既にメッセージ／syscall の client stub として実装されている。

- テキスト: `TextField`（`read`, `write`, `append`, `clear`, `focus`）。`<TextArea>` は `wrap={true}` で
  長い行を幅で折り返し、`onEdit` は中身を読まずに変わったことだけを、`onCaret` はキャレットの行と列
  （1 から）を受ける
- メニュー: `<Menu title="File" items="New\nOpen..." onSelect={...}>` はメニューバーのボタン。押すと
  項目のメニューが下に開き、選んだ項目の番号が `onSelect` に来る。項目の中のタブで、ショートカットの
  列をそろえられる
- ウィンドウの大きさへの追従: `anchor={ui.GROW_W | ui.GROW_H}` で一緒に伸び縮みし、`ui.PIN_BOTTOM` で
  下端からの距離を保つ（`Box`, `Label`, `TextArea`, `Terminal`）。`<Window title={...}>` は状態に
  ついて変わる
  ラベルの文字は `text={...}` の属性が状態から更新する
- 端末: `<Terminal screen={...}>` は等幅の文字の格子。`terminal.spawn_bound(path, screen)` で
  プロセスを載せると、その出力と、そのプロセスが起こしたプロセスの出力がここに出る。打った行は
  最後に載ったプロセスの stdin へ行く。`Screen.write` はプロセスの出力と同じく書く
- リストとダイアログ: `<List items={...} selected={...}>`、`<Dialog when={...}>`。項目も選択も
  出し入れも値で渡す。行の中のタブで列がそろう（n 個目のタブは行頭から 20px × n の位置へ移る）ので、
  見出しの `<Label>` と行に同じ数のタブを入れれば表になる。`<Dialog>` が出ている間、アプリの
  ウィンドウはクリックを受けない。Esc は `onCancel`、Enter は `onConfirm` になる。`other=` で 3 つ目の
  答え（`onOther`）、`w=` と `h=` で大きさを決め、中に書いた要素は問いとボタンの間に入る
- グラフ: `<Graph fill={...} line={...} samples={60}>`。0〜100 の百分率を空白区切りで古い順に渡す。
  `fill` は塗りつぶし、`line` はその上の 2 本目の線（色は `ui.GRAPH_SECOND`）。新しい値が右端
- 他アプリ: `ui.open(path)` → `opens` を持つアプリへ

## 裏側

### `@app` はメタデータ

```mylang
// annotations.mln — 宣言だけ
export void app(bool single = false, char *name = "", bool opens = false);
export void timer(i32 ms);
```

`@timer(100)` を `Terminal` の `poll` に付けると、コンパイラは terminal.dom.mln の
`annotations` セクションに 1 行 `["timer", Terminal__poll, "Terminal", 1, 100, 0, 0]`
を静的データとして出す。リンカがその実行形式の全モジュールの行を 1 本の
`annotations` セクションに連結する。読み手は 2 つある：

- SDK の `runtime/runtime.mln` は、実行中 image の `@app` row から型名を取り、同じ receiver type の
  `@timer` / `@key` を登録する。インスタンスは `app.run<T>()` が `T::start()` で作る。
- MyOS の `shell/app.mln` は、`annotations.in_image(image)` でディスク上の MBIN を読み、
  `@app` の name / single / opens だけを installed-app registry に入れる。関数ポインタは
  読まず、アプリコードもロードしない。

MyStdLib は名前による filter と row access を提供するだけで、上記の意味を知らない。他の
framework が独自 annotation を同じ表に混ぜても、各読み手は要求した名前だけを見る。仕様は
`docs/design/toolchain-collected-sections.md`。

- **ハンドラ**: `owner` はインスタンスのアドレスで、続く引数はイベントが運ぶ値（無し、`bool`、
  `str`、`i32`）。`onClick={c.click}` も `@timer` のメソッドも**メソッドの実体**をそのまま登録する。
  トランポリンは無い。
- **イベント** (`docs/design/ui-protocol.md`): 所有ノードへのクリック・変更・タイマは
  UI domainのevent sinkからservice層を通って`Event`としてプロセスのチャネルに積まれ、
  `app.run` が `runtime.next()` で取って表を引きメソッドを呼ぶ。要求は `OS_CALL`
  syscallでMyOSへ入り、`ipc/gateway.mln` がユーザメモリをコピーしてservice taskの
  チャネルへ渡し、返事が来るまで
  アプリは sticky notification の `WAIT_NOTIFICATION` で待つ。DOM を触るのはサーバのタスクだけ（automation は `dom.lock`
  を取って読む）。`@key` は「キーを widget に渡すか」を即決する必要があるので、アプリが
  起動時に CLAIM_KEY で組み合わせを申告し、シェルが照合して KEY イベントにする。
- **owner はプロセス**: 要求の `owner` はカーネルが pid に書き換える。`Node.owner` には
  それが入るので、アプリが途中で作ったダイアログやタイマも同じ owner になり、ウィンドウを
  閉じる → CLOSE → アプリが `close` を走らせて EXIT → シェルが `dom.remove_owned(pid)`
  で一括回収し、プロセスは終了する。
- **レジストリ** (`MyOS/src/shell/app.mln`): ディスク上の実行形式ごとに path / type /
  single / name / opens（起動時に全 `.mbin` のヘッダから）。`"Ctrl+S"` の解釈は
  `key_spec_matches`（claim 表を照合）。タイマとキーのメソッドはシェルにはなく、アプリの
  `runtime.start_annotations()` が自分の行から登録する。
- **ウィンドウの配置** (`MyOS/src/shell/app.mln`): 新しいウィンドウは一段ずつ右下へずれ、タイトル
  バーが残る。六段で最初に戻る。
- **メタデータ表** (`toolchain/MyStdLib/meta/annotations.mln`): 1 行 7 ワード（名前・関数・
  型名・引数数・引数×3）。`section.as_slice<AnnotationRow>("annotations")` で表を
  スライスとして取り、`Annotations` カーソルが名前／型でフィルタしながら歩く。
- **表の集約はリンカ**（`.section annotations` の束ね、`__sections` ディレクトリ）。`main.mln`
  に目印は無い。アプリを fs 上の .mbin にする段階では、同じディレクトリを MBIN ヘッダに載せて
  ローダが読む。

## 検証

```
make build                                        # アプリは build/user/*.mbin → disk.img
python3 system/MyOS/tests/app_framework_test.py   # launcher / single / @key / opens / close / dialog（make framework-test）
python3 system/MyOS/tests/dom_click_test.py       # Editor のメニュー、入力、キャレット
python3 system/MyOS/tests/apps_e2e_test.py        # プロセス / エディタ / ファイラ
```

## 既知の制限

- 1 アプリ（プロセス）が持てるノードは 96 まで（`dom.NODES_PER_OWNER`）。超えると作成は
  `UiError::OwnerNodeLimit` になり、markup はプロセスを終える。id は閉じたウィンドウのものから
  再利用される。
- `f->items[0]` のように、ポインタ経由の配列フィールドは添字できない。
  バッファはモジュール変数か heap に置く。
- `@task` は無くした（`scheduler.spawn_task` がタスク引数を取れるようになったら関数を 1 つ足すだけ）。
- struct のフィールド初期化子 (`i32 step = 1;`) は効かない（生成コードが無い）。`start` で入れる。
- グローバルの**ポインタ配列**への実行時代入 (`char *g[16]; g[0] = "x";`) は誤コンパイル
  される（要素ストライドの既知バグ）。シェルの `app.mln` は `i32` 配列 + キャストで持っている。
  静的初期化 (`char *g[] = {"a", "b"};`) は `.word` で正しく出る。
- 1 プロセス 1 インスタンス。状態は `app.run` のスタックにある。`@app(single)` は
  シェルが 2 回目の起動を既存プロセスへ向けることで実現する。
- プロセスのスタックは 16 KiB。大きな値は DOM（`TextField`）か `app_allocator` のヒープに置く。
- 文字列の上限：要求の s0 は 1024 バイト（`items=`）、s1 は 128、TEXT_COPY の戻りは 4096、
  イベントの文字列（OPEN のパス）は 64。
