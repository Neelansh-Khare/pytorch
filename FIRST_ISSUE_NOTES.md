# First-issue triage — pytorch/pytorch

Working notes for picking a first contribution. Personal branch; not intended for upstream.

Verified against `pytorch/pytorch@main` on 2026-08-20.

---

## Recommendation: [#130958 — Document `ProcessGroupOptions`](https://github.com/pytorch/pytorch/issues/130958)

**Labels:** `oncall: distributed`, `module: docs`, `triaged`, `good first issue`
**Opened:** 2024-07-16 · **Unclaimed** — the only comment is an unrelated request for beginner issues.

### The defect

Six docstrings in `torch/distributed/distributed_c10d.py` document a parameter with the type
`ProcessGroupOptions`:

| Line | Function |
|------|----------|
| 2439 | `init_process_group` |
| 6724 | `split_group` |
| 7006 | `new_group` |
| 7281 | `new_subgroups` |
| 7381 | `new_subgroups_by_enumeration` |
| 7548 | `shrink_group` |

`ProcessGroupOptions` does not exist. Repo-wide code search returns exactly **one** file —
`distributed_c10d.py` itself, i.e. only these docstring mentions. There is no class, no type alias,
no stub, and **zero** occurrences anywhere under `docs/`.

So every one of these docstrings names a type the reader cannot look up. The real types are
backend-specific and unrelated in name:

- `ProcessGroupNCCL.Options`
- `ProcessGroupGloo.Options`

The issue body puts it plainly: *"It is referenced in the docs but not defined anywhere."*

### Why this one

- **It cannot be secretly fixed already.** Unlike the runner-ups below, the gap is verifiable in
  seconds by grep, and it is still open right now.
- **Genuinely unclaimed.** No PR, no "can I take this", no maintainer steering someone else to it.
- **Bounded blast radius.** Docstrings and `.rst` — no C++ build, no distributed test harness, no
  multi-GPU runner required to validate. Iteration is fast.
- **Reads the same code as the Ray work.** Process-group construction, backend options, and rendezvous
  are the same conceptual territory as node identity and transport setup in
  [ray-project/ray#64350](https://github.com/ray-project/ray/pull/64350).

### Scope question to settle first

The referenced [PR #130957](https://github.com/pytorch/pytorch/pull/130957) ("Fix pyi annotation for
`ProcessGroupNCCL.Options`") was **closed, not merged** — so the typing side of this was never
resolved. That leaves two possible readings, and it's worth asking in the issue before writing code:

1. **Documentation-only** — replace the phantom `ProcessGroupOptions` in all six docstrings with the
   concrete backend types, and add a short section to the distributed docs describing what
   `pg_options` accepts per backend. Smallest, safest, almost certainly what the label intends.
2. **Introduce the symbol** — define a real `ProcessGroupOptions` base/alias so the docstrings become
   accurate as written. Larger surface, touches `.pyi` stubs, needs maintainer buy-in.

Default to (1). Propose it in a comment, note that (2) is the alternative, and let a maintainer choose.
Asking a crisp scoping question on a two-year-old docs issue is itself a good first interaction.

### Plan of attack

1. Comment on the issue: state the grep evidence, propose scope (1), ask whether they'd prefer (2).
2. Fix the six docstring sites to name `ProcessGroupNCCL.Options` / `ProcessGroupGloo.Options`.
3. Add the missing prose to the distributed docs so `pg_options` has somewhere to point.
4. Build the docs locally to confirm no Sphinx reference warnings.
5. Open the PR with `Fixes #130958`, sign off commits (`git commit -s`) for DCO.

---

## Runner-ups — both verified STALE, do not take

### [#85234](https://github.com/pytorch/pytorch/issues/85234) — `all_gather` SIGFPE when `numel() == 0`

**Already fixed upstream.** The crash was integer division by zero at `% outBytes` in gloo's
`allgather.cc`. Gloo added the guard in `d96897b33` ("fix allgather for empty buffers", 2023-05-10):

```cpp
// Short circuit if there is only a single process or the output is empty.
if (context->size == 1 || outBytes == 0) {
  return;
}
```

PyTorch pins `third_party/gloo` at `74cc005ae13f69c11d8a41e50b42025b6730e796` (2026-06-10), which is
**127 commits ahead of the fix and 0 behind**. The guard is present in the pinned tree. The repro in
the issue cannot still SIGFPE.

*Residual value:* run the repro to confirm, then comment with the evidence and ask a maintainer to
close. A regression test for empty-tensor `all_gather` on gloo would be a legitimate small PR, but
it's a test-only contribution, not the bug fix the issue advertises.

### [#74602](https://github.com/pytorch/pytorch/issues/74602) — `my_rank` in `scatter_object_list` should be global

**Already fixed.** Current `scatter_object_list` does:

```python
group_src = _canonicalize_group_rank(group, src, group_src, return_global=False)
...
my_group_rank = group.rank()
if my_group_rank == group_src:
```

Both sides of the comparison are now group-relative, which is what the 2022 report asked for. The
maintainers requested a repro at the time and never received one, so it simply sat open through the
refactor that fixed it.

---

## Also checked — claimed or contested, skip

| Issue | State |
|---|---|
| [#191394](https://github.com/pytorch/pytorch/issues/191394) FileStore fd leak | PR [#191425](https://github.com/pytorch/pytorch/pull/191425) already open |
| [#191395](https://github.com/pytorch/pytorch/issues/191395) etcd `find_free_port` | Contested; `d4l3k` said they prefer the existing PR |
| [#191397](https://github.com/pytorch/pytorch/issues/191397) MemoryTracker op count | Claimed in comments |
| [#78842](https://github.com/pytorch/pytorch/issues/78842) `TORCH_SHOW_CPP_STACKTRACES` | Claimed; contributor already traced the init path |
| [#121587](https://github.com/pytorch/pytorch/issues/121587) `torch.distributed` vs `.nn` docs | Mid-scoping with maintainers |
| [#108744](https://github.com/pytorch/pytorch/issues/108744) `MultithreadTestCase` | Partially claimed across several PRs |
| [#192874](https://github.com/pytorch/pytorch/issues/192874) Set-family VariableTracker | Open umbrella — maintainer says no permission needed, but 20 comments of contributors colliding over the same VTs |

---

## Method note

The `good first issue` label alone is a weak signal in this repo: 54 open, 42 nominally unassigned,
but assignment is rarely used and several of the unassigned ones already have PRs. Two of the three
oldest-looking "real bug" candidates turned out to be fixed years ago and never closed.

Check before investing:

1. **Is it claimed?** Read the comments, not the assignee field.
2. **Is it still real?** Reproduce it, or read the current source. Age correlates with staleness.
3. **For submodule bugs, check the pin.** Compare the fix commit against the SHA in
   `third_party/<name>`, not against the submodule's `main`.

Further hunting grounds beyond the label: `module: bootcamp`, `pt_distributed_rampup`, `actionable`.
