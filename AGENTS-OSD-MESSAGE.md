# Agent prompt: finishing the OSD transient message

Continue the work started in `OSD: WIP: Non-blocking transient message overlay`.

## What exists

`osd_core_show_message(const char *text, float seconds)` posts a message that
draws itself for the given duration and then disappears. It is rendered by
`osd_core_draw_indicators()` in `src/osd/osd_core.cpp` as a borderless,
click-through ImGui window pinned to the top left, fading out over its final
second. State is a single mutex-guarded slot (`message_text`, `message_left`),
so any thread may post.

The overlay is non-blocking in both senses:

* `ImGuiWindowFlags_NoInputs | NoNav | NoFocusOnAppearing` — the overlay never
  takes mouse, keyboard or focus.
* Rendering was decoupled from OSD visibility. `osd_core_message_active()`
  reports whether a message still needs drawing, and the hosts gate *rendering*
  on it while leaving *input* on the old visibility check:
  * Qt: `qt_osd_needs_render()` (`g_visible || osd_core_message_active()`) is
    used by the OpenGL, Vulkan and software renderers, including the software
    renderer's two repaint schedulers so messages still animate while paused.
    Every input path still uses `qt_osd_is_visible()`.
  * SDL: `osd_present()` returns early only when neither visible nor a message
    is active, guards `osd_core_build_ui()` behind `osd_visible`, and calls
    `osd_core_draw_indicators()` in both the shader and SDL-renderer branches.

`ImGuiConfigFlags_NoMouseCursorChange` is set in `sdl_osd.cpp`. Do not remove
it: both `ImGui_ImplSDL2_NewFrame()` and `ImGui_ImplSDL3_NewFrame()` call
`UpdateMouseCursor()`, which would otherwise call `SDL_ShowCursor()` on every
frame a message is drawn — including with the OSD closed and the guest holding
the mouse.

## What to do

1. **Remove the test scaffolding.** `message_test_pending` in
   `osd_core.cpp`, set in `osd_core_reset_to_menu()` and consumed in
   `osd_core_build_ui()`, fires a hardcoded 30-second message on every OSD
   invocation. Both blocks are marked `TEST CASE`.
2. **Add real call sites.** The obvious candidates are the mount and eject
   paths in `mount_path()` and `activate_menu_item()` — confirming what was
   loaded or ejected is exactly the feedback the OSD currently lacks, since
   `osd_core_draw_indicators()` was previously an empty stub.
3. **Decide on queueing.** There is one slot; a second `osd_core_show_message()`
   call replaces the first outright. Fine for occasional feedback, wrong if
   several events can fire together.
4. **Consider the SDL cost.** While a message is live the SDL host runs a full
   ImGui frame per blit, where it previously did nothing when the OSD was
   closed. Bounded to the message duration, but it is real work on the
   Raspberry-Pi-class targets this frontend exists for. Measure before adding
   frequent messages.

Layout is currently fixed at `osd_core_scaled(8.0f)` from the top left, and
messages do not survive a restart. Both are deliberate; revisit only if asked.

## Recorded implementation

All four items above were implemented and verified against `a5d75a42b`, then
reverted on request so they can be re-applied one step at a time. Nothing below
is in the tree. The code is post-`clang-format`; applying it verbatim keeps
`git clang-format --diff` clean. Every step touches only `src/osd/osd_core.cpp`
unless stated otherwise.

### Step 1 — include and duration constant

`path_get_filename()` is declared in `86box/path.h`, which carries its own
`extern "C"` guard, so it goes with the other guarded headers rather than in the
`extern "C"` block:

```cpp
#include <86box/path.h>
#include <86box/plat.h>
#include <86box/ui.h>
```

One constant for the duration, after `OSD_REF_HEIGHT`:

```cpp
static constexpr float OSD_MESSAGE_SECONDS  = 4.0f;
```

### Step 2 — extract `view_image_path()`

The view-to-mounted-path mapping inside `browser_initial_path()` is needed a
second time by the mount message, so lift it out. Replaces the `switch` and the
local `path` initialisation; the rest of `browser_initial_path()` is unchanged:

