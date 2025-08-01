# Tap-Hold Configuration Options

While Tap-Hold options are fantastic, they are not without their issues.  We have tried to configure them with reasonable defaults, but that may still cause issues for some people.

These options let you modify the behavior of the Tap-Hold keys.

## Tapping Term

The crux of all of the following features is the tapping term setting.  This determines what is a tap and what is a hold.  The exact timing for this to feel natural can vary from keyboard to keyboard, from switch to switch, and from key to key.

::: tip
`DYNAMIC_TAPPING_TERM_ENABLE` enables three special keys that can help you quickly find a comfortable tapping term for you. See "Dynamic Tapping Term" for more details.
:::

You can set the global time for this by adding the following setting to your `config.h`:

```c
#define TAPPING_TERM 200
```

This setting is defined in milliseconds and defaults to 200ms.  This is a good average for the majority of people.

For more granular control of this feature, you can add the following to your `config.h`:
```c
#define TAPPING_TERM_PER_KEY
```

You can then add the following function to your keymap:

```c
uint16_t get_tapping_term(uint16_t keycode, keyrecord_t *record) {
    switch (keycode) {
        case SFT_T(KC_SPC):
            return TAPPING_TERM + 1250;
        case LT(1, KC_GRV):
            return 130;
        default:
            return TAPPING_TERM;
    }
}
```

### Dynamic Tapping Term {#dynamic-tapping-term}

`DYNAMIC_TAPPING_TERM_ENABLE` is a feature you can enable in `rules.mk` that lets you use three special keys in your keymap to configure the tapping term on the fly.

| Key                           | Aliases | Description                                                                               |
|-------------------------------|---------|-------------------------------------------------------------------------------------------|
|`QK_DYNAMIC_TAPPING_TERM_PRINT`|`DT_PRNT`| Types the current tapping term, in milliseconds                                           |
|`QK_DYNAMIC_TAPPING_TERM_UP`   |`DT_UP`  | Increases the current tapping term by `DYNAMIC_TAPPING_TERM_INCREMENT`ms (5ms by default) |
|`QK_DYNAMIC_TAPPING_TERM_DOWN` |`DT_DOWN`| Decreases the current tapping term by `DYNAMIC_TAPPING_TERM_INCREMENT`ms (5ms by default) |

Set the tapping term as usual with `#define TAPPING_TERM <value>` in `config.h` and add `DYNAMIC_TAPPING_TERM_ENABLE = yes` in `rules.mk`. Then, place the above three keys somewhere in your keymap and flash the new firmware onto your board.

Now, you can try using your dual-role keys, such as layer-taps and mod-taps, and use `DT_DOWN` and `DT_UP` to adjust the tapping term immediately. If you find that you frequently trigger the modifier of your mod-tap(s) by accident, for example, that's a sign that your tapping term may be too low so tap `DT_UP` a few times to increase the tapping term until that no longer happens. On the flip side, if you get superfluous characters when you actually intended to momentarily activate a layer, tap `DT_DOWN` to lower the tapping term. Do note that these keys affect the *global* tapping term, you cannot change the tapping term of a specific key on the fly.

Once you're satisfied with the current tapping term value, open `config.h` and replace whatever value you first wrote for the tapping term by the output of the `DT_PRNT` key.

It's important to update `TAPPING_TERM` with the new value because the adjustments made using `DT_UP` and `DT_DOWN` are not persistent.

The value by which the tapping term increases or decreases when you tap `DT_UP` and `DT_DOWN` can be configured in `config.h` with `#define DYNAMIC_TAPPING_TERM_INCREMENT <new value>`. Note that the tapping term is *not* modified when holding down the tap term keys so if you need to, for example, decrease the current tapping term by 50ms, you cannot just press down and hold `DT_DOWN`; you will have to tap it 10 times in a row with the default increment of 5ms.

If you need more flexibility, nothing prevents you from defining your own custom keys to dynamically change the tapping term.

```c
enum custom_dynamic_tapping_term_keys = {
    DT_UP_50 = SAFE_RANGE,
    DT_DOWN_50,
    DT_UP_X2,
    DT_DOWN_X2,
}

bool process_record_user(uint16_t keycode, keyrecord_t *record) {
    switch (keycode) {
    case DT_UP_50:
        if (record->event.pressed) {
            g_tapping_term += 50;
        }
        break;
    case DT_DOWN_50:
        if (record->event.pressed) {
            g_tapping_term -= 50;
        }
        break;
    case DT_UP_X2:
        if (record->event.pressed) {
            g_tapping_term *= 2;
        }
        break;
    case DT_DOWN_X2:
        if (record->event.pressed) {
            g_tapping_term /= 2;
        }
        break;
    }
    return true;
};
```

