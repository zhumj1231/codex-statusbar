# Composer overlap fix

Version: `1.0.0-rc3.usage-fix2`. Includes the [usage and timing fix](usage-timing-fix.md).

The status strip no longer prefixes speed with a lightning glyph. The compact field label is
`TPS`, avoiding confusion with Codex's own Fast mode indicator.

## Placement changes

- Dock only against a measured Context indicator or model selector. Window/composer corner
  offsets remain diagnostic values, but cannot authorize drawing over the toolbar. While no
  safe reference is available the docked strip hides; it returns when discovery succeeds.
  Manual positioning remains available through the tray.
- An unlabeled image or a generic text-colour CSS class no longer qualifies as the Context
  indicator. Model-menu fallback must be in the right-hand bottom part of the composer.
- Use geometric boundaries for the left control cluster rather than mixing DOM indices with
  indices in a filtered list. Keep the right edge left of the measured model boundary.
- Shorten immediately when space decreases, including while the reference moves. Only expansion
  waits for settling. A previously hidden variant is not used as an array index.
- If the chosen text still does not fit, hide it rather than pushing it right over the selector.
  Screen clamping cannot override either horizontal boundary or the measured toolbar row.
- Re-read the held model rectangle alongside the Context reference so model-label changes trigger
  rediscovery without waiting for the periodic full walk.

## Evidence and validation

The local diagnostic run recorded `window-fallback` with no measured composer/reference, placing
the strip at x=2559..2892. A subsequent read of the visible Work toolbar measured the model button
at x=2634..2878, and the Context icon at x=2604. These were successive observations, not a captured
simultaneous reproduction of the user's screenshot, but show why a fixed window offset is unsafe.

The production self-test passes 353 checks, covering measurement selection, immediate shrinking,
deferred expansion, missing references, insufficient width, screen clamping, and existing token
accounting/timing. Live installation validation must additionally inspect the actual dock geometry.
Arbitrary future Codex accessibility-tree changes and every monitor/DPI transition are not proven
by these checks. No UI automation clicks or Codex restarts are required for the update.