```cpp
/* Path of the image/folder this view's drive currently holds */
static char *
view_image_path(OsdView view)
{
    switch (view) {
        case VIEW_FILE_FLOPPY:
            return floppyfns[0];
        case VIEW_FILE_CD:
        case VIEW_CD_FOLDER:
            return cdrom[0].image_path;
        case VIEW_FILE_RDISK:
            return rdisk_drives[0].image_path;
        case VIEW_FILE_CART:
            return cart_fns[0];
        case VIEW_FILE_MO:
            return mo_drives[0].image_path;
        default:
            return nullptr;
    }
}

/* Get the start directory from currently mounted image */
static const char *
browser_initial_path(OsdView view)
{
    char *path = view_image_path(view);
```

### Step 3 — message text helpers

Both go at the top of the `Mount helpers` section, above `mount_path()`:

```cpp
/* Name of the media a view handles, for messages */
static const char *
view_media_name(OsdView view)
{
    switch (view) {
        case VIEW_FILE_FLOPPY:
            return "Floppy";
        case VIEW_FILE_CD:
        case VIEW_CD_FOLDER:
            return "CD-ROM";
        case VIEW_FILE_RDISK:
            return "Removable disk";
        case VIEW_FILE_CART:
            return "Cartridge";
        case VIEW_FILE_MO:
            return "MO";
        default:
            return "Media";
    }
}

/* Last path element, or the whole path when it has none (drive roots) */
static const char *
short_name(const char *path)
{
    const char *name = path_get_filename((char *) path);

    return (name[0] != '\0') ? name : path;
}
```

`short_name()` needs the fallback: `path_get_filename()` returns `""` for a path
ending in a separator, which the VISO folder browser can produce from a Windows
drive root (`osd_explorer.cpp` builds those as `C:\`).

### Step 4 — mount message

Appended to `mount_path()`, after the trailing `pclog_toggle_suppr()`. Reuses
the existing `msg` buffer; `current_view` is still the browser view here because
`draw_browser()` only switches to `VIEW_LOG` after `mount_path()` returns:

```cpp
    pclog_toggle_suppr();

    /* The mount calls return void; the stored path says whether one took. */
    const char *mounted = view_image_path(current_view);
    if ((mounted != nullptr) && (mounted[0] != '\0'))
        snprintf(msg, sizeof(msg), "%s: %s", view_media_name(current_view), short_name(mounted));
    else
        snprintf(msg, sizeof(msg), "%s: could not open %s", view_media_name(current_view), short_name(path));
    osd_core_show_message(msg, OSD_MESSAGE_SECONDS);
}
```

Reads as `Floppy: DISK1.IMG`, `CD-ROM: GAME.ISO`, `CD-ROM: MyFolder` for VISO,
or `Floppy: could not open DISK1.IMG`.

The failure branch is sound because every backing loader clears the stored path
when it fails, which is the same signal `ui_sb_update_icon_state()` already uses
to pick the empty-drive icon in these very functions:

| Loader | Failure behaviour |
|---|---|
| `fdd_load()` | `memset(floppyfns[drive], ...)` on the error path |
| `cdrom_load()` | sets `dev->image_path[0] = 0` when `dev->local == NULL` |
| `cart_load()` | `cart_load_error()` memsets `cart_fns[drive]` |
| `rdisk_load()` | writes `image_path` only under `if (ret)`, after `rdisk_disk_close()` cleared it |
| `mo_load()` | same shape as `rdisk_load()` |

### Step 5 — eject messages

Helper above `activate_menu_item()`:

```cpp
static void
eject_message(const char *media)
{
    char msg[64];

    snprintf(msg, sizeof(msg), "%s ejected", media);
    osd_core_show_message(msg, OSD_MESSAGE_SECONDS);
}
```

The five eject cases then become — note `clang-format` expands the one-line
table style once these lines are touched, so the compact form cannot be kept:

```cpp
        case ACT_EJECT_FLOPPY:
            floppy_eject(0);
            eject_message("Floppy");
            return;
        case ACT_EJECT_CD:
            cdrom_eject(0);
            eject_message("CD-ROM");
            return;
        case ACT_EJECT_RDISK:
            rdisk_eject(0);
            eject_message("Removable disk");
            return;
        case ACT_EJECT_CART:
            cartridge_eject(0);
            eject_message("Cartridge");
            return;
        case ACT_EJECT_MO:
            mo_eject(0);
            eject_message("MO");
            return;