In order for this feature to be effective if you use per-key tapping terms, you need to make a few changes to the syntax of the `get_tapping_term` function. All you need to do is replace every occurrence of `TAPPING_TERM` in the `get_tapping_term` function by lowercase `g_tapping_term`. If you don't do that, you will still see the value typed by `DT_PRNT` go up and down as you configure the tapping term on the fly but you won't feel those changes as they don't get applied. If you can go as low as 10ms and still easily trigger the tap function of a dual-role key, that's a sign that you forgot to make the necessary changes to your `get_tapping_term` function.

For instance, here's how the example `get_tapping_term` shown earlier should look after the transformation:

```c
uint16_t get_tapping_term(uint16_t keycode, keyrecord_t *record) {
    switch (keycode) {
        case SFT_T(KC_SPC):
           return g_tapping_term + 1250;
        case LT(1, KC_GRV):
            return 130;
        default:
            return g_tapping_term;
    }
}
```

The reason is that `TAPPING_TERM` is a macro that expands to a constant integer and thus cannot be changed at runtime whereas `g_tapping_term` is a variable whose value can be changed at runtime. If you want, you can temporarily enable `DYNAMIC_TAPPING_TERM_ENABLE` to find a suitable tapping term value and then disable that feature and revert back to using the classic syntax for per-key tapping term settings. In case you need to access the tapping term from elsewhere in your code, you can use the `GET_TAPPING_TERM(keycode, record)` macro. This macro will expand to whatever is the appropriate access pattern given the current configuration.

## Tap-Or-Hold Decision Modes

The code which decides between the tap and hold actions of dual-role keys supports three different modes, in increasing order of preference for the hold action:

1. The default mode selects the hold action only if the dual-role key is held down longer than the tapping term.  In this mode pressing other keys while the dual-role key is held down does not influence the tap-or-hold decision. In other words, this mode ignores interrupts.

2. The “permissive hold” mode, in addition to the default behavior, immediately selects the hold action when another key is tapped (pressed and then released) while the dual-role key is held down, even if this happens earlier than the tapping term.  If another key is just pressed, but then the dual-role key is released before that other key (and earlier than the tapping term), this mode will still select the tap action.

3. The “hold on other key press” mode, in addition to the default behavior, immediately selects the hold action when another key is pressed while the dual-role key is held down, even if this happens earlier than the tapping term.

Note that until the tap-or-hold decision completes (which happens when either the dual-role key is released, or the tapping term has expired, or the extra condition for the selected decision mode is satisfied), key events are delayed and not transmitted to the host immediately.  The default mode gives the most delay (if the dual-role key is held down, this mode always waits for the whole tapping term), and the other modes may give less delay when other keys are pressed, because the hold action may be selected earlier.

### Comparison {#comparison}

To better illustrate the tap-or-hold decision modes, let us compare the expected output of each decision mode in a handful of tapping scenarios involving a mod-tap key (`LSFT_T(KC_A)`) and a regular key (`KC_B`) with the `TAPPING_TERM` set to 200ms.

Note: "`kc` held" in the "Physical key event" column means that the key wasn't physically released yet at this point in time.

#### Distinct taps (AABB) {#distinct-taps}

| Time | Physical key event |    Default     | `PERMISSIVE_HOLD` |  `HOLD_ON_OTHER_KEY_PRESS` |
|------|--------------------|----------------|-------------------|----------------------------|
| 0    | `LSFT_T(KC_A)` down|                |                   |                            |
| 199  | `LSFT_T(KC_A)` up  | a              | a                 |  a                         |
| 210  | `KC_B`      down   | ab             | ab                |  ab                        |
| 220  | `KC_B`      up     | ab             | ab                |  ab                        |

| Time | Physical key event |    Default     | `PERMISSIVE_HOLD` |  `HOLD_ON_OTHER_KEY_PRESS` |
|------|--------------------|----------------|-------------------|----------------------------|
| 0    | `LSFT_T(KC_A)` down|                |                   |                            |
| 200  | `LSFT_T(KC_A)` held|<kbd>Shift</kbd>| <kbd>Shift</kbd>  | <kbd>Shift</kbd>           |
| 201  | `LSFT_T(KC_A)` up  |<kbd>Shift</kbd>| <kbd>Shift</kbd>  | <kbd>Shift</kbd>           |
| 205  | `KC_B`      down   | b              | b                 |  b                         |
| 210  | `KC_B`      up     | b              | b                 |  b                         |

