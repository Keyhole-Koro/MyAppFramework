# MyAppFramework

Application framework for [MyOS](https://github.com/Keyhole-Koro/MyOS), written
in MyLang. Lives at `system/MyAppFramework` in
[MyComputer](https://github.com/Keyhole-Koro/MyComputer).

| file | what it is |
| --- | --- |
| `src/annotations.mln` | `@app`, `@timer`, `@key`, `@open`, `@on_close`, `@task` -- declared as MyLang `annotation`s; `@app`'s template is what a struct expands into |
| `src/app.mln` | registry of installed apps, instances, launch / single-instance / `ui.open()` routing, timers, shortcuts, window close and owner sweep |
| `src/ui.mln` | the UI API an app talks to: i32 and `char*` only, so it can become a syscall surface |
| `docs/APP_FRAMEWORK.md` | how to write an app, and how the framework runs it |

An app:

```mylang
import dom_elements from "../ui/dom/dom_elements.mln";
import ui from "../../../MyAppFramework/src/ui.mln";
import { app, timer } from "../../../MyAppFramework/src/annotations.mln";

@app
struct Counter { i32 clicks = 0; i32 label; };

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

The compiler knows nothing about apps: it checks each `@name` against its
`annotation` declaration and expands the template
(`toolchain/MyLangCompiler/docs/grammar.md`, "Attributes and annotations").
The build (`qa/runners/gen_app_manifest.py`) lists every `@app` struct under
`system/MyOS/src/apps` and boot calls `app.install()`.

Tests: `make framework-test` in MyComputer.
