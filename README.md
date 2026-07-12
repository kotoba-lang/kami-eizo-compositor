# kami-eizo-compositor

**Node-based VFX compositing data model — chroma-key, roto/mask,
2D point tracking, and blend-mode compositing — for kotoba-lang's
`eizo` (video production) domain.** An L3-authoring
[kotoba-lang](https://github.com/kotoba-lang) capability library per
[ADR-2607121400](https://github.com/com-junkawasaki/root/blob/main/90-docs/adr/2607121400-kami-ongaku-eizo-commercial-grade-cljs-stack.md),
analogous to what Nuke/After Effects/Fusion's node graph models
internally.

Portable `.cljc` across JVM / ClojureScript, zero external dependencies.

## Node-graph execution: compatible with, NOT dependent on, comfyui

The ADR calls for reusing `kotoba-lang/comfyui`'s ComfyUI-style
node-graph execution engine rather than building a second graph
executor. This repo does that at the **data-shape level only**:
`kami.eizo.compositor.nodes/node-pack` is a seq of node-type maps in
exactly the shape `comfyui.node/register!` expects (`:type` string,
`:category`, `:inputs`, `:outputs` vector, `:fn`), and
`kami.eizo.compositor.nodes/example-workflow` is a ComfyUI-API-format
workflow (`{"id" {:class_type ... :inputs ...}}`) wiring them together.

**This repo does NOT take `kotoba-lang/comfyui` as a `deps.edn`
dependency.** `comfyui` is GPL-3.0-licensed (it matches upstream
ComfyUI's license). GPL-3.0 is a strong copyleft license — a work that
requires a GPL-3.0 library is generally obligated to also be licensed
GPL-3.0 (or compatible). Pulling that into this Apache-2.0 capability
library would silently change its license terms for everyone who
depends on it, which is a much bigger decision than "which node-graph
engine to reuse" and isn't this repo's call to make unilaterally. So:
the node types and example workflow here are plain data, importable
and registerable into a `comfyui.node/registry` by any **consumer**
that has separately decided to accept comfyui's GPL-3.0 terms — that
choice lives at the app/integration layer, not baked into this
library's own dependency graph.

Consequence: `example-workflow` is illustrative of graph topology and
has not been executed end-to-end through `comfyui.exec/execute` in
this repo (that would require the GPL dependency this repo
deliberately avoids). What's verified here is that every `:class_type`
in the example workflow resolves to a real node type in `node-pack`
(see `nodes-test`), and that every node type's `:fn` produces correct
output when called directly.

## Scope (v0)

- **In scope**: chroma-key alpha computation (distance-from-key-color
  with a smoothstep soft-edge falloff), roto/mask as a straight-edge
  polygon with ray-casting point-in-polygon (correct for non-convex
  shapes), 2D point tracking with linear interpolation between
  samples, standard blend modes (normal/multiply/screen/add) with
  separate opacity mixing, and comfyui-compatible node-type wrappers
  around all of the above.
- **Not in scope**: actual image/video decoding or pixel-buffer
  iteration (`:pixel`/`:top`/`:bottom` inputs are single RGB triples,
  not images — a real compositor would map these over every pixel of
  a decoded frame, which needs `utsushi`/`org-w3-webcodecs`), bezier
  roto handles (polygon vertices are straight-line only), spline or
  Kalman-smoothed tracking (linear interpolation only), GPU execution,
  and actual node-graph execution (see above — that's a consumer-side
  concern once they accept comfyui's license).

## Contract

```clojure
(require '[kami.eizo.compositor.chroma-key :as chroma-key]
         '[kami.eizo.compositor.roto :as roto]
         '[kami.eizo.compositor.tracking :as tracking]
         '[kami.eizo.compositor.blend :as blend]
         '[kami.eizo.compositor.nodes :as nodes])

(chroma-key/key-alpha [0.0 1.0 0.0] {:key-color [0.0 1.0 0.0] :tolerance 0.1 :softness 0.1})
;=> 0.0 (exact key-color match -> fully transparent)

(roto/point-in-polygon? [[0 0] [10 0] [10 5] [5 5] [5 10] [0 10]] [2 8])
;=> true (non-convex L-shape, ray casting)

(def track (tracking/make-track [{:frame 0 :x 0.0 :y 0.0} {:frame 10 :x 100.0 :y 0.0}]))
(tracking/position-at track 5) ;=> [50.0 0.0]

(blend/composite :multiply [1.0 0.5] [0.5 1.0] 0.5) ;=> [0.5 0.75]

nodes/node-pack       ;=> comfyui-compatible node-type maps (data only)
nodes/example-workflow ;=> ComfyUI-API-format workflow wiring them
```

## Naming note

`kami-eizo-*` mirrors the `kami-engine-*`/`kami-mangaka-*`/
`kami-ongaku-*` domain-authority naming convention (ADR-2607121400
§2.1); `eizo` (映像) is the video-production domain, parallel to
`ongaku` (音楽) for music. This library owns compositing node
*types/data*; it is not a compositor *application* and not
`kotoba-lang/comfyui` (the execution engine, consumed only by data
shape, never as a dependency — see above).
