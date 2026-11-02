# Acceptance criteria: a repo-owned ELK layout for Mermaid diagrams

Status: **reviewed; all questions answered. No engine code exists yet.** It was written and reviewed in **tiptap-intelligent-updates**, the first consumer, whose diagrams prompted the work; paths such as `scripts/render-diagrams.ts`, `docs/diagrams/mermaid.config.json` and `components.mmd` are in that repository, which this file calls **the consumer**.

Terms: this file says **box** for what Mermaid and ELK call a node (the consumer's glossary reserves "node" for something else), **group** (a Mermaid subgraph), **edge**, **head** and **tail** (the arrow end and the other end), **port** (an attachment point on a box's side) and **diagram writer** (whoever edits a `.mmd`). Option and function names are quoted as Mermaid and ELK spell them.

## What this is for

Mermaid 12.2.0's ELK layout offers one switch for merging edges, `elk.mergeEdges`, applied to every box and group. In the consumer's committed `components.svg` it draws lines running in opposite directions as one trunk: at the bottom of "Edit propagation service", three outgoing edges and the incoming "Reports each new version" share one trunk, and the incoming arrowhead sits where the outgoing lines start, so direction and label are ambiguous. ELK has no option that merges only matching edges; it can do that only through explicit ports, which Mermaid never creates. Group padding is a hard-coded 24px (`SUBGRAPH_PADDING`), so `scripts/render-diagrams.ts` adds 16px afterwards (`padGroups`).

## Decided in review

- **Two passes** to choose port sides: lay out once without merging, read which side each edge end falls on, then lay out again with ports on those sides.
- **Port model:** on each side of a box, edge ends that look the same at that box share one port. They look the same when they have the same marker at that end (arrowhead type, or none) and the same line type. So there is one port for solid lines ending in the default arrow, one for solid lines with no arrowhead, one for dotted lines ending in the default arrow, and so on. A side with only one such group of ends has a single port, centred on the side. (This replaces the in/out model from the second review.)
- **No fork of the whole Mermaid repository.** The layout is the ELK renderer's source, taken at tag `mermaid@12.2.0`, plus only the Mermaid internals it needs.
- **Own repository,** a monorepo, so that a Playwright-based renderer can be packaged beside the layout later. The consumer depends on it.
- **Labels** that cannot be placed cleanly leave their class unmerged, with a warning.
- **Padding:** `elk.groupPadding` is set to 40 in `mermaid.config.json` and `padGroups` is deleted, with one deliberate re-render of the committed SVGs.
- **A reference diagram of our own** lives in this repository, because `version-to-apply.mmd` is a sequence diagram and cannot show the layout unchanged.
- **This repository is about Mermaid,** not about this project: it holds the ELK layout package now and other Mermaid tooling soon (R2). It is hosted on GitHub and named **`learnexperts/mermaid-tools`**; packages are scoped `@learnexperts` (`@learnexperts/mermaid-elk-layout`, later `@learnexperts/mermaid-render`), and the layout registers as `elk-selective` (S6).
- **The repository is public,** so the consumer installs the layout from release tarballs on GitHub, with no token (R3).
- **Per-diagram control is wanted but optional.** v1 provides it for merging on or off through registered layout names (S6); per-diagram padding is not possible and is accepted.
- **Tests use vitest,** the browser-driven ones in their own vitest project. No CI for now.
- **No ADR in the consumer.** Decisions are recorded in this repository; the consumer's code is fixed when the layout becomes available.

## What I checked while writing this

