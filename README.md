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

Three kinds of file, by who imports them:

- `src/*.mln` -- the **app-facing API**: what a `.dom.mln` imports
- `src/protocol/` -- the **contract** shared with the OS: the only files MyOS imports from here
- `src/os/`, `src/runtime/` -- SDK internals: how a request reaches the OS, and the app's event loop

| file | what it is |
| --- | --- |
| `src/annotations.mln` | `@app`, `@timer`, `@key`, `@open`, `@on_close` -- declared as prototypes `(i32 fn, char *type, i32 size, ...)`, like a Java `@interface`; the compiler records each use as a metadata row |
| `src/protocol/ui.mln` | the UI protocol: `UiMsg` / `UiEvent`, the op and event tables (`docs/design/ui-protocol.md`) and the prototypes of its carriers; both sides import it |
| `src/protocol/services.mln` | the `OS_CALL` service numbers (`OsService`) and error codes the OS handler (`MyOS/src/proc/os_calls.mln`) shares |
| `src/ui.mln` | the UI API an app talks to: i32 and `char*` only. Each function is one request; the UI server answers it (`MyOS/src/ui/ui_server.mln`) |
| `src/elements.mln` | the markup vocabulary (`<Window>`, `<Label>`, ...) with its defaults; each is one CREATE_* request (`MyOS/src/ui/elements_server.mln`). Handlers stay in the app |
| `src/os/syscall.masm`, `src/os/uiproto.mln` | the syscall stubs (`os_call`, `sys_yield`, `sys_exit`, `sys_sbrk`) and the UI carriers on top of them (`request` / `poll` / `idle` / `exit`) |
| `src/runtime/runtime.mln` | the app side: handler table, `start()` (runs the @app view, sets up @timer / @key / @open / @on_close from the app's own annotation rows), `run()` (the event loop), `log()` |
| `src/runtime/app_main.mln` | the process entry: finds the @app row in the image, starts the instance, runs the loop |
| `src/fs.mln`, `src/console.mln` | files and other processes, as `FS_*` / `PROC_*` services |
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
