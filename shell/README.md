# HalCode GTK shell (phase 1)

Thin native chrome. The kernel (`HalCode9000.x`) will own AppDesk MDI and
write a CAD-style framebuffer. This process only blits.

```
make -C shell
make -C shell run
```

State dir (`HALCODE_APP_STATE`, default `/tmp/halcode_app`):

| File | Meaning |
|---|---|
| `meta.bin` | three `int32`: width, height, pitch |
| `frame.raw` | BGRA, `pitch * height` |
| `gen.txt` | integer; shell reloads when it changes |

Input will be `@halcode/shell` JSON. Not `cmd.txt`.
