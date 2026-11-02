# hy3 Features

**hy3** is a manual tiling engine plugin for [Hyprland](https://github.com/hyprwm/hyprland) that brings the deliberate, tree-based window management paradigm of **i3** and **sway** into Hyprland's modern Wayland compositor environment.

---

## Feature Summary Matrix

| Feature | hy3 | Hyprland Native (dwindle / master) | i3 / sway |
| :--- | :---: | :---: | :---: |
| **Manual Splits (H / V)** | **Yes** (Arbitrary nesting) | Limited (`dwindle` pseudo-splits) | **Yes** |
| **Container / Node Operations** | **Yes** (Move / close / focus subtrees) | No (Windows only) | **Yes** |
| **First-Class Tabbed Groups** | **Yes** (Native GL tabs, blur, OkLab colors) | Basic grouped windows | **Yes** (Standard titles) |
| **Hierarchical Focus (Raise/Lower)** | **Yes** (`changefocus`) | No | **Yes** (`focus parent/child`) |
| **Optional Autotiling** | **Yes** (Configurable triggers & workspaces) | Built-in (Automatic) | External scripts only |
| **Ephemeral Container Splitting** | **Yes** (Auto-cleans single-child splits) | N/A | No |
| **Multi-Monitor Traversal** | **Yes** (Both focus and window movement cross monitor boundaries) | Separate dispatchers | **Yes** |
| **Lua Configuration API** | **Yes** (First-class `hl.plugin.hy3.*`) | General Hyprland Lua bindings | No |

---

## 1. Manual Tiling & Tree Hierarchy

hy3 organizes windows into an explicit tree hierarchy instead of an automatic binary space partition.

### Arbitrary Nested Splits
- **Horizontal Splits (`SplitH`)**: Places child windows or sub-containers side-by-side along the X-axis.
- **Vertical Splits (`SplitV`)**: Stacks child windows or sub-containers top-to-bottom along the Y-axis.
- **Tree-Based Manipulation**: Containers can be arbitrarily nested (e.g. vertical splits inside horizontal splits inside tabs). Commands can act on a single window or an entire container branch.

### Group Wrapping & In-Place Layout Switching
- Wrap the focused window or container in a new split group using `hy3:makegroup`.
- Toggle or swap the split orientation in-place with `hy3:changegroup` or the `togglesplit` layout message without destroying the container structure.

### Ephemeral Containers
- Groups can be flagged as **ephemeral** (`ephemeral` or `force_ephemeral`).
- Ephemeral containers exist as long as they contain 2 or more children. If windows are closed or moved such that only one child remains, the ephemeral group automatically collapses and dissolves into its parent.

### Configurable Node Collapse Policies (`node_collapse_policy`)
Controls how the tree consolidates when nodes are closed or moved:
- `0`: Collapse any single-child container.
- `1`: Keep all nested groups intact (only empty containers are removed).
- `2` *(Default)*: Automatically remove single-child split containers (`SplitH`, `SplitV`), but preserve single-child tab groups unless nested in another tab group.

### Single-Window Group Insets (`group_inset`)
- Applies a configurable pixel offset (`group_inset`, default `10px`) when only one window is inside a group split, visually indicating that the window is inside a container ready to receive new splits.

---

## 2. Advanced Native Tabbed Groups

hy3 replaces Hyprland's default grouped window implementation with native, GPU-rendered tab bars.

```
+-------------------------------------------------------------+
| [ Tab 1: Terminal ] [ Tab 2: Editor ] [ Tab 3: Browser ]   | <- Hardware-accelerated Tab Bar
+-------------------------------------------------------------+
|                                                             |
|                                                             |
|                    Active Tab Window Content                |
|                                                             |
|                                                             |
+-------------------------------------------------------------+
```

### Rendering & Visual Styling
- **Hardware-Accelerated GL Rendering**: Tab bars are drawn directly in the render pass (`Hy3TabPassElement`) via custom OpenGL ES 2.0 shaders.
- **Antialiased Rounded Corners & Borders**: Configurable corner radius (`tabs:radius`) and borders (`tabs:border_width`) with sub-pixel antialiasing.
- **Compositor Background Blur**: Dynamically samples Hyprland's blur framebuffer (`tabs:blur = true`) for translucent, blurred tab backgrounds.
- **Cairo & Pango Typography**: Window titles are crisp, antialiased, and automatically ellipsized (`tabs:render_text`, `tabs:text_font`, `tabs:text_height`, `tabs:text_center`, `tabs:text_padding`).

### Dynamic Color States & OkLab Interpolation
Tab segments blend smoothly across six distinct visual states using perceptual **OkLab** color space interpolation:
1. **Active**: The active tab on the currently focused monitor.
2. **Active Alt Monitor**: The active tab on an unfocused monitor.
3. **Focused**: The active tab within a container that is not currently focused.
4. **Inactive**: Unselected tabs.
5. **Urgent**: Tab containing a window demanding attention (`m_isUrgent`).
6. **Locked**: Tab group locked via `hy3:locktab`.

### Interactive Tab Features
- **Mouse Click Selection**: Clicking any tab segment immediately focuses and raises the associated window.
- **Tab Navigation (`hy3:focustab`)**:
  - Switch tabs sequentially (`left`, `right`) with optional edge wrapping (`wrap`).
  - Jump directly to a tab by 1-based index (`index, <num>`).
  - Mouse hover modes: `prioritize_hovered` (targets tab group under pointer) or `require_hovered` (acts only if cursor is over tab bar).
- **Tab Locking (`hy3:locktab`)**:
  - Locks a tab container so it behaves like a single opaque node.
  - Prevents directional movement commands from accidentally inserting windows into or breaking windows out of the tab group.
- **Automatic First-Window Tab (`tab_first_window`)**:
  - Automatically wraps the very first window spawned in an empty workspace inside a tabbed container.

---

## 3. Directional Focus, Movement & Sizing

### Directional Focus Navigation (`hy3:movefocus`)
- Move focus `left`, `right`, `up`, or `down` through nested containers and across screen splits.
- **Visible-Only Filtering (`visible`)**: Skips hidden background tabs and only jumps between visually rendered windows.
- **Cursor Warping Control (`warp` / `nowarp`)**: Override compositor-level cursor warping per keybinding.
- **Cross-Monitor Navigation**: Seamlessly jumps to the adjacent physical monitor when reaching the edge of a workspace.

### Directional Window Movement (`hy3:movewindow`)
- Shift windows across the container tree in any direction (`left`, `right`, `up`, `down`).
- **`once` Option**: Moves the window directly into or out of the immediate parent container without drilling into deeper nested child splits.
- **`visible` Option**: Moves directly to visible siblings, skipping hidden tabs.
- **Seamless Cross-Monitor Movement**: When moving a window past the outer boundary of the workspace root, hy3 automatically shifts the window (or focused container subtree) to the adjacent physical monitor.
- **Inverted Edge Docking**: Windows cleanly dock at the opposing edge of the target display (e.g. moving `right` enters on the `left` edge of the next monitor). If the destination root split does not match the movement axis, it wraps into an appropriate split automatically.
- **Configurable & Overridable**: Enabled by default (`plugin:hy3:move_window_cross_monitor = true`). Can be toggled per command via `[cross_monitor | no_cross_monitor]`. If no adjacent monitor exists, hy3 falls back to wrapping into a local split.

### Hierarchical Focus Navigation (`hy3:changefocus`)
Traverse the tree vertically along its depth axis:
- `top`: Focuses the entire workspace container.
- `bottom`: Focuses the single leaf window at the bottom of the active branch.
- `raise`: Moves focus one level up to the parent container.
- `lower`: Moves focus one level down into the focused child.
- `tab`: Raises focus to the nearest ancestral tab container.
- `tabnode`: Raises focus to the direct child node beneath the nearest tab container.

### Layer & Cursor Dispatchers
- **`hy3:togglefocuslayer`**: Toggles focus back and forth between the tiled layout tree and floating windows on the workspace.
- **`hy3:warpcursor`**: Warps the mouse pointer directly to the center of the currently focused window or container.
- **`hy3:killactive`**: Recursively closes all windows inside the currently focused node or container (e.g. closing an entire split branch at once).

### Window Size Equalization (`hy3:equalize`)
- **Group Equalization**: Resets `size_ratio` of all immediate siblings in the focused container to equal proportions.
- **Workspace Equalization (`workspace`)**: Recursively resets window size ratios across the entire workspace tree.

---

## 4. Cross-Workspace & Monitor Operations

### Tree-Preserving Workspace Migration (`hy3:movetoworkspace`)
- Moves the **entire selected container node or subtree** to another workspace, preserving its internal layout, tab configurations, and split ratios.
- Options:
  - `follow`: Switch focus to the target workspace after moving.
  - `warp` / `nowarp`: Explicitly enable or disable cursor warping.
- **Special Workspace Support**: Fully compatible with Hyprland scratchpads and special workspaces.
- **hyprsplit Interoperability**: Detects and integrates with the [hyprsplit](https://github.com/shezdy/hyprsplit) plugin via dynamic linking (`dlsym`) for per-monitor workspace indexing.

---

## 5. Intelligent Autotiling

For users who want automatic split direction decisions without losing manual tree control:

- **Dimension Triggering**:
  - `trigger_width`: Automatically creates a vertical split if squishing a window horizontally would reduce its width below this pixel threshold.
  - `trigger_height`: Automatically creates a horizontal split if squishing a window vertically would reduce its height below this pixel threshold.
- **Ephemeral Autotile Splits**: Splits created by autotiling can be configured as ephemeral (`autotile:ephemeral_groups = true`), collapsing cleanly when windows close.
- **Per-Workspace Filtering**:
  - Whitelist: `autotile:workspaces = 1,2,3`
  - Blacklist: `autotile:workspaces = not:4,5`
  - Universal: `autotile:workspaces = all`

---

## 6. Experimental & Preview Features

> [!WARNING]
> These features are labeled as alpha/experimental in the codebase.

- **Tree Expansion (`hy3:expand`)**:
  - Zoom / latch a node to cover the area of its containing parent without destroying sibling windows.
  - Modes: `expand` (zoom out one level), `shrink` (zoom in one level), `base` (restore normal layout).
  - Maximization options: `intermediate_maximize`, `fullscreen_maximize`, `maximize_only`.
- **Window Swallowing (`hy3:setswallow`)**:
  - Toggle window containment / swallowing on the focused node (`true`, `false`, `toggle`).
- **Tree Debugger (`hy3:debugnodes`)**:
  - Dumps the exact in-memory node hierarchy (addresses, layouts, sizes, focus states, and window IDs) into the Hyprland log.

---

## 7. Complete Configuration Reference

Add these options within `plugin:hy3` in your `hyprland.conf`:

```ini
plugin {
  hy3 {
    # General layout options
    no_gaps_when_only = 0          # (int) 0 = disabled, 1 = no gaps with 1 window, 2 = no gaps with single container
    node_collapse_policy = 2       # (int) 0 = collapse single-child groups, 1 = keep nested, 2 = collapse empty splits
    group_inset = 10               # (int) Pixel inset for single-child groups
    tab_first_window = false       # (bool) Auto-create tab group on first window in workspace
    move_window_cross_monitor = true # (bool) Move windows across monitors when shifting past workspace edge

    # Tab bar settings
    tabs {
      height = 22                  # (int) Tab bar height in pixels
      padding = 5                  # (int) Padding between tab bar and focused window
      from_top = false             # (bool) Tab bar slides in from top instead of bottom
      radius = 6                   # (int) Corner radius of tab segments
      border_width = 2             # (int) Border stroke width
      render_text = true           # (bool) Render window titles on tabs
      text_center = true           # (bool) Center window titles within tab segments
      text_font = Sans             # (string) Font family for titles
      text_height = 8              # (int) Font height in points
      text_padding = 3             # (int) Text horizontal padding
      blur = true                  # (bool) Enable background blur on translucent tabs
      opacity = 1.0                # (float) Master opacity multiplier for tab bar

      colors {
        # Active tab (focused window on focused monitor)
        active = rgba(33ccff40)
        active_border = rgba(33ccffee)
        active_text = rgba(ffffffff)

        # Active tab on unfocused monitor
        active_alt_monitor = rgba(60606040)
        active_alt_monitor_border = rgba(808080ee)
        active_alt_monitor_text = rgba(ffffffff)

        # Focused tab in unfocused container
        focused = rgba(60606040)
        focused_border = rgba(808080ee)
        focused_text = rgba(ffffffff)

        # Inactive tab
        inactive = rgba(30303020)
        inactive_border = rgba(606060aa)
        inactive_text = rgba(ffffffff)

        # Urgent tab
        urgent = rgba(ff223340)
        urgent_border = rgba(ff2233ee)
        urgent_text = rgba(ffffffff)

        # Locked tab
        locked = rgba(90903340)
        locked_border = rgba(909033ee)
        locked_text = rgba(ffffffff)
      }
    }

    # Autotiling settings
    autotile {
      enable = false               # (bool) Enable automatic split management
      ephemeral_groups = true      # (bool) Mark autotile splits as ephemeral
      trigger_width = 0            # (int) Pixel width threshold for vertical split (0 = always, -1 = never)
      trigger_height = 0           # (int) Pixel height threshold for horizontal split (0 = always, -1 = never)
      workspaces = all             # (string) Workspaces enabled for autotile (e.g. "1,2", "not:3,4", "all")
    }
  }
}
```

---

## 8. Dispatcher Reference

### Layout & Group Control

| Dispatcher | Arguments | Description |
| :--- | :--- | :--- |
| `hy3:makegroup` | `<h \| v \| tab \| opposite>, [toggle], [ephemeral \| force_ephemeral]` | Wrap focused node in a split or tab group. `toggle` unwraps single-child parents. |
| `hy3:changegroup` | `<h \| v \| tab \| untab \| toggletab \| opposite>` | Changes layout of the current group container in-place. |
| `hy3:setephemeral` | `<true \| false>` | Changes ephemerality of the containing group. |
| `hy3:equalize` | `[workspace]` | Equalizes window ratios in immediate group, or whole workspace if `workspace` is passed. |

### Navigation & Window Movement

| Dispatcher | Arguments | Description |
| :--- | :--- | :--- |
| `hy3:movefocus` | `<l \| r \| u \| d>, [visible], [warp \| nowarp]` | Moves focus directionally across splits, tabs, and monitors. |
| `hy3:movewindow` | `<l \| r \| u \| d>, [once], [visible], [cross_monitor \| no_cross_monitor]` | Moves focused window/container directionally across containers and monitors. |
| `hy3:movetoworkspace` | `<workspace>, [follow, [warp \| nowarp]]` | Moves entire focused node/subtree to target workspace. |
| `hy3:changefocus` | `<top \| bottom \| raise \| lower \| tab \| tabnode>` | Moves focus up/down the hierarchy tree. |
| `hy3:togglefocuslayer` | `[nowarp]` | Toggles focus between tiled windows and floating windows. |
| `hy3:warpcursor` | *None* | Warps pointer to the center of the focused window/node. |
| `hy3:killactive` | *None* | Recursively closes all windows inside the focused container. |

### Tab Control

| Dispatcher | Arguments | Description |
| :--- | :--- | :--- |
| `hy3:focustab` | `[l \| r \| index, <idx>], [prioritize_hovered \| require_hovered], [wrap]` | Shifts tab focus left/right or selects tab by 1-based index. |
| `hy3:locktab` | `[lock \| unlock \| toggle]` | Locks tab group to behave as a single opaque node. |

### Experimental

| Dispatcher | Arguments | Description |
| :--- | :--- | :--- |
| `hy3:expand` | `<expand \| shrink \| base>` | Expands/collapses focused node over parent container bounds. |
| `hy3:setswallow` | `<true \| false \| toggle>` | Toggles window swallowing state on containing node. |
| `hy3:debugnodes` | *None* | Dumps the layout node hierarchy tree into the Hyprland log. |

---

## 9. Lua Dispatcher API Reference

For configurations utilizing Hyprland's Lua bindings, hy3 registers native dispatcher factories under `hl.plugin.hy3.*`:

```lua
local hy3 = hl.plugin.hy3

-- Group management
hy3.make_group("h" | "v" | "tab" | "opposite", {
    toggle = true | false,              -- default: false
    ephemeral = true | false | "force", -- default: false
})
hy3.change_group("h" | "v" | "tab" | "untab" | "toggletab" | "opposite")
hy3.set_ephemeral(true | false)

-- Navigation & movement
hy3.move_focus("left" | "right" | "up" | "down", {
    visible = true | false, -- default: false
    warp = true | false,    -- default: follows cursor:no_warps
})
hy3.move_window("left" | "right" | "up" | "down", {
    once = true | false,           -- default: false
    visible = true | false,        -- default: false
    cross_monitor = true | false,  -- default: follows plugin:hy3:move_window_cross_monitor
})
hy3.move_to_workspace("<workspace>", {
    follow = true | false,  -- default: false
    warp = true | false,    -- default: follows cursor:no_warps
})
hy3.change_focus("top" | "bottom" | "raise" | "lower" | "tab" | "tabnode")
hy3.toggle_focus_layer({ warp = true | false })
hy3.warp_cursor()
hy3.kill_active()

-- Tab manipulation
hy3.focus_tab({
    direction = "left" | "right",                                -- or index = <number>
    mouse = "ignore" | "prioritize_hovered" | "require_hovered", -- default: "ignore"
    wrap = true | false,                                         -- default: false
})
hy3.lock_tab("lock" | "unlock" | "toggle")

-- Resizing & equalization
hy3.equalize({
    scope = "group" | "workspace", -- default: "group"
    workspace = true | false,
    recursive = true | false,
})

-- Experimental & Debug
hy3.expand("expand" | "shrink" | "base")
hy3.set_swallow("true" | "false" | "toggle")
hy3.debug_nodes()
```