- **[code]** The installed ELK build file exports `render = createCommonLayoutRenderer({...})`. Mermaid calls it as `render(data4Layout, svg, internalHelpers, { algorithm })`. `internalHelpers` carries `getConfig`, `log`, `insertCluster`, `insertEdge`, `insertEdgeLabel`, `insertMarkers`, `insertNode`, `labelHelper`, `positionEdgeLabel` and `interpolateToCurve`. It does not carry `createCommonLayoutRenderer`, `defaultMeasureLayout`, `applyLineJumpsToSvg`, `clusterPaintsTitle` or `markerOffsets`; the ELK build file imports those from other build files. The single-file `mermaid.js` does not export them.
- **[source]** Tag `mermaid@12.2.0` exists on GitHub and its `elk/` directory has the same files as the installed build file (`render.ts`, `elkOptionCatalogue.ts`, `geometry.ts`, `lineHops.ts`, `subgraphFeedbackEdges.ts`, `find-common-ancestor.ts`, `algorithms.ts`, `plugin.ts`, `index.ts`). `render.ts` has `SUBGRAPH_PADDING = 24`, an exported `evenGroupFrames`, and `elk.layered.mergeEdges` set in the same two places. I did not diff it line by line.
- **[source]** The same tag has `packages/mermaid-layout-elk` (`@mermaid-js/layout-elk` 1.0.1, peer `mermaid ^12.0.0`), built from `elk/plugin.ts` as a standalone package. So a loader built outside Mermaid's own bundle is a supported shape. Not checked: how that build gets at the internals above.
- **[code]** `sanitizeDirective` deletes any key that is not in Mermaid's default config. A new `elk.*` key written in a diagram's frontmatter or directive would be removed. Site config passed to `mermaid.initialize` goes through `setSiteConfig`, which does not sanitise; I expect new keys to survive there but have not run it.
- **[code]** A layout name that is not registered silently falls back to `dagre` with a log warning (`getRegisteredLayoutAlgorithm`).
- **[code]** An edge's `@{ ... }` metadata reaches the layout only for `animate`, `animation` and `curve`. Nothing per edge reaches ELK.
- **[inferred]** ELK's default port alignment centres a lone port on its side and spreads several evenly. Not tested.
- `version-to-apply.mmd` is a sequence diagram; it never uses ELK.

## 1. Selective merging

**End keys.** Each edge end has a **key**: the marker drawn at that end (default arrow, open circle `o`, cross `x`, or none) and the edge's line type (solid `--`, dotted `-.-`, thick `==`). Examples at the box an edge touches:

| Edge, at this end | Key |
| --- | --- |
| tail of `A --> B`, at A | no marker, solid |
| head of `A --> B`, at B | arrow, solid |
| either end of `A <--> B` | arrow, solid |
| either end of `A --- B` | no marker, solid |
| head of `A -.-> B`, at B | arrow, dotted |
| head of `A --o B`, at B | circle, solid |

**Ports.** On each side of a box there is one port per key that occurs there. Ends on the same side of a box with the same key share that port, which is what merging means here. Two edges are **matching** when they share a port at either end. That covers both your rules, same source, target and direction, and same side going the same way, and also keeps apart lines that look different. Opposite directions stay apart because a plain tail (no marker) and a plain head (arrow) have different keys.

- **M1. Different keys never share a point; in particular, opposite directions never merge.** On any side of any box, ends with different keys have different attachment points. Check: geometry test on fixtures F2, F6, F7 and F7b and on the reference diagram. For every box side, attachment points of different keys differ by at least 8px (initial value, tune in the spike), and no two lines with different keys at that end overlap along a stretch longer than 1px.
- **M2. One port per key per side; a lone one centred.** A side with ends of one key has one attachment point, at its centre (within 1px). A side with ends of *n* keys has exactly *n*, spread along the side. Check: geometry test; count distinct attachment points per box side in every fixture against the keys there, and measure the single one's offset from the side's centre.
- **M3. Same side, same key: merged.** All ends with one key on one side of a box share one attachment point. Check: fixtures F4 (fan-out), F5 (fan-in) and F7 (`<-->` heads sharing with plain heads, `---` ends sharing with plain tails), and the reference diagram; end points within 1px of each other.
- **M4. Parallel edges merge at both ends.** Edges with the same from-box, to-box and direction use the same side at each end, so they share both ports. The second pass puts the whole class on the side most of its ends took in the first pass, ties broken by order in the `.mmd`. Check: fixture F3.
- **M5. The side is the side drawn.** Ports are placed on the sides the first pass produced, not on sides derived from the layout direction. Check: unit test of the side-reading function on layout results with known geometry; the geometry test on all fixtures, including F12 (`LR`, `RL`, `BT`), finds no two ends with one key on a side of a box without a shared attachment point, unless M7 applies.
- **M6. The key is what is drawn at the end.** Only the marker at that end and the line type count; the marker at the far end, the label and the far box do not. Edges given their own look by `linkStyle` or a class are keyed by that style as well, so a red line never merges with a grey one (proposed; it follows from "matching lines"). Check: unit test of the key function over every Mermaid edge form; fixtures F7 and F7b.
- **M7. Unsupported cases stay unmerged and are drawn as stock ELK draws them:** self-loops, invisible links (`~~~`), and edges whose from-box or to-box is a group. Check: fixture F9 draws without error and with no shared attachment points.
- **M8. Edges across group borders merge like any others,** including between a box inside a group and one outside it, and in nested groups. The reference diagram has them, as `components.mmd` does. Check: fixture F8 and the reference diagram; each group frame is still drawn around its boxes and no line is drawn inside another box.
- **M9. Merged lines are drawn.** Every edge keeps its own arrowhead, path, style and DOM id as in stock ELK; the SVG has as many edge paths as the diagram has edges. Check: count edge elements in each fixture's SVG against its `.mmd`.
- **M10. Global merging is off when selective merging is on.** With selective merging enabled, `elk.layered.mergeEdges` is not true for any box or group (it merges opposite directions). Check: unit test on the ELK graph the layout builds; M1 on fixture F2 proves the effect.

