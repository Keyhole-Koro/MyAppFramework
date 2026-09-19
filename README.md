# MyAppFramework

Application framework for [MyOS](https://github.com/Keyhole-Koro/MyOS), written
in MyLang. Lives at `system/MyAppFramework` in
[MyComputer](https://github.com/Keyhole-Koro/MyComputer).

| file | what it is |
| --- | --- |
| `src/annotations.mln` | `@app`, `@timer`, `@key`, `@open`, `@on_close` -- declared as prototypes `(i32 fn, char *type, i32 size, ...)`, like a Java `@interface`; the compiler records each use as a metadata row |
| `src/meta.mln` | reads the metadata rows the linker collected (`__annotations_start..end`): `count()`, `name(i)`, `fn(i)`, `type(i)`, `size(i)`, `arg(i, k)` |
| `src/app.mln` | registry of installed apps, instances, launch / single-instance / `ui.open()` routing, timers, shortcuts, window close and owner sweep |
| `src/ui.mln` | the UI API an app talks to: i32 and `char*` only, so it can become a syscall surface |
| `docs/APP_FRAMEWORK.md` | how to write an app, and how the framework runs it |

An app:

```mylang
import dom_elements from "../ui/dom/dom_elements.mln";
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
annotations"). At boot `app.install()` reads every row through `meta.mln`
and decides what it means -- the lifecycle is the framework's. The linker
gathers every module's rows (`docs/design/toolchain-collected-sections.md`);
MyOS's `boot/main.mln` only imports the apps so they are part of the program.

Tests: `make framework-test` in MyComputer.