#### Nested tap (ABBA) {#nested-tap}

| Time | Physical key event |    Default     | `PERMISSIVE_HOLD` |  `HOLD_ON_OTHER_KEY_PRESS` |
|------|--------------------|----------------|-------------------|----------------------------|
| 0    | `LSFT_T(KC_A)` down|                |                   |                            |
| 110  | `KC_B` down        |                |                   | B                          |
| 120  | `KC_B` up          |                | B                 | B                          |
| 199  | `LSFT_T(KC_A)` up  | ab             | B                 | B                          |

| Time | Physical key event |    Default     | `PERMISSIVE_HOLD` |  `HOLD_ON_OTHER_KEY_PRESS` |
|------|--------------------|----------------|-------------------|----------------------------|
| 0    | `LSFT_T(KC_A)` down|                |                   |                            |
| 110  | `KC_B` down        |                |                   | B                          |
| 120  | `KC_B` up          |                | B                 | B                          |
| 200  | `LSFT_T(KC_A)` held| B              | B                 | B                          |
| 210  | `LSFT_T(KC_A)` up  | B              | B                 | B                          |

| Time | Physical key event |    Default     | `PERMISSIVE_HOLD` |  `HOLD_ON_OTHER_KEY_PRESS` |
|------|--------------------|----------------|-------------------|----------------------------|
| 0    | `LSFT_T(KC_A)` down|                |                   |                            |
| 200  | `LSFT_T(KC_A)` held|<kbd>Shift</kbd>| <kbd>Shift</kbd>  | <kbd>Shift</kbd>           |
| 205  | `KC_B` down        | B              | B                 | B                          |
| 210  | `KC_B` up          | B              | B                 | B                          |
| 220  | `LSFT_T(KC_A)` up  | B              | B                 | B                          |

#### Rolling keys (ABAB) {#rolling-keys}

| Time | Physical key event |    Default     | `PERMISSIVE_HOLD` |  `HOLD_ON_OTHER_KEY_PRESS` |
|------|--------------------|----------------|-------------------|----------------------------|
| 0    | `LSFT_T(KC_A)` down|                |                   |                            |
| 110  | `KC_B` down        |                |                   | B                          |
| 130  | `LSFT_T(KC_A)` up  | ab             | ab                | B                          |
| 140  | `KC_B` up          | ab             | ab                | B                          |

| Time | Physical key event |    Default     | `PERMISSIVE_HOLD` |  `HOLD_ON_OTHER_KEY_PRESS` |
|------|--------------------|----------------|-------------------|----------------------------|
| 0    | `LSFT_T(KC_A)` down|                |                   |                            |
| 110  | `KC_B` down        |                |                   | B                          |
| 200  | `LSFT_T(KC_A)` held| B              | B                 | B                          |
| 205  | `LSFT_T(KC_A)` up  | B              | B                 | B                          |
| 210  | `KC_B` up          | B              | B                 | B                          |

### Default Mode
Example sequence 1 (the `L` key is also mapped to `KC_RGHT` on layer 2):

```
             TAPPING_TERM
  +---------------|--------------------+
  | +-------------|-------+            |
  | | LT(2, KC_A) |       |            |
  | +-------------|-------+            |
  |               | +--------------+   |
  |               | | KC_L         |   |
  |               | +--------------+   |
  +---------------|--------------------+
```
The above sequence would send a `KC_RGHT`, since `LT(2, KC_A)` is held longer than the `TAPPING_TERM`.

---

Example sequence 2 (the `L` key is also mapped to `KC_RGHT` on layer 2):

```
                           TAPPING_TERM
  +-----------------------------|------+
  | +---------------+           |      |
  | | LT(2, KC_A)   |           |      |
  | +---------------+           |      |
  |            +--------------+ |      |
  |            | KC_L         | |      |
  |            +--------------+ |      |
  +-----------------------------|------+
```
The above sequence will not send `KC_RGHT` but `KC_A` `KC_L` instead, since `LT(2, KC_A)` is not held longer than the `TAPPING_TERM`.

---

Example sequence 3 (Mod Tap):

