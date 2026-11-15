# Lofi and Games fork of daisyUI

Branch `lofi-and-games` is upstream daisyUI plus selector changes that stop the browser from
recalculating the style of the whole page on small DOM changes. Nothing looks different: the rewritten
components were compared pixel by pixel against upstream (toggle, input, select, textarea,
floating label, alert, dropdown, fab flower with 0 to 6 actions, with and without main action / close).

## Why

Some `:has()` rules make Chrome treat almost any change as something that could change the match, so
it restyles every element of the page (measured on lofiandgames.com, about 2,300 elements, 4x CPU
throttle: 30 to 50 ms per change, down to 5 to 14 ms with this branch):

| Rule in upstream | Triggered by | Here |
|---|---|---|
| `.dropdown > :not(:has(~ [class*="dropdown-content"])):focus` | any class change | `:has(~ .dropdown-content)` |
| `.alert:has(> :last-child:is(button, .btn, [class*="btn"]), ...)` | any class change | `:is(button, .btn)` |
| `.fab-flower:has(> :nth-child(N)) > :nth-child(M)` | any element added or removed | `> :nth-child(M):nth-last-child(n + N-M+1)` (the same "at least N children", seen from the child) |
| `.fab-flower:not(:has(.fab-main-action, .fab-close)) > :nth-child(M)` | element added or removed, `disabled`, checkbox | `> :nth-child(M):not(.fab-main-action ~ *, .fab-close ~ *)` |
| `.toggle:has(:checked) > :nth-child(2 or 3)` | element added or removed, any checkbox | `> :checked ~ :nth-child(2 or 3)` |
| `.input:has(> input[disabled]) ::placeholder` (also `select`, `textarea`) | any `disabled` change | `> input[disabled]::placeholder` |
| `.floating-label:not(:has(input:placeholder-shown, ...)) ::placeholder` | any element added or removed | only `:focus-within ::placeholder` (a placeholder that isn't shown needs no style) |

The pattern to avoid: `:has()` (or `:not(:has())`) on a parent, followed by a child, a descendant or a
pseudo-element as the subject, when the argument is structural (`:nth-child`), a state
(`:checked`, `[disabled]`) or a substring attribute match (`[class*=...]`). A rule whose subject is the
element that carries `:has()` is fine.

Two of the rewrites are slightly narrower than upstream, and match the documented markup:
- toggle icons: the checkbox must be a child of `.toggle` placed before the icons;
- fab flower: `.fab-main-action` / `.fab-close` must come before the actions.

## Updating from upstream

```bash
git remote add upstream https://github.com/saadeghi/daisyui.git   # once
git fetch upstream --tags
git merge v5.9.0          # the new upstream tag; conflicts can only be in the 8 files under packages/daisyui/src/components
bun install && bun run build
```

Check that nothing brought the problem back: build the site with the new package and run the probes
described in the site's PR (class toggle, element insert, `disabled`, checkbox, then forced style).

## Releasing

The site installs the built package from a release of this fork (the built files are not in git):

```bash
cd packages/daisyui && npm pack
gh release create v5.9.0-lofi.1 daisyui-5.9.0.tgz --repo brunocalou/daisyui --target lofi-and-games --title "5.9.0 + restyle fixes"
```

Then in the site: `"daisyui": "https://github.com/brunocalou/daisyui/releases/download/v5.9.0-lofi.1/daisyui-5.9.0.tgz"`.
