# Pending Design Notes

Design questions and research tasks carried over from other repositories or earlier feedback, waiting to be incorporated into the main architecture work.

## Geological Addressing (carried over from concord root 2026-04-08)

**Source:** User feedback in the concord (root Matrix repo) on 2026-04-12, re-routed to this repo on 2026-04-08 with user directive: *"meshes are a strictly concord-beta's domain. That instruction should be put there instead."*

**Question:** Should mesh addresses be tied to the geographical location of the node?

**Original framing:** User wrote that mesh addresses *"could need to be tied to the geological location of the node."* The phrasing is hedged — unclear whether this is a firm requirement or an exploration to evaluate later.

**Context / prior decisions:**
- The existing mesh addressing scheme is HMAC-SHA256 deterministic with 3-level locale partitioning (region/cluster/subnet). See `concord-core` mesh_map module.
- Locale partitioning already gives a rough geographic grouping based on self-reported locale, but it is NOT cryptographically tied to actual physical coordinates.
- Adding a true geohash component would affect: (a) the address derivation path, (b) node mobility (addresses change when a node physically moves), (c) privacy (addresses leak approximate location), (d) the LWW merge protocol for mesh map entries.

**Open questions to answer during design:**
1. Is this a firm requirement for v1 of the mesh addressing scheme, or a v2+ exploration?
2. If firm: what geohash precision? (Country-level leaks less; city-level leaks more but enables useful local discovery.)
3. If firm: how does node mobility interact with a position-bound address? Do addresses rotate on movement? Stay fixed? Both?
4. Privacy trade-off: nodes that want to hide their location must either opt out (compromising mesh efficiency) or lie (compromising mesh integrity). How does the system handle both?
5. Interaction with phantom/disposable nodes, which have no persistent physical anchor.

**Next action:** Evaluate against the existing `mesh_map.rs` locale partitioning — determine whether this is a parameter refinement or a ground-up change. Then present the user with a concrete decision matrix (precision vs privacy vs mobility).

**Blocking:** none. This is research/design work, not implementation.