```
                         TAPPING_TERM
  +---------------------------|--------+
  | +-------------+           |        |
  | | SFT_T(KC_A) |           |        |
  | +-------------+           |        |
  |       +--------------+    |        |
  |       | KC_X         |    |        |
  |       +--------------+    |        |
  +---------------------------|--------+
```
In the above sequence, `SFT_T(KC_A)` has been released before the end of its `TAPPING_TERM` and as such will be interpreted as `KC_A`,
followed by any key event that happened after the initial press of `SFT_T(KC_A)`. In this instance, the output would be `KC_A` `KC_X`.

### Permissive Hold

The “permissive hold” mode can be enabled for all dual-role keys by adding the corresponding option to `config.h`:

```c
#define PERMISSIVE_HOLD
```

This makes tap and hold keys (like Layer Tap) work better for fast typists, or for high `TAPPING_TERM` settings.

If you press a dual-role key, tap another key (press and release) and then release the dual-role key, all within the tapping term, by default the dual-role key will perform its tap action.  If the `PERMISSIVE_HOLD` option is enabled, the dual-role key will perform its hold action instead.

An example of a sequence that is affected by the “permissive hold” mode:

- `LT(2, KC_A)` Down
- `KC_L` Down (the `L` key is also mapped to `KC_RGHT` on layer 2)
- `KC_L` Up
- `LT(2, KC_A)` Up

```
                         TAPPING_TERM   
  +---------------------------|--------+
  | +----------------------+  |        |
  | | LT(2, KC_A)          |  |        |
  | +----------------------+  |        |
  |    +--------------+       |        |
  |    | KC_L         |       |        |
  |    +--------------+       |        |
  +---------------------------|--------+
```

Normally, if you do all this within the `TAPPING_TERM` (default: 200ms), this will be registered as `al` by the firmware and host system.  With the `PERMISSIVE_HOLD` option enabled, the Layer Tap key is considered as a layer switch if another key is tapped, and the above sequence would be registered as `KC_RGHT` (the mapping of `L` on layer 2). We could describe this sequence as a “nested tap” (the modified key's key down and key up events are “nested” between the dual-role key's key down and key up events).

However, this slightly different sequence will not be affected by the “permissive hold” mode:

- `LT(2, KC_A)` Down
- `KC_L` Down (the `L` key is also mapped to `KC_RGHT` on layer 2)
- `LT(2, KC_A)` Up
- `KC_L` Up

```
                         TAPPING_TERM   
  +---------------------------|--------+
  | +-------------+           |        |
  | | LT(2, KC_A) |           |        |
  | +-------------+           |        |
  |       +--------------+    |        |
  |       | KC_L         |    |        |
  |       +--------------+    |        |
  +---------------------------|--------+
```