### Edge labels on merged edges

- **L1. Every edge keeps its own label,** verbatim, with its own protocol line. Labels are never joined, shortened or dropped. Check: label text in the SVG equals the label text in the `.mmd`, edge by edge.
- **L2. A label sits on its edge's own stretch,** between the point where its line leaves a shared trunk and its other end; never on a stretch used by two or more edges. Check: geometry test; for each label box, count the edge paths that cross it. Exactly one, its own.
- **L3. Labels do not collide.** No label box overlaps another label box (gap of at least 4px, the gap Mermaid's `separateOppositeEdgeLabels` already keeps) or any box. Check: geometry test on all fixtures.
- **L4. If L2 and L3 cannot both hold for a port's edges, those edges are left unmerged** (each on its own port, as stock ELK would place them) and the render prints a warning naming them. The render does not fail. Check: fixture F10 (long labels on a parallel pair in a tight space) renders unmerged with the warning.

## 2. Group padding

- **P1. Padding is a setting.** One number in px, `elk.groupPadding`, default 24 (today's value), applies to every group. Check: unit test of the config reader; a value below 0, or not a number, makes the render fail, naming the key.
- **P2. One value feeds both ELK's own padding and the post-layout trim.** `evenGroupFrames` shrinks each group to its children plus `SUBGRAPH_PADDING`; ELK's `elk.padding` uses the same constant. Both read the new setting, so the trim can neither undo a larger padding nor leave a smaller one loose. Check: render fixture F11 at 24 and at 40; in both, the gap from a group's frame to the bounding box of its boxes and the lines inside it equals the setting on the left, right and bottom (within 0.5px), except where the group's title is wider than its content and widens the frame.
- **P3. The top gap changes by the same amount.** From 24 to 40, the gap above a group's contents grows by exactly 16px. Check: differential test on F11 and the reference diagram.
- **P4. Spacing between groups and between boxes does not depend on the padding setting,** as today (ELK's 50px). Check: same differential test; those gaps are equal at 24 and 40.
- **P5. `padGroups` goes away.** Once `docs/diagrams/mermaid.config.json` sets `elk.groupPadding` to 40 (today's 24 + 16), `padGroups` and its `groupPadding` constant are deleted from `scripts/render-diagrams.ts`, and the viewBox is not edited after layout. Until then it stays, and runs only for diagrams not laid out by the new layout, so padding is never added twice. Check: grep for `padGroups` after migration; a group's title stays inside its frame at a fixed offset from the top (visual check, V3).
- **P6. The migration is a deliberate re-render.** Layout with padding 40 will not reproduce the committed SVGs byte for byte (box positions shift). The criterion is the visual one in V3 plus P2 to P4. Check: you review both renderings of `components.svg` before they are committed.

## 3. Turning it on, and where settings live

- **S1. Selection.** `"layout": "<name>"` in `docs/diagrams/mermaid.config.json` selects the layout for all flowcharts. A diagram can opt in alone with `config: layout: <name>` in its frontmatter (`layout` is an existing key, so it is not sanitised away). Check: fixture rendered both ways.
- **S2. Settings are site-wide config keys,** beside the existing `elk` block: `elk.mergeMatching` (boolean, default `false`) and `elk.groupPadding` (number, default 24), in `mermaid.config.json`. Check: unit test of the reader; each key's default, and a wrong type fails the render with the key and value in the message.
- **S3. A key set per diagram is refused, not lost.** Mermaid removes unknown keys from frontmatter before the layout sees them. The renderer, which owns the config file, warns if a diagram's frontmatter contains `elk.mergeMatching` or `elk.groupPadding`. Check: a fixture with such frontmatter produces the warning.
- **S4. With neither key set, the layout is stock ELK,** including honouring `elk.mergeEdges` as a boolean. See N1.
- **S5. A failed registration fails the render.** If the configured layout name is not registered, the renderer exits non-zero with a message, rather than letting Mermaid fall back to dagre. Check: remove the registration call; the command exits 1 and writes no SVG.
- **S6. Per-diagram merging switch, through layout names.** The layout registers three names that share one implementation, each with its own loader: `<name>` takes `elk.mergeMatching` from config; `<name>-merge` always merges selectively; `<name>-plain` never does (stock merging, per `elk.mergeEdges`). A diagram picks one with `config: layout: <name>-plain` in its frontmatter, whatever the site config says. Padding stays site-wide: a number cannot ride in a layout name, and Mermaid removes new keys from frontmatter. Check: one fixture rendered under each name with `elk.mergeMatching` both `true` and `false` in the site config; `-merge` and `-plain` give the same SVG in both cases, `<name>` follows the config. The unregistered-name check (S5) covers all three.

Ways to carry the choice per edge or per box, since flowchart syntax has no ports:

| Option | Reaches the layout? | Recommendation |
| --- | --- | --- |
| **A. Automatic.** The layout forms ports from the graph (part 1) and two site-wide keys switch it. | Yes | **Recommended for v1.** No new syntax; every diagram benefits; nothing to forget. |
| B. Per-edge metadata, `e1@{ merge: false }`. | No. Mermaid's flowchart database reads only `animate`, `animation` and `curve`, and the layout receives a fixed field list. Needs a fork of Mermaid's parser and database. | Rule out. |
| C. Hints through classes: a class on an edge or box (`class e1 keep-apart`) that the layout reads as "do not merge". | Probably: edges and boxes carry their class names to the layout. Not checked. | Keep in reserve; build only if A produces a wrong merge a diagram writer cannot avoid. |
| D. Several registered names sharing one implementation (`<name>`, `<name>-merge`, `<name>-plain`), chosen per diagram in frontmatter. Each name has its own loader, which hands the shared `render` its mode. | Yes. (The `algorithm` string is already ELK's algorithm id, so the mode rides in the loader, not there.) | **Included in v1 as S6**, since you want per-diagram control. Merging only; not padding. |

## 4. Non-regression

"Options off" means: layout `<name>` selected, neither `elk.mergeMatching` nor `elk.groupPadding` set, `elk.mergeEdges: true` as in today's config, and the renderer's `padGroups` still active.

The **reference diagram** is a flowchart in this repository, modelled on `components.mmd` with generic names and in the same C4 label style: two groups, one nested; boxes inside and outside groups; edges across group borders; an opposite pair; fan-out and fan-in; a parallel pair; labels with protocol lines; one `<-->` edge. It is committed with its light and dark SVG, rendered with stock ELK, as the reference. (Proposed contents; the diagram is written in step 2.)

- **N1. The new layout equals stock ELK when its options are off.** Rendering the reference diagram and every fixture with `layout: elk` and with `layout: <name>` gives byte-identical SVG, for the light and the dark rendering. Check: a script renders each both ways in the same Chromium run and compares the strings; light and dark are reported separately.
- **N2. The committed reference SVG reproduces.** With options off, rendering the reference diagram matches its committed SVG except line 1 (the stamp). Check: `git diff` after a render shows one changed line. The stamp changes whenever the renderer or the layout changes, which is why it is exempt.
- **N3. Deterministic.** The same source rendered three times in separate browser tabs is byte-identical. Without this, N1 and N2 mean nothing. Check: the harness does it.
- **N4. Machine caveat.** The renderer's header says layout can differ slightly between machines and browser versions. N1 and N3 compare within one run and always hold; N2 compares against files made on another machine and is checked on the machine that made them. A mismatch on a second machine is reported, not treated as a defect in the layout.
- **N5. Adoption in the consumer.** Before the new layout is switched on in the consumer, with options off and the consumer's config, `pnpm diagrams` regenerates `docs/diagrams/components.svg` and `docs/diagrams/version-to-apply.svg` identical to the committed files except line 1. `version-to-apply.svg` shows only that sequence diagrams are untouched; `components.svg` shows the layout is unchanged on a real diagram. Check: `git diff` shows one changed line per file.

## 5. Registration, packaging and this repository

- **R1. Registration.** The renderer loads the layout's built script in its browser tab after `mermaid.js`, and calls `mermaid.registerLayoutLoaders(layouts)` before `mermaid.initialize`. The loader returns a module whose `render` has the signature Mermaid calls (`data4Layout, svg, helpers, { algorithm }`). Check: S5, and N1.
- **R2. A separate monorepo, about Mermaid.** This repository, `mermaid-tools`, is named for Mermaid, not for this project. It holds, as separate packages, the layout (source, tests, fixtures, reference diagram, README) and soon other Mermaid tooling, among it the Playwright-based renderer extracted from `scripts/render-diagrams.ts`. Until the renderer is extracted, the consumer's script stays as the renderer and consumes the layout. Check: a clean clone of this repository installs, builds, and passes every check in part 6 without the consumer.
- **R3. How the consumer gets it.** As a versioned dependency in `package.json`, with the built script in the installed package; the renderer serves it to its browser tab from `node_modules`. Recommended: each release of the layout package is a `.tgz` attached to a GitHub Release of this repository, and the consumer depends on that URL (`pnpm add https://github.com/learnexperts/mermaid-tools/releases/download/…/….tgz`). It carries the built script, needs no registry, and needs no token, since the repo is public (decided). A git dependency is not recommended: the built script is not committed, so installing would mean building in the consumer. Check: in the consumer, `pnpm install` on a clean checkout and `pnpm diagrams` work with no step in this repository's tree.
- **R4. It builds into one browser-loadable script,** `dist/elk-layout.js`, with elkjs, d3 and the Mermaid internals it needs inlined; no network at build or render time. Build tool: this repository's choice, with Vite library mode the default candidate. The built script ships in the package. Check: a clean clone builds it, and a browser tab loads it with no network and no console errors.
- **R5. The SVG stamp covers the layout.** `stampFor` in the renderer also hashes the installed layout package's built script (and its version), so a new layout version makes every SVG out of date. `pnpm diagrams:check` reads it from `node_modules` and needs no build. Check: bump the layout version in a scratch install; `pnpm diagrams:check` fails naming the SVGs; revert; it passes.
- **R6. Version lock.** The layout package records the Mermaid release it is based on and declares an exact `mermaid` peer dependency. The renderer fails with a clear message if the installed `mermaid` differs. Check: change the pin; the command fails with both versions in the message.
- **R7. The fork is reviewable against upstream.** The vendored files record the upstream tag and commit; every deviation sits in a region marked `FORK:`; a script prints the difference against the pinned upstream tag. Mermaid's MIT licence notice is kept with the vendored files. Check: the script's output shows only `FORK:` regions.
- **R8. The vendored set stays small.** The layout takes only the ELK renderer's source and the Mermaid internals it imports. If the spike finds that closure is most of Mermaid's `src/`, work stops and reports; it does not fall back to forking the whole repository. Check: the spike's file count and line count are recorded in the README.
- **R9. Cost of a Mermaid upgrade is written down and bounded.** The README lists the steps: change the pin; diff the upstream `elk/` directory between the two tags; re-apply the `FORK:` regions; rebuild; run every check in part 6; re-render the reference diagram and the consumer's SVGs and review them. The spike also compares `elk/` between tags 12.1.0 and 12.2.0 to size typical churn, and the README quotes the result (files and lines changed). Expected: small if the directory and the helpers' shape are unchanged; larger if upstream rewrites the renderer or the `internalHelpers`.

## 6. Verification

Fixtures live in this repository's layout package, none under the consumer's `docs/` (its renderer turns every `docs/**/*.mmd` into a committed SVG). Each is a small `.mmd`:

| Id | Shows | Proves |
| --- | --- | --- |
| F1 | the reference diagram | M1 to M3, M8, L, P, N |
| F2 | A to B and B to A, with labels | M1, M10 |
| F3 | A to B twice, same direction, different labels | M4, L2 |
| F4 / F5 | one box fanning out to three; three fanning in | M2, M3 |
| F6 | a box with edges out and in on the same side | M1, M2 |
| F7 / F7b | `<-->` and `---` edges beside plain ones; ends with every marker (arrow, `o`, `x`, none) and line type (solid, dotted, thick), plus a `linkStyle` override, on one side of a box | M1, M3, M6 |
| F8 | edges across group borders; nested groups | M8, P |
| F9 | self-loop; edge to a group | M7 |
| F10 | parallel pair with long labels in a tight space | L3, L4 |
| F11 | a group whose title is wider than its content | P2 |
| F12 | F2, F4 and F6 again in `LR`, `RL` and `BT` | M5 in all four layout directions |

Checks:

- **V1. Unit tests** (vitest, `unit` project, no browser): side reading and port assignment (M2 to M7), config reading (P1, S2), the ELK graph builder's options (M10).
- **V2. Geometry tests** (vitest, separate `browser` project, which drives Playwright's Chromium as a library): render each fixture, parse edge paths, label boxes and group frames from the SVG, and assert M1 to M4, M9, L1 to L3, P2 to P4, S6. Kept apart so unit runs stay fast. Thresholds are starting values to tune in the spike.
- **V3. Visual check.** Render the reference diagram, and at adoption the consumer's `components.mmd`, in light and dark at 1x and 2x, before (stock ELK) and after. A reviewer checks, side by side:
  1. Where an edge comes in and others go out on one side of a box (the service box in `components.mmd`), the incoming edge's arrowhead has its own port, apart from the outgoing edges.
  2. A side with ends of one key has one port, centred; ends that look different at a box never share a point.
  3. Edges that leave a box on one side share a trunk, and each label sits on its own stretch.
  4. At "TipTap editor" in `components.mmd`, the plain edge leaving for the dev server and the `<-->` edge to the collab server (an arrowhead at the editor) are on different ports, and nothing looks ambiguous.
  5. No label covers a line that is not its own; none overlaps another.
  6. Groups have the intended padding on all four sides, their titles inside the frame.
  7. Nothing else moved without reason.
  I run this and attach the screenshots; **you sign off** before any SVG is committed.
- **V4. Non-regression harness:** N1, N2, N3 in one command, run before every commit that touches the layout. Failing N1 blocks the commit.
- **V5. Failure modes:** S5 (no registration), R6 (wrong Mermaid), R5 (stale stamp), L4 (label fallback) each have a test that provokes them.
- **V6. Reported, not gated:** render time of the reference diagram per scheme, stock against new; size of `dist/elk-layout.js`.

## 7. Out of scope

- Other diagram types, and ELK algorithms other than `elk.layered` (stress, force, mrtree, box and the others stay as Mermaid ships them).
- Merging edges to or from groups, and any per-edge or per-box syntax (options B and C), unless A proves insufficient.
- Changes to Mermaid's parser or flowchart database, theming, fonts, or colours.
- Forking the whole Mermaid repository.
- Libavoid and hyperedge routing; replacing Mermaid or driving elkjs directly.
- Extracting the renderer into this repository (a later step; this file only keeps the door open, R2), and publishing to a public registry.
- Sending the changes upstream to Mermaid. (Keeping the `FORK:` regions clean leaves that open.)
- Changing the look of today's committed SVGs beyond what P6 and V3 describe.

## 8. Open questions

None. Every question from the reviews is answered above.

## 9. Risks

- **Internals not public.** Beyond what `internalHelpers` carries, the layout needs Mermaid's `createCommonLayoutRenderer`, measurement, line-hop and marker code, none of it in the single-file bundle. The vendored set may be large. Mitigation: spike first (step 1), and R8's stop rule.
- **Ports in a layout with groups.** ELK's hierarchical layout with ports on boxes inside groups, fixed sides and `INCLUDE_CHILDREN` may route worse or fail where plain merging worked. Not tested. Mitigation: try F8 and the reference diagram early.
- **Shared ports may not draw a trunk.** I expect ELK to route edges on a shared port as one hyperedge **[inferred]**; if it only overlaps them, M3 holds visually but L2 gets harder. Mitigation: check on F4 before building anything else.
- **The second pass can move things.** Fixing sides changes the layout, so edges the first pass placed neatly may cross more in the second. Mitigation: V3 on the reference diagram; if bad, fall back per class as in L4.
- **Ports crowd small sides.** Several keys on one side means several ports, each needing room; on a narrow box ELK may widen it or route awkwardly. Mitigation: F7 puts many keys on one side; V3 judges the look.
- **Label placement.** ELK places a label at a mid-edge point, possibly on the shared stretch; Mermaid already moves labels after layout in places, so a further step is likely.
- **Silent fallback to dagre** if registration breaks; covered by S5.
- **Per-diagram numeric settings cannot be carried** (sanitised away); merging on or off can, through layout names (S6). Per-diagram padding is accepted as lost.
- **Version coupling.** The fork depends on `data4Layout`'s shape and `internalHelpers`, both internal. A Mermaid bump can break it without a type error. Covered by R6 and R9.
- **Upstream churn.** The 12.2.0 ELK renderer is about 3,000 lines and still changing quickly; every deviation we keep is a cost at each upgrade. Keep the forked regions small.
- **Two repositories drift.** The consumer's renderer and the layout are versioned apart; a layout bump changes every SVG. Covered by R5 and R6.
- **Byte comparisons across machines** are fragile (N4).

## 10. Recommended order of work

0. ~~Create the public repository `learnexperts/mermaid-tools`.~~ Finished.
1. **Spike, stock only,** in this repository. Vendor the 12.2.0 ELK renderer and the internals it imports with no changes, build it to `dist/elk-layout.js`, register it under the new name, render the consumer's two committed diagrams through it. Prove N1, N3, N5, R1, R4, S5. Measure the import closure (R8) and diff `elk/` between 12.1.0 and 12.2.0 (R9). Nothing in the layout changes yet.
2. **Harness, fixtures and the reference diagram.** F1 to F12, the geometry checker, the unit-test setup, the stock reference renderings; write the criteria as failing tests.
3. **Padding setting** (P1 to P4). Smallest and independent.
4. **Merging spike on F2 and F4:** ports with one pass and direction-derived sides, to learn what ELK does with shared ports before the second pass is built.
5. **Two-pass merging** (M1 to M10), then labels (L1 to L4), then F8 and F1.
6. **Adoption in the consumer:** add the dependency, run N5, switch `mermaid.config.json` (layout name, `elk.mergeMatching`, `elk.groupPadding: 40`), delete `padGroups` (P5), re-render, V3 sign-off by you, commit.
7. **Docs and records:** the layout README with the upgrade steps (R9); correct the consumer's `docs/diagrams/README.md`, which still says merging is `elk.mergeEdges` and padding is added by the script; write the ADRs below in this repository.

Each step ends with V4 green.

## 11. Decisions to record

All go in this repository's `docs/adr/`, numbered from 0001; none are written yet. The consumer gets no ADR: it adopts the layout by changing its config and script when the layout is available.

1. **Own the ELK layout by forking Mermaid's renderer's source, vendored, without forking the whole repo,** rather than patching Mermaid in `node_modules`, waiting for upstream, hand-drawn junction boxes, or driving elkjs directly. Hard to reverse: each Mermaid upgrade costs more the longer it stands.
2. **Ports: one per end key (marker at that end and line type) per side, centred when alone, sides chosen by a first pass** (part 1). Changing it later changes every diagram.
3. **Packaging and version pinning:** vendored source built to one script in its own monorepo, exact Mermaid release lock (R6), how it is consumed (R3).
4. **Settings surface:** two site-wide config keys, layout names for the per-diagram merging switch, and no flowchart syntax, because Mermaid strips unknown frontmatter keys and its database passes nothing per edge to the layout.
5. **Group padding has one source of truth in the layout;** consumers drop their post-layout padding.