```

Ejects are reported unconditionally rather than state-checked like mounts. The
menu items are already gated on a loaded medium, and the eject calls have no
meaningful failure path.

### Step 6 — drop the test scaffolding

Delete the `message_test_pending` static and its comment, the assignment in
`osd_core_reset_to_menu()` (leaving only `show_main_menu()`), and the whole
`TEST CASE` block at the top of `osd_core_build_ui()`.

### Step 7 — document the single slot

In `osd_core.hpp`:

```c
/* Show a non-blocking message that expires after the given seconds. There is
 * one slot: a second call replaces the message still on screen. */
void osd_core_show_message(const char *text, float seconds);
```

## Decisions taken

**Queueing: keep the single slot.** Every call site above is a menu activation
on the UI thread, one per keypress, and `mount_path()` is the only thing posting
during a mount, so two messages cannot overlap. A queue would be machinery with
no caller. Step 7 puts the constraint on the declaration so the next call site
has to confront it.

**SDL cost: no measurement needed at this scale.** Both hosts self-terminate —
`osd_core_draw_indicators()` is what decrements `message_left`, so the render
gate it feeds goes false on its own. Worst case is ~4 s of ImGui frames per
mount or eject, i.e. per deliberate keypress. Re-measure if messages are ever
attached to emulation events instead of menu actions.

Untouched, and worth knowing: on SDL `osd_present()` runs off the blit, so a
message posted while the machine is paused does not tick down until it resumes.
Qt's software renderer has repaint schedulers and does not share this.

## Verification performed

* Qt (`build/`) and SDL (`build/sdl/`) both compiled clean, with `osd_core.cpp`
  rebuilt in each and `sdl_osd.cpp` in the SDL one.
* `git clang-format --diff` reported no modifications.
* Compiled `osd_core.cpp` at `a5d75a42b` and with the change under
  `-Wall -Wextra`, and diffed the warning sets: the same four
  `-Wmissing-field-initializers` on `menu_items`, line-shifted only. No delta.
* **Not verified interactively.** The host had a display and ROMs but no
  key-injection tool (`xdotool`, `xte`, `ydotool` all absent), so the OSD could
  not be opened and walked to watch a message render. The overlay drawing and
  render gating are untouched by these steps, but the on-screen wording and the
  4-second duration are unconfirmed by eye.

## Constraints

* Formatting must come from running the tool, not from reading `.clang-format`:
  `git clang-format --force -- <files>`, then re-run with `--diff` and confirm
  it reports no modifications. Never format whole files — existing sources do
  not conform and a full-file run rewrites hundreds of unrelated lines.
* Build **both** frontends. `src/CMakeLists.txt` selects them exclusively, so a
  `QT=ON` build never compiles `src/unix` at all and vice versa — SDL-side
  changes go completely unverified in a Qt-only build.
* `CONTRIBUTING.md` requires that no new compiler warnings are introduced.
  `osd_core.cpp` has four pre-existing `-Wmissing-field-initializers` warnings
  on the `menu_items` separator rows; anything beyond those is yours.
* Keep diffs minimal and comments to a single line, matching the surrounding
  code.

## Files

| File | Role |
|---|---|
| `src/osd/osd_core.cpp` / `.hpp` | backend-neutral core: message state, drawing, view state machine |
| `src/osd/osd_explorer.cpp` / `.hpp` | generic file browser; knows nothing about 86Box media types |
| `src/qt/qt_osd.cpp` / `.hpp` | Qt host: context, input translation, `qt_osd_needs_render()` |
| `src/unix/sdl_osd.cpp` | SDL host: context, event routing, `osd_present()` |
| `src/qt/qt_{opengl,software,vulkanwindow}renderer.cpp` | render gates |
