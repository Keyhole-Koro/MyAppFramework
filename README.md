# MyAppFramework

The SDK a [MyOS](https://github.com/Keyhole-Koro/MyOS) application links
against, written in MyLang. Lives at `system/MyAppFramework` in
[MyComputer](https://github.com/Keyhole-Koro/MyComputer). An app is a
process -- an MBIN executable on the disk, built from its `.dom.mln` plus
this SDK -- and this is everything it links: the SDK imports nothing from
MyOS or MyKernel. It reaches the OS only through the `OS_CALL` syscall
(the UI protocol, files, other processes), and the OS talks back only
through events. See `docs/design/os-app-boundaries.md` in MyComputer for
the layers and `docs/design/ui-protocol.md` for the message table.

Two kinds of framework file, by who imports them:

- `src/*.mln` -- the **app-facing API**: what a `.dom.mln` imports
- `src/runtime/` -- SDK internals: request transport and the app's event loop

The shared application contract lives at MyComputer's root `contracts/myapp/`;
both this SDK and MyOS consume it, and neither owns it. MyAppFramework never
imports MyOS or MyKernel, and MyOS never imports MyAppFramework. Application
sources import the app-facing files directly under `src/`, never `src/runtime/`
or the wire contracts directly.
Generic filesystem, process and logging APIs live in MyStdLib's `hosted/`
layer; shared service identifiers and filesystem semantics live under
`contracts/`.

| file | what it is |
| --- | --- |
| `src/annotations.mln` | `@app`, `@timer`, `@key`, `@open`, `@on_close` -- declared as prototypes `(i32 fn, char *type, i32 size, ...)`, like a Java `@interface`; the compiler records each use as a metadata row |
| `../../contracts/myapp/message.contract.mln` | domain-tagged `Request`, `Event` / `EventKind` data contract |
| `../../contracts/myapp/ui.contract.mln` | UI-only operation table (`UiOp`) |
| `../../contracts/myapp/lifecycle.contract.mln` | app-host operations (`AppOp`: open, key claim, main window, exit) |
| `src/ui.mln` | the UI API an app talks to; the MyOS UI protocol endpoint answers it |
| `src/elements.mln` | the markup vocabulary (`<Window>`, `<Label>`, ...) and handler registration |
| `src/runtime/transport.mln` | `Request` / `Event` carriers on top of MyStdLib's MyOS syscall binding |
| `src/runtime/runtime.mln` | the app side: handler table, `start()` (runs the @app view, sets up @timer / @key / @open / @on_close from the app's own annotation rows), `run()` (the event loop), `log()` |
| `src/runtime/app_main.mln` | the process entry: finds the @app row in the image, starts the instance, runs the loop |
| `src/terminal.mln` | UI-specific child stdout ↔ TextArea binding; generic process control is MyStdLib |
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
annotations"). MyStdLib only exposes the rows as data. The SDK runtime reads
the current process's rows to start its view and handlers; MyOS reads the same
rows from `/apps/*.mbin` to discover, launch, and reap installed applications.
The linker lays each executable's rows out as one section
(`docs/design/toolchain-collected-sections.md`).

Tests: `make qa-boundaries` and `make framework-test` in MyComputer.
