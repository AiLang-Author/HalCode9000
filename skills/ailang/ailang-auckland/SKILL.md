---
name: ailang-auckland
description: Auckland UI layout engine and simplified HTML GUI for Ailang-Self-Hosting Display system. Use whenever working on Library.Auckland*, AucklandBind, AucklandEvent, config/*.html app markup, AppHost, TextRegion draw paths, widget tags (button, textfield, canvas, etc.), or expanding the Display UI toolkit. Also use for Display/Relmem/Pgmem tasks involving Auckland inventory, layout solver, or HTML-to-widget binding — even if the user says "UI", "markup", "deskbar", or "window content".
---

# Auckland Display UI

Load **`ailang/ailang-dev`** first for general AILANG syntax (6-arg ABI, StoreValue, flattened expressions).

Persistent context: `Pgmem pickup key=auckland-inventory` or read `/home/bob/Ailang-Self-Hosting-/AUCKLAND_INVENTORY.md`.

Symbol lookup: `Relmem op=where symbol=AK_DrawNode` (index path: `Librarys/Display`).

---

## Pipeline

```
config/foo.html
  → HTMLParse (AST)
  → AucklandBind (AKParse_File / AKParse_String)
  → Auckland (AK_Solve + AK_Draw)
  → WinManager content surface
```

Entry point for apps: `AppHost_Open(html_file, action_cb)` in `Librarys/Library.AppHost.ailang`.

---

## File Map — Read Before Writing

| Task | Read first |
|------|------------|
| New widget draw | `UI/Library.Auckland.ailang` → `AK_DrawNode` |
| New HTML attr/tag | `UI/Library.AucklandBind.ailang` → `AKBind_ApplyAttr`, `AKBind_TagNameToAKTag` |
| Input/events | `UI/Library.AucklandEvent.ailang` → `AK_HitWalk`, `AK_EventMouse` |
| Text rendering | `UI/Library.TextRegion.ailang` |
| App from HTML | `Librarys/Library.AppHost.ailang` |
| Example markup | `config/settings.html`, `config/notepad.html` |

Core paths (repo root `Ailang-Self-Hosting-`):
- `Librarys/Display/UI/Library.Auckland.ailang`
- `Librarys/Display/UI/Library.AucklandBind.ailang`
- `Librarys/Display/UI/Library.AucklandEvent.ailang`

---

## AKContext Rules

- Always pass `ctx: LinkagePool.AKContext` — use `ctx@field` or accessor functions (`AK_GetDesignW`, etc.).
- Nodes: `AK_CreateNode(ctx, tag)` → `AK_AddChild(ctx, parent, child)` → `AK_SetRoot(ctx, root)`.
- Fields via `AK_Get(ctx, node, AKF.*)` / `AK_Set` and `AK_ExtraGet` / `AK_ExtraSet`.
- Render: `AK_Draw(ctx, canvas, canvas_w, canvas_h)` calls solve then draw.
- Mark dirty after dynamic changes: `AK_Set(ctx, node, AKF.DIRTY, 1)`.

---

## Widget Status (2026 baseline)

**Working:** `window`, `group`, `panel`, `button`, `label`, `separator`, `checkbox`, `textfield`, `spacer` (layout only).

**Stubbed (tag exists, no/full draw):** `scroll`, `tabs`, `tab`, `text`, `display`, `radio`, `slider`, `progress`, `image`, `canvas`, `theme`, `flow` layout.

Do not assume stubbed tags work — verify `AK_DrawNode` has an `EqualTo(tag, AKTag.X)` block.

---

## Canonical Patterns

### Bind a color attribute (AucklandBind)

```
cmp_bg = StringCompare(attr_name, "bg")
IfCondition EqualTo(cmp_bg, 0) ThenBlock: {
    bgc = AKBind_ParseColor(attr_val)
    AK_ExtraSet(ctx, ak_node, AKExtra.COLOR_BG, bgc)
    ReturnValue(0)
}
```

Colors: `#RGB`, `#RRGGBB`, `#RRGGBBAA` → `TVG_PackColor` / packed BGRA.

### Draw a filled widget (Auckland)

```
IfCondition EqualTo(tag, AKTag.BUTTON) ThenBlock: {
    bg = AK_ExtraGet(ctx, node, AKExtra.COLOR_BG)
    IfCondition EqualTo(bg, 0) ThenBlock: { bg = Theme.ak_btn_bg }
    Draw_Pix_FillRect(canvas, sx, sy, sw, sh, bg)
    // ... border, TextRegion label, state handling ...
}
```

Use `Theme.*` and `UIScale.*` defaults — not hardcoded pixels.

### Button state handling

States: `AKState.NORMAL`, `HOVER`, `PRESSED`, `DISABLED`. Set via `AK_SetState` in AucklandEvent. Lighten/darken BGRA channels for hover/press (see existing BUTTON block).

### Text label

```
tr = AK_ExtraGet(ctx, node, AKExtra.TR_HANDLE)
IfCondition LessThan(tr, 0) ThenBlock: {
    tr = TextRegion_Create(canvas, sx, sy, sw, sh)
    AK_ExtraSet(ctx, node, AKExtra.TR_HANDLE, tr)
}
TextRegion_SetRect(tr, sx, sy, sw, sh)
TextRegion_SetColor(tr, fg)
TextRegion_Render(tr, sp, sl)
```

### HTML app window root

```html
<window title="My App" design-w="640" design-h="480" toolbar="about">
  <group layout="vbox" gap="8" padding="8" bg="#0E1730">
    <button label="OK" action="app.ok" height="28"/>
  </group>
</window>
```

Toolbar modes: `none`, `about`, `file`, `full`, `browser`. Optional `addressbar="true"`.

---

## Layout

| `layout` | Status |
|----------|--------|
| `vbox` | measure + layout + grow |
| `hbox` | measure + layout + grow |
| `grid` | measure + layout (`cols` attr) |
| `flow` | parsed only — **not implemented** |

Sizing attrs in bind: `width`, `height`, `min-w`, `min-h`, `max-w`, `max-h`, `grow`, `shrink`, `gap`, `padding`, `pad-t/r/b/l`, `cols`, `align`.

**Not wired:** `justify` (constants exist in `AKJustify`).

Scale: design-space values → `AK_Scale(ctx, val)` at layout/draw time.

---

## Events & Actions

- Mouse: `AK_EventMouse(ctx, mx, my, ev)` — `AKMouseEv.MOVE/DOWN/UP`.
- Click on hittable node → `AK_FireAction` → `action` string → `ctx@action_cb` via `CallIndirect`.
- Hittable: `button`, `label`, `text`, `checkbox`, `radio`, `slider`, `canvas`, `display`.
- **Gap:** `textfield` not hittable — focus only at bind or `ctx@focus_node`.
- Keyboard: `AK_EventKey` → focused `TEXTFIELD` + `TextBuf_*`.

---

## Known Gaps (expansion backlog)

1. Draw stubs: `image`, `canvas`, `radio`, `slider`, `progress`, `text`, `display`, `scroll`, `tabs`
2. `justify` + `flow` layout
3. Per-node `font-size` (parsed, not used in draw)
4. `textfield` click-to-focus
5. No `AK_FindById` in core (FileDialog has `FD_FindNodeById`)
6. Auckland calls `SynColor_Line` without explicit `LibraryImport.Display.UI.SyntaxColor`
7. AucklandBind is compile-time HTML→tree, not live data binding

Priority: image/canvas → form controls → justify/flow → scroll/tabs → `AK_FindById`.

---

## Workflow Checklist

Before editing Auckland code:

1. `Skills read ailang/ailang-dev`
2. `Skills read ailang/ailang-auckland` (this file)
3. `Pgmem search query=auckland inventory` or read `AUCKLAND_INVENTORY.md`
4. `Relmem op=where symbol=<TargetFunction>`
5. Read 2 existing functions in the same file — match style exactly
6. After edits: `Ailang op=analyze source=<abs-path-to-entry.ailang>`

For leaf tasks via sub-agent: park context with `Pgmem op=park key=agent-input-<id>`, spawn `Agent`, `Sleep seconds=90`, `Pgmem op=task_get`.

---

## Imports Reference

```
LibraryImport.Display.UI.Auckland
LibraryImport.Display.UI.AucklandBind
LibraryImport.Display.UI.AucklandEvent
LibraryImport.Display.UI.TextRegion
LibraryImport.Display.Content.HTMLParse
LibraryImport.Display.Theme.UITheme
LibraryImport.Display.Theme.UIScale
LibraryImport.TextBuffer
```

Render deps (via Auckland): `DSurface`, `DDrawPixel`, `SurfaceBlit`, `Fonts`, `VIF`.

---

## Anti-Patterns

- Do not use C/Python control flow — use `IfCondition`/`ThenBlock`/`ElseBlock`, `WhileLoop`.
- Do not pass more than 6 args to `SystemCall`.
- Do not add `LibraryImport` without checking the link graph (SysDisplay pulls the full tree).
- Do not implement a new HTML tag without: `AKTag` (if new), `AKBind_TagNameToAKTag`, `AKBind_ApplyAttr` attrs, `AK_DrawNode` case, and optionally `AK_HitWalk` + `AK_MeasureNode`.
- Do not poll Pgmem in a loop while an Agent is running.