# Passthrough Chroma-Key Void — Code Review Findings (2026-07-19)

Review of the uncommitted `spatial-passthrough` feature branch (xhigh effort: 10 finder angles,
14 adversarial verifiers; the final gap-sweep pass was cut for cost). This is the canonical record;
the June review of the spatial feature itself lives in `spatial-review-backlog.md`.

Overall verdict: the core design held (7 scary candidates refuted with proof — see bottom), but the
2D-generalization step introduced one real regression, and the D3D11 clear path regressed to
per-call view churn. 13 of 15 findings fixed 2026-07-19/20; 2 deliberately kept.

## Correctness

1. **[FIXED] OpenVR spatial regression.** `is_using_screen_capture()`'s `is_openxr()` term turned
   the three OpenVR spatial eye void-clears black — increment 1 had them keyed and user-verified.
   UI offered the toggle on OpenVR with zero effect; OpenVR 2D paid one wasted
   `IsDashboardVisible()` IPC per frame. Fix: `VR::is_chroma_void_active()` (OpenXR → any screen
   capture; OpenVR → spatial) + `is_chroma_pad_active()`; UI hint "OpenVR: applies in Spatial Mode
   only".
2. **[FIXED] Void-clear view churn.** `clear_void_tex` created+destroyed an RTV+SRV per call
   (regressing the cached `m_left/right_eye_rtv` path), QI'd `ID3D11DeviceContext1` per call, and
   its bool return was ignored at 5 sites (persistent failure = frozen ghost frame behind the
   void). Fix: cached `m_context1`, eye clears take the cached RTVs, one void-color fetch per frame
   passed down.
3. **[FIXED] Extreme-compat ring gamma.** Extreme compat leaves the 2D screen views sRGB, so the
   raw-sRGB ring key was double-encoded → brighter, unkeyable ring (masked by pure magenta,
   visible with mid-range keys). Fix: `m_2d_screen_srgb_views` picks the linear key there.
4. **[FIXED] Edge guard vs imageRect.** Guard ring was inset from the TEXTURE edge, but the
   submitted rect is `view_bounds`-cropped; in-app projection overrides (2D mode) crop more than
   the guard width, deleting the ring and re-exposing the FOV-edge key flash. Spatial was immune
   (bounds reset to full). Fix: `chroma::guard_interior` insets inside the view-bounds region.
5. **[KEPT AS-IS] Focus-gate black pops.** The OpenXR `FOCUSED`-only gate can flash the
   passthrough room black on non-dimming overlay transitions (no hysteresis). Verified narrow:
   load-screen flapping refuted (event drain + SYNCHRONIZED frames invisible). Accepted risk;
   correct on SteamVR where VISIBLE = dimmed dashboard.
6. **[FIXED] Unclamped color input.** CTRL+click typed values >255/negative bled into adjacent
   bytes of the packed config color. Fix: clamp before packing.
7. **[FIXED] Tiny-window rect inversion.** Pad/ring rects had no size clamp (sub-8px windows →
   inverted rects). Fix: `chroma::clamped_pad`.

## Cleanup / latent

8. **[FIXED]** Dead `void_clear[4]` locals in both backends' on_frame (the OpenVR-2D dead fetch was
   the wasted IPC).
9. **[FIXED]** chroma_pad predicate duplicated across backends + friend-scoped private access →
   hoisted to the VR predicates.
10. **[FIXED]** Ring/guard math + pad constant duplicated across backends → shared
    `src/mods/vr/ChromaVoid.hpp` (`PAD_PX`, `EDGE_GUARD_PX`, `clamped_pad`, `border_ring`,
    `guard_interior`; NB `(std::min)` parenthesized — windows.h macros).
11. **[FIXED]** `CommandContext::clear_rtv` rects overload duplicated the barrier dance → rects
    folded into the base overload, others forward.
12. **[FIXED]** Three stale "projection layer is black by design" comments → "void by design".
13. **[FIXED]** Dead code: two caller-less `render_srv_to_rtv` overloads deleted; UI reuses
    `get_void_key_color_srgb()` instead of re-implementing the unpack.
14. **[FIXED]** Whitespace-only DirectXTK.cpp/.hpp hunks + stray blank line restored to HEAD.
15. **Latent trio:** (a) [FIXED] slate-quad mouse intersection now maps into the chroma inset
    (was 4px off; zero consumers today) + pre-existing `get_framework_intersect_state()`
    wrong-member bug fixed; (b) [KEPT] void clears write alpha 1 vs old 0 — safer convention for
    opaque layers, only matters to a future alpha-blend path; (c) [FIXED 2026-07-20] orphan
    `shaders/chroma_guard_sprite_ps.fx` + `Compiled/chroma_guard_*` files deleted (user-approved).

## Post-review change (2026-07-20, user decision)

The "Void Edge Guard (px)" slider and `PassthroughVoidEdgeGuard` config were removed — esoteric
edge-case fix, default sufficed in testing. Guard is fixed at `chroma::EDGE_GUARD_PX = 16`.

## Refuted (verified non-issues, do not re-report)

- session_state data race / mid-frame ring-void divergence — everything runs on the present thread
  (`consume_events` → on_frame, single writer).
- Resize-window rect mismatch — ResizeBuffers hook resets the component synchronously before any
  frame can run against stale textures.
- Null-RTV ClearView crash / typeless-swapchain frozen frame / OpenVR eye gamma — unreachable by
  format-case analysis (setup gates + UE typeless RTs + typed-sRGB swapchain requests).
- No-D3D11.1 ring skip — OpenXR requires Win10+, whose immediate contexts always implement
  `ID3D11DeviceContext1`.
- Black-key (0x000000) sentinel collision — provably convergent (gated and active arms paint
  identical texels); maintainability note only.
