---
name: clone-style
description: Clone a UI style from a live reference page into the user's own project, using ground truth from clone-style-source.html (a fixed file at the project root, checked for and created automatically) instead of guesses, memory, or prior hand-typed values. Acquisition goes through the user's Copy CSS Chrome extension, pasted into that file. Use when the user points at clone-style-source.html and an element/class from it and asks to "do"/"apply"/"clone"/"match" that style, or when they give a URL/element and want to recreate its look.
license: MIT
---

# Clone Style From a Reference Page

Some projects deliberately clone another product's production UI (matching a design system, embedding inside a host app, or just wanting pixel-fidelity to a specific reference). The source of truth is a real site/app's actual CSS — never guesses, memory, or prior hand-typed values, which are frequently wrong in small but visible ways (opacity, letter-spacing sign/magnitude, font-size) once actually checked.

## The source file: one fixed name, checked and created automatically

Everything this skill gathers — pasted from the Copy CSS extension's clipboard output — lands in one file: **`clone-style-source.html`, at the project root.** Not `clone.css`, not `figma-clone.css`, not a same-conversation-only scratch file — always this exact name, so a later session (or a later turn in the same one) finds the same accumulated source instead of fragmenting it across differently-named files.

**At the start of any clone-style task**, before asking the user for anything:
```
ls clone-style-source.html
```
- **Exists:** that's the destination. New material gets appended to it (with a `<!-- ... -->` separator noting what was added and when, so the file stays legible as it grows), not overwritten — earlier captures stay available for elements that come up again later.
- **Missing:** create it empty (`touch clone-style-source.html`) *before* saying anything, then tell the user with this exact message:

> We'll need a place for the cloning source code, so I've created an empty `clone-style-source.html` file at the project root.
>
> To clone a style:
> 1. Install the [Copy CSS Chrome extension](https://chromewebstore.google.com/detail/copy-css/pgikmlmndobkjkjbpigbhbbgganbfopg) to grab styles from the target element. You can use other tools too.
> 2. Paste the clipboard into `clone-style-source.html`.
> 3. Run `/clone-style` again, telling me which element in your project it should apply to.

If a differently-named clone-source-shaped file already exists in the repo from before this convention (`clone.css`, a `clone-style/` folder, etc.), don't silently ignore it — ask whether to fold it into `clone-style-source.html` or treat it as superseded, rather than grepping two sources of truth in parallel.

**When the file already exists** and the source values are needed again, give the same numbered steps 1-3 above (without the "I've created..." opener, since nothing was just created) — every time this method is needed, not just the first.

## Input shape

Once acquired, you have:
- An element's `outerHTML`/class list.
- Usually a screenshot for visual reference.
- Which element in **their own project** this should be applied to.
- `clone-style-source.html` itself, checked/created per the step above.

## Procedure

1. **List every class** on the target element (and on child elements if it's a compound style — e.g. a heading + description, a button + its inner spans).

2. **Grep each class in `clone-style-source.html`** for its literal rule:
   ```
   grep -F -n -A3 -- ".some-class-name {" clone-style-source.html
   ```
   Use `grep -F` (fixed-string) — hashed class names commonly contain `--`, `.`, `__`, or other characters that break normal/extended regex. Some classes resolve to a single property, others (component classes) resolve to several properties plus custom-property (`--foo-*`) assignments used elsewhere in the cascade.

3. **Resolve `var(--token)` chains recursively.** A class rule often just points to a custom property (`color: var(--text-secondary)`), which itself points to another, down to a literal (hex color, `rem`/`px` length, unitless number). Keep grepping `-- "--token-name:"` until you hit a literal.

4. **Watch for multiple/theme-scoped definitions of the same token.** Design-system CSS dumps often define the same custom-property names more than once, for different themes or contexts (dark vs light, a colored/overlay surface vs default, a "compact" vs "comfortable" density). Find the block that actually applies by default — usually the plain, unconditional `:root { ... }` — verified by scanning backwards from your match to the nearest such block (not inside a `[data-theme=...]`/media-query/other conditional selector). Cross-check by resolving one value you're already confident about and confirming it matches that block, before trusting the rest of it.

5. **Convert units to match the target project's convention.** Reference CSS is frequently in `rem` (commonly `1rem = 16px`, but confirm against the file's own base font-size if it sets one) while the target project may use `px` or its own custom properties — convert and stay consistent with whatever the target file already does.

6. **Reuse the target project's existing variables where the concept already matches** (its own color tokens, spacing scale, radius scale, type scale) instead of inventing new ones or hardcoding a value that's already expressed by an existing variable. Only fall back to a literal value when the resolved number doesn't match any existing variable, or when exact fidelity specifically matters (e.g. a precise alpha-channel color rather than nudging to a close existing gray). State explicitly when you left an existing variable alone because it already matched — this is the main way the patch stays non-bloated.

7. **Apply only the properties actually needed** to reproduce the target element — this is a targeted patch, not a refactor. Don't restructure unrelated CSS, don't add unrequested properties, don't "clean up" nearby rules while you're in there. If the user only gave you one variant/state (e.g. one button style but not its sibling), touch only that variant and say so plainly, so they know what's still unaddressed.

8. **Structural clues from the real DOM, not just computed styles:**
   - An explicit inline `style="width: ..."` on the reference element usually means a *fixed* size shared across sibling variants (e.g. two buttons with different label lengths staying the same width) — replicate as an explicit `width`/`min-width`, not organic content-sizing.
   - `outline` + negative `outline-offset` is a common trick to draw a border-like line that doesn't add to the box's layout size. If you see that pattern in the source, replicate the mechanism (`outline`), not just the resulting color as a `border`.

## Known traps

- **`[hidden]` can be silently overridden.** If a target element toggles visibility via the `hidden` attribute *and* has a class that also sets `display` (e.g. `display: flex`), an author CSS rule beats the User-Agent's built-in `[hidden] { display: none }` on a specificity tie (author origin wins ties over UA origin). Add an explicit `.your-class[hidden] { display: none; }` whenever you combine `hidden` with a `display`-setting class.
- **Native `<select>` always renders its own selected-option text.** If you want a custom-styled label next to a decorative chevron with a working native dropdown underneath, make the `<select>` an invisible (`opacity: 0`, `position: absolute; inset: 0`) full-cover overlay over the visible text — don't render both a text span and a visible select side by side, or the label duplicates.
- **Equal-specificity ties with interaction states.** A new attribute-selector rule (e.g. `[data-state="x"]` adding a border/outline) can tie in CSS specificity with an existing `:hover`/`:focus-visible` rule on the same element. Check whether your new rule could hide a focus ring or hover state, and if so, reorder (put the accessibility/interaction rule last so it wins the tie) or raise its specificity with a combined selector.
- **A format/loading hint can silently invalidate an entire declaration.** E.g. `@font-face { src: url(...) format('woff2-variations'); }` — if the runtime doesn't recognize that exact format string, the whole `src` is treated as invalid and the font-face fails to register with no visible error, silently falling back to something else that merely *looks* similar (this is how a variable-weight font can still render as a heavier system-font fallback despite being "embedded"). When something is declared but visibly not taking effect, check for exactly this kind of silent-rejection failure mode before assuming the values themselves are wrong.
- **Don't trust hand-typed/remembered values over the source.** Values given verbally earlier in conversation (a color's opacity, a letter-spacing sign/magnitude, a font-size) can turn out to be wrong in small but visible ways once actually resolved from the clone-source file. When there's a conflict, flag it to the user and default to what the source file says.

## After editing

1. Run the target project's normal build/typecheck/lint (whatever it uses) and keep it clean.
2. Follow the project's normal commit workflow rather than inventing a new one.
3. Summarize which classes you resolved, the literal values you found, and flag anywhere the result diverged from a prior assumption or only covers part of what was asked (e.g. one variant given, its sibling not yet specified).
