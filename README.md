# MyAppFramework

The SDK a [MyOS](https://github.com/Keyhole-Koro/MyOS) application links
against, written in MyLang. Lives at `system/MyAppFramework` in
[MyComputer](https://github.com/Keyhole-Koro/MyComputer). It imports nothing
from MyOS or MyKernel: every file is an interface (prototypes), implemented on
the OS side -- the UI server for `ui` / `elements`, the shell
(`MyOS/src/shell/app.mln`) for what `@app` means. See
`docs/design/os-app-boundaries.md` in MyComputer for the layers.

| file | what it is |
| --- | --- |
| `src/annotations.mln` | `@app`, `@timer`, `@key`, `@open`, `@on_close` -- declared as prototypes `(i32 fn, char *type, i32 size, ...)`, like a Java `@interface`; the compiler records each use as a metadata row |
| `src/ui.mln` | the UI API an app talks to: i32 and `char*` only, so it can become a message / syscall surface. Prototypes only; the UI server implements it (`MyOS/src/ui/ui_server.mln`) |
| `src/elements.mln` | the markup vocabulary (`<Window>`, `<Label>`, ...) with its defaults. Prototypes only; implemented by `MyOS/src/ui/elements.mln` |
| `docs/APP_FRAMEWORK.md` | how to write an app, and how the framework runs it |

An app:

```mylang
import elements from "../../../MyAppFramework/src/elements.mln";
import ui from "../../../MyAppFramework/src/ui.mln";
import { app, timer } from "../../../MyAppFramework/src/annotations.mln";

struct Counter { i32 clicks; i32 label; };

@app
i32 (Counter *c) view() {
    return <Window title="Counter" w={400} h={272}>
        <Label ref={c->label} text="clicks: 0" bold={1} />
        <PrimaryButton text="Click me" w={120} h={36} onClick={c->click} />
    </Window>;
}

void (Counter *c) click(i32 id) {
    c->clicks = c->clicks + 1;
    ui.set_text_fmt(c->label, "clicks: %d", c->clicks, 0);
}
```

The compiler knows nothing about apps: `@app` on `view` becomes the row
`["app", Counter__view, "Counter", sizeof(Counter), ...]` in the module's
metadata table (`toolchain/MyLangCompiler/docs/grammar.md`, "Attributes and
annotations"). At boot `app.install()` walks the rows with MyStdLib's
iterator (`annotations.named("app")`, `it.next()`, `it.fn()`, ...;
`toolchain/MyStdLib/meta/annotations.mln`) and decides what they mean -- the
lifecycle is the framework's. The linker lays every module's rows out as one
section (`docs/design/toolchain-collected-sections.md`); MyOS's
`boot/main.mln` only imports the apps so they are part of the program.

Tests: `make framework-test` in MyComputer.