In the sequence above the dual-role key is released before the other key is released, and if that happens within the tapping term, the “permissive hold” mode will still choose the tap action for the dual-role key, and the sequence will be registered as `al` by the host. We could describe this as a “rolling press” (the two keys' key down and key up events behave as if you were rolling a ball across the two keys, first pressing each key down in sequence and then releasing them in the same order).

::: tip
The `PERMISSIVE_HOLD` option is not noticeable if you also enable `HOLD_ON_OTHER_KEY_PRESS` because the latter option considers both the “nested tap” and “rolling press” sequences like shown above as a hold action, not the tap action. `HOLD_ON_OTHER_KEY_PRESS` makes the Tap-Or-Hold decision earlier in the chain of key events, thus taking a precedence over `PERMISSIVE_HOLD`.
:::

For more granular control of this feature, you can add the following to your `config.h`:

```c
#define PERMISSIVE_HOLD_PER_KEY
```

You can then add the following function to your keymap:

```c
bool get_permissive_hold(uint16_t keycode, keyrecord_t *record) {
    switch (keycode) {
        case LT(1, KC_BSPC):
            // Immediately select the hold action when another key is tapped.
            return true;
        default:
            // Do not select the hold action when another key is tapped.
            return false;
    }
}
```

### Hold On Other Key Press

The “hold on other key press” mode can be enabled for all dual-role keys by adding the corresponding option to `config.h`:

```c
#define HOLD_ON_OTHER_KEY_PRESS
```

This mode makes tap and hold keys (like Layer Tap) work better for fast typists, or for high `TAPPING_TERM` settings.  Compared to the “permissive hold” mode, this mode selects the hold action in more cases.

If you press a dual-role key, press another key, and then release the dual-role key, all within the tapping term, by default the dual-role key will perform its tap action.  If the `HOLD_ON_OTHER_KEY_PRESS` option is enabled, the dual-role key will perform its hold action instead.

An example of a sequence that is affected by the “hold on other key press” mode, but not by the “permissive hold” mode:

- `LT(2, KC_A)` Down
- `KC_L` Down (the `L` key is also mapped to `KC_RGHT` on layer 2)
- `LT(2, KC_A)` Up
- `KC_L` Up

```
                         TAPPING_TERM
  +---------------------------|--------+
  | +-------------+           |        |
  | | LT(2, KC_A) |           |        |
  | +-------------+           |        |
  |       +--------------+    |        |
  |       | KC_L         |    |        |
  |       +--------------+    |        |
  +---------------------------|--------+
```

Normally, if you do all this within the `TAPPING_TERM` (default: 200ms), this will be registered as `al` by the firmware and host system.  With the `HOLD_ON_OTHER_KEY_PRESS` option enabled, the Layer Tap key is considered as a layer switch if another key is pressed, and the above sequence would be registered as `KC_RGHT` (the mapping of `L` on layer 2).

For more granular control of this feature, you can add the following to your `config.h`:

```c
#define HOLD_ON_OTHER_KEY_PRESS_PER_KEY
```

You can then add the following function to your keymap:

```c
bool get_hold_on_other_key_press(uint16_t keycode, keyrecord_t *record) {
    switch (keycode) {
        case LT(1, KC_BSPC):
            // Immediately select the hold action when another key is pressed.
            return true;
        default:
            // Do not select the hold action when another key is pressed.
            return false;
    }
}
```

## Quick Tap Term

When the user holds a key after tapping it, the tapping function is repeated by default, rather than activating the hold function. This allows keeping the ability to auto-repeat the tapping function of a dual-role key. `QUICK_TAP_TERM` enables fine tuning of that ability. If set to `0`, it will remove the auto-repeat ability and activate the hold function instead.

`QUICK_TAP_TERM` is set to `TAPPING_TERM` by default, which is the maximum allowed value for `QUICK_TAP_TERM`. To override its value (in milliseconds) add the following to your `config.h`:

```c
#define QUICK_TAP_TERM 120
```

Example:

- `SFT_T(KC_A)` Down
- `SFT_T(KC_A)` Up
- `SFT_T(KC_A)` Down
- (wait until tapping term expires...)

With default settings, `a` will be sent on the first release, then `a` will be sent on the second press allowing the computer to trigger its auto repeat function until the key is released.

With `QUICK_TAP_TERM` configured, the timing between `SFT_T(KC_A)` up and `SFT_T(KC_A)` down must be within `QUICK_TAP_TERM` to trigger auto repeat. Otherwise the second press will be sent as a Shift. If `QUICK_TAP_TERM` is set to `0`, the second press will always be sent as a Shift, effectively disabling auto-repeat.

::: warning
`QUICK_TAP_TERM` timing will also impact anything that uses tapping toggles (Such as the `TT` layer keycode, and the One Shot Tap Toggle).
:::

For more granular control of this feature, you can add the following to your `config.h`:

```c
#define QUICK_TAP_TERM_PER_KEY
```

You can then add the following function to your keymap:

```c
uint16_t get_quick_tap_term(uint16_t keycode, keyrecord_t *record) {
    switch (keycode) {
        case SFT_T(KC_SPC):
            return QUICK_TAP_TERM - 20;
        default:
            return QUICK_TAP_TERM;
    }
}
```

::: tip
If `QUICK_TAP_TERM` is set higher than `TAPPING_TERM`, it will default to `TAPPING_TERM`.
:::

## Flow Tap

Flow Tap modifies mod-tap `MT` and layer-tap `LT` keys such that when pressed within a short timeout of the preceding key, the tapping behavior is triggered. This is particularly useful for home row mods to avoid accidental mod triggers. It basically disables the hold behavior during fast typing, creating a "flow of taps." This also helps to reduce the input lag of tap-hold keys during fast typing, since the tapped behavior is sent immediately.

Flow Tap is enabled by defining `FLOW_TAP_TERM` in your `config.h` with the desired timeout in milliseconds. A timeout of 150&nbsp;ms is recommended as a starting point:

```c
#define FLOW_TAP_TERM 150
```

By default, Flow Tap is enabled when:

* The tap-hold key is pressed within `FLOW_TAP_TERM` milliseconds of the previous key press.

* The tapping keycodes of the previous key and tap-hold key are *both* among `KC_A`&ndash;`KC_Z`, `KC_COMM`, `KC_DOT`, `KC_SCLN`, `KC_SLSH` (the main alphas area of a conventional QWERTY layout) or `KC_SPC`.

As an exception to the above, Flow Tap is temporarily disabled while a tap-hold key is undecided. This is to allow chording multiple mod-tap keys without having to wait out the Flow Tap term.


### is_flow_tap_key()

Optionally, define the `is_flow_tap_key()` callback to specify where Flow Tap is enabled. The callback is called for both the tap-hold key *and* the key press immediately preceding it, and if the callback returns true for both keycodes, Flow Tap is enabled.

The default implementation of this callback is:

```c
bool is_flow_tap_key(uint16_t keycode) {
    if ((get_mods() & (MOD_MASK_CG | MOD_BIT_LALT)) != 0) {
        return false; // Disable Flow Tap on hotkeys.
    }
    switch (get_tap_keycode(keycode)) {
        case KC_SPC:
        case KC_A ... KC_Z:
        case KC_DOT:
        case KC_COMM:
        case KC_SCLN:
        case KC_SLSH:
            return true;
    }
    return false;
}
```

Copy the above to your `keymap.c` and edit to customize. For instance, remove the `case KC_SPC` line to disable Flow Tap for the Space key.

### get_flow_tap_term()

Optionally, for further flexibility, define the `get_flow_tap_term()` callback. Flow Tap acts only when key events are closer together than the time returned by the callback. Return a time of 0 to disable filtering. In this way, Flow Tap may be disabled for certain tap-hold keys, or when following certain previous keys.

The default implementation of this callback is

```c
uint16_t get_flow_tap_term(uint16_t keycode, keyrecord_t* record,
                           uint16_t prev_keycode) {
    if (is_flow_tap_key(keycode) && is_flow_tap_key(prev_keycode)) {
        return FLOW_TAP_TERM;
    }
    return 0;
}
```

In this callback, `keycode` and `record` correspond to the current tap-hold key, and `prev_keycode` is the keycode of the previous key. Return the timeout to use. Returning `0` disables Flow Tap. This callback enables setting per-key timeouts. It is also possible to enable or disable Flow Tap for certain tap-hold keys or when following certain previous keys. Example:

```c
uint16_t get_flow_tap_term(uint16_t keycode, keyrecord_t* record, 
                           uint16_t prev_keycode) {
    if (is_flow_tap_key(keycode) && is_flow_tap_key(prev_keycode)) {
        switch (keycode) {
            case LCTL_T(KC_F):
            case RCTL_T(KC_H):
              return FLOW_TAP_TERM - 25;  // Short timeout on these keys.

            default:
              return FLOW_TAP_TERM;  // Longer timeout otherwise.
        }
    }
    return 0;  // Disable Flow Tap.
}
```

::: tip If you define both `is_flow_tap_key()` and `get_flow_tap_term()`, then the latter takes precedence.
:::

## Chordal Hold

Chordal Hold is intended to be used together with either Permissive Hold or Hold
On Other Key Press. Chordal Hold is enabled by adding to your `config.h`:

```c
#define CHORDAL_HOLD
```

Chordal Hold implements, by default, an "opposite hands" rule. Suppose a
tap-hold key is pressed and then, before the tapping term, another key is
pressed. With Chordal Hold, the tap-hold key is settled as tapped if the two
keys are on the same hand.

Otherwise, if the keys are on opposite hands, Chordal Hold introduces no new
behavior. Hold On Other Key Press or Permissive Hold may be used together with
Chordal Hold to configure the behavior in the opposite hands case. With Hold On
Other Key Press, an opposite hands chord is settled immediately as held. Or with
Permissive Hold, an opposite hands chord is settled as held provided the other
key is pressed and released (nested press) before releasing the tap-hold key.

Chordal Hold may be useful to avoid accidental modifier activation with
mod-taps, particularly in rolled keypresses when using home row mods.

Notes:

* Chordal Hold has no effect after the tapping term. 

* Combos are exempt from the opposite hands rule, since "handedness" is
  ill-defined in this case. Even so, Chordal Hold's behavior involving combos
  may be customized through the `get_chordal_hold()` callback.

An example of a sequence that is affected by “chordal hold”: 

- `SFT_T(KC_A)` Down
- `KC_C` Down
- `KC_C` Up
- `SFT_T(KC_A)` Up

```
                         TAPPING_TERM   
  +---------------------------|--------+
  | +----------------------+  |        |
  | | SFT_T(KC_A)          |  |        |
  | +----------------------+  |        |
  |    +--------------+       |        |
  |    | KC_C         |       |        |
  |    +--------------+       |        |
  +---------------------------|--------+
```

If the two keys are on the same hand, then this will produce `ac` with
`SFT_T(KC_A)` settled as tapped the moment that `KC_C` is pressed. 

If the two keys are on opposite hands and the `HOLD_ON_OTHER_KEY_PRESS` option
enabled, this will produce `C` with `SFT_T(KC_A)` settled as held when `KC_C` is
pressed.

Or if the two keys are on opposite hands and the `PERMISSIVE_HOLD` option is
enabled, this will produce `C` with `SFT_T(KC_A)` settled as held when that
`KC_C` is released.

### Chordal Hold Handedness

Determining whether keys are on the same or opposite hands involves defining the
"handedness" of each key position. By default, if nothing is specified,
handedness is guessed based on keyboard geometry.

Handedness may be specified with `chordal_hold_layout`. In keymap.c, define
`chordal_hold_layout` in the following form:

```c
const char chordal_hold_layout[MATRIX_ROWS][MATRIX_COLS] PROGMEM =
    LAYOUT(
        'L', 'L', 'L', 'L', 'L', 'L',  'R', 'R', 'R', 'R', 'R', 'R', 
        'L', 'L', 'L', 'L', 'L', 'L',  'R', 'R', 'R', 'R', 'R', 'R', 
        'L', 'L', 'L', 'L', 'L', 'L',  'R', 'R', 'R', 'R', 'R', 'R', 
                       'L', 'L', 'L',  'R', 'R', 'R'
    );
```

Use the same `LAYOUT` macro as used to define your keymap layers. Each entry is
a character indicating the handedness of one key, either `'L'` for left, `'R'`
for right, or `'*'` to exempt keys from the "opposite hands rule." A key with
`'*'` handedness may settle as held in chords with any other key. This could be
used perhaps on thumb keys or other places where you want to allow same-hand
chords.

Keyboard makers may specify handedness in keyboard.json. Under `"layouts"`,
specify the handedness of a key by adding a `"hand"` field with a value of
either `"L"`, `"R"`, or `"*"`. Note that if `"layouts"` contains multiple
layouts, only the first one is read. For example:

```json
{"matrix": [5, 6], "x": 0, "y": 5.5, "w": 1.25, "hand": "*"},
```

Alternatively, handedness may be defined functionally with
`chordal_hold_handedness()`. For example, in keymap.c define:

```c
char chordal_hold_handedness(keypos_t key) {
    if (key.col == 0 || key.col == MATRIX_COLS - 1) {
        return '*';  // Exempt the outer columns.
    }
    // On split keyboards, typically, the first half of the rows are on the
    // left, and the other half are on the right.
    return key.row < MATRIX_ROWS / 2 ? 'L' : 'R';
}
```

Given the matrix position of a key, the function should return `'L'`, `'R'`, or
`'*'`. Adapt the logic in this function according to the keyboard's matrix.

::: warning
Note the matrix may have irregularities around larger keys, around the edges of
the board, and around thumb clusters. You may find it helpful to use [this
debugging example](faq_debug#which-matrix-position-is-this-keypress) to
correspond physical keys to matrix positions.
:::

::: tip If you define both `chordal_hold_layout[MATRIX_ROWS][MATRIX_COLS]` and
`chordal_hold_handedness(keypos_t key)` for handedness, the latter takes
precedence.
:::


### Per-chord customization

Beyond the per-key configuration possible through handedness, Chordal Hold may
be configured at a *per-chord* granularity for detailed tuning. In keymap.c,
define `get_chordal_hold()`. Returning `true` allows the chord to be held, while
returning `false` settles as tapped.

For example:

```c
bool get_chordal_hold(uint16_t tap_hold_keycode, keyrecord_t* tap_hold_record,
                      uint16_t other_keycode, keyrecord_t* other_record) {
    // Exceptionally allow some one-handed chords for hotkeys.
    switch (tap_hold_keycode) {
        case LCTL_T(KC_Z):
            if (other_keycode == KC_C || other_keycode == KC_V) {
                return true;
            }
            break;

        case RCTL_T(KC_SLSH):
            if (other_keycode == KC_N) {
                return true;
            }
            break;
    }
    // Otherwise defer to the opposite hands rule.
    return get_chordal_hold_default(tap_hold_record, other_record);
}
```

As shown in the last line above, you may use
`get_chordal_hold_default(tap_hold_record, other_record)` to get the default tap
vs. hold decision according to the opposite hands rule.


## Retro Tapping

To enable `retro tapping`, add the following to your `config.h`:

```c
#define RETRO_TAPPING
```

Holding and releasing a dual-function key without pressing another key will result in nothing happening. With retro tapping enabled, releasing the key without pressing another will send the original keycode even if it is outside the tapping term.

For instance, holding and releasing `LT(2, KC_SPC)` without hitting another key will result in nothing happening. With this enabled, it will send `KC_SPC` instead.

```
               TAPPING_TERM
  +-----------------|------------------+
  | +---------------|-------+          |
  | | LT(2, KC_SPC) |       |          |
  | +---------------|-------+          |
  |                 |                  |
  |                 |                  |
  |                 |                  |
  +-----------------|------------------+
```

For more granular control of this feature, you can add the following to your `config.h`:

```c
#define RETRO_TAPPING_PER_KEY
```

You can then add the following function to your keymap:

```c
bool get_retro_tapping(uint16_t keycode, keyrecord_t *record) {
    switch (keycode) {
        case LT(2, KC_SPC):
            return true;
        default:
            return false;
    }
}
```

If the programs you use bind an action to taps of modifier keys (e.g. tapping left GUI to bring up the applications menu or tapping left Alt to focus the menu bar), you may find that using retro-tapping falsely triggers those actions. To counteract this, you can define a `DUMMY_MOD_NEUTRALIZER_KEYCODE` in `config.h` that will get sent in between the register and unregister events of a held mod-tap key. That way, the programs on your computer will no longer interpret the mod suppression induced by retro-tapping as a lone tap of a modifier key and will thus not falsely trigger the undesired action.

Naturally, for this technique to be effective, you must choose a `DUMMY_MOD_NEUTRALIZER_KEYCODE` for which no keyboard shortcuts are bound to. Recommended values are: `KC_RIGHT_CTRL` or `KC_F18`. 
Please note that `DUMMY_MOD_NEUTRALIZER_KEYCODE` must be a basic, unmodified, HID keycode so values like `KC_NO`, `KC_TRANSPARENT` or `KC_PIPE` aka `S(KC_BACKSLASH)` are not permitted.

By default, only left Alt and left GUI are neutralized. If you want to change the list of applicable modifier masks, use the following in your `config.h`:

```c
#define MODS_TO_NEUTRALIZE { <mod_mask_1>, <mod_mask_2>, ... }
```

Examples:

```c
#define DUMMY_MOD_NEUTRALIZER_KEYCODE KC_RIGHT_CTRL

// Neutralize left alt and left GUI (Default value)
#define MODS_TO_NEUTRALIZE { MOD_BIT(KC_LEFT_ALT), MOD_BIT(KC_LEFT_GUI) }

// Neutralize left alt, left GUI, right GUI and left Control+Shift
#define MODS_TO_NEUTRALIZE { MOD_BIT(KC_LEFT_ALT), MOD_BIT(KC_LEFT_GUI), MOD_BIT(KC_RIGHT_GUI), MOD_BIT(KC_LEFT_CTRL)|MOD_BIT(KC_LEFT_SHIFT) }
```

::: warning
Do not use `MOD_xxx` constants like `MOD_LSFT` or `MOD_RALT`, since they're 5-bit packed bit-arrays while `MODS_TO_NEUTRALIZE` expects a list of 8-bit packed bit-arrays. Use `MOD_BIT(<kc>)` or `MOD_MASK_xxx` instead.
:::

### Retro Shift

[Auto Shift,](features/auto_shift) has its own version of `retro tapping` called `retro shift`. It is extremely similar to `retro tapping`, but holding the key past `AUTO_SHIFT_TIMEOUT` results in the value it sends being shifted. Other configurations also affect it differently; see [here](features/auto_shift#retro-shift) for more information.

## Why do we include the key record for the per key functions?

One thing that you may notice is that we include the key record for all of the "per key" functions, and may be wondering why we do that.

Well, it's simple really: customization.  But specifically, it depends on how your keyboard is wired up.  For instance, if each row is actually using a row in the keyboard's matrix, then it may be simpler to use `if (record->event.key.row == 3)` instead of checking a whole bunch of keycodes.  Which is especially good for those people using the Tap Hold type keys on the home row. So you could fine-tune those to not interfere with your normal typing.

## Why are there no `*_kb` or `*_user` functions?!

Unlike many of the other functions here, there isn't a need (or even reason) to have a quantum or keyboard-level function. Only user-level functions are useful here, so no need to mark them as such.
