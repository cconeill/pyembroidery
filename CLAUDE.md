# pyembroidery Fork — Modernization Project

## What This Project Is

A fork of [pyembroidery](https://github.com/EmbroidePy/pyembroidery) (v1.5.1), a pure-Python embroidery file I/O library supporting 46 read formats and 20+ write formats. We are modernizing it in phases while maintaining full backward compatibility.

## Key Commands

```bash
# Run all tests (232 tests, should always pass)
python -m unittest discover -s test -t .

# Run a specific test file
python -m unittest test.test_convert_pes

# Run a single test
python -m unittest test.test_embpattern.TestEmbPattern.test_add_stitch
```

## Project Conventions

- **Zero external dependencies** — This is a core design constraint. Do not add runtime dependencies.
- **Backward compatible** — All public API changes must be additive. Never break existing signatures.
- **Tests must pass** — Run the full suite before committing. All 232 tests must pass.
- **Type hints** — Use `from __future__ import annotations` at the top of any modified file. Use Python 3.9+ generic syntax in annotations (e.g., `list[int]` not `List[int]`).
- **No Python 2** — Minimum Python version is 3.8. Remove any `basestring`, `unicode`, or Py2 compat code you encounter.
- **Don't over-document** — Add docstrings to public methods and complex logic only. No boilerplate docstrings on obvious methods.
- **Preserve format behavior** — Format readers/writers encode domain knowledge from reverse-engineering. Change carefully and test round-trips.

## Architecture Overview

```
User API:  read() / write() / convert() / EmbPattern methods
              │
              ▼
EmbPattern:  Central data structure. Holds stitches ([x, y, command] lists),
             threadlist (EmbThread objects), and extras dict.
              │
              ▼
EmbEncoder:  Normalizes patterns for format-specific constraints
             (max stitch length, supported commands, tie-on/off).
              │
              ▼
Readers/Writers:  One module per format (e.g., DstReader.py, PesWriter.py).
                  Use ReadHelper/WriteHelper for binary I/O.
```

**Key files by importance:**
- `pyembroidery/EmbPattern.py` — Core class (1,747 lines, 103 methods). THE God class we're breaking up.
- `pyembroidery/EmbEncoder.py` — Format transcoder (692 lines)
- `pyembroidery/EmbThread.py` — Thread/color management (425 lines)
- `pyembroidery/EmbConstant.py` — Command constants and bit masks
- `pyembroidery/EmbFunctions.py` — Encoding/decoding helpers
- `pyembroidery/ReadHelper.py` / `WriteHelper.py` — Binary I/O utilities
- `pyembroidery/GenericWriter.py` — Configurable text output (CSV, JSON, SVG, TXT)

## Modernization Plan — Progress Tracker

### Phase 1: Modernize Foundation _(Low risk, high value)_

- [ ] Add type hints to `EmbConstant.py`
- [ ] Add type hints to `EmbFunctions.py`
- [ ] Add type hints to `EmbThread.py` + remove `basestring` compat (line ~140)
- [ ] Add type hints to `EmbMatrix.py`
- [ ] Add type hints to `ReadHelper.py` + replace silent `None` returns with logging
- [ ] Add type hints to `WriteHelper.py`
- [ ] Add type hints to `EmbCompress.py`
- [ ] Add type hints to `EmbEncoder.py`
- [ ] Add type hints to `EmbPattern.py` + remove `basestring` compat (line ~1745)
- [ ] Remove `basestring` compat from `GcodeWriter.py`
- [ ] Fix JefWriter color count bug (documented at line 32)
- [ ] Update `setup.py` — set minimum Python to 3.8, remove 3.6/3.7 classifiers

### Phase 2: Improve Architecture _(Medium risk, high value)_

- [ ] Extract I/O methods from `EmbPattern` into a `PatternIO` module
- [ ] Extract query/analysis methods into a `PatternAnalysis` module
- [ ] Add named constants for magic numbers in `DstReader.py` (decode_dx/decode_dy)
- [ ] Add named constants for magic numbers in `PecReader.py`
- [ ] Add named constants for magic numbers in `JefReader.py` / `JefWriter.py`
- [ ] Create `Stitch` namedtuple/dataclass with backward-compatible `__getitem__`/`__iter__`
- [ ] Reduce nesting depth in `GenericWriter.py`
- [ ] Reduce elif chain in `EmbEncoder.py`

### Phase 3: Extend & Harden _(Medium risk, medium value)_

- [ ] Cache `bounds()` calculation with invalidation
- [ ] Migrate `setup.py` to `pyproject.toml`
- [ ] Add `py.typed` marker (PEP 561)
- [ ] Add error-path tests (malformed files, edge cases)
- [ ] Add round-trip property tests for major formats
- [ ] Update CI to test Python 3.9 - 3.13

## Parallelization Guide

Many Phase 1 tasks are independent and can be run as parallel agents with worktree isolation. Here are the safe parallel groups:

**Group A — Independent type-hint modules (no cross-dependencies):**
- `EmbConstant.py`, `EmbMatrix.py`, `EmbCompress.py`, `ReadHelper.py`, `WriteHelper.py`
- These can ALL be done in parallel — they don't import each other.

**Group B — Depends on Group A:**
- `EmbFunctions.py` (imports `EmbConstant`)
- `EmbThread.py` (imports `EmbConstant`, `EmbFunctions`)

**Group C — Depends on Groups A+B:**
- `EmbEncoder.py` (imports `EmbConstant`, `EmbThread`, `EmbFunctions`)
- `EmbPattern.py` (imports almost everything)

**Group D — Independent fixes (can parallel with anything):**
- `JefWriter.py` bug fix
- `GcodeWriter.py` basestring removal
- `setup.py` update

**Rule:** When running parallel agents, use `isolation: "worktree"` so each agent works on its own copy. Merge results sequentially and run the full test suite after each merge.

## Common Pitfalls

- **Stitch commands use bit masking** — The `command` field in `[x, y, command]` encodes command type, thread index, needle index, and order in different bit ranges. See `EmbConstant.py` and `EmbFunctions.py:encode_thread_change()`.
- **`EmbPattern.stitches` is a list of lists, not tuples** — Stitches are mutable `[x, y, cmd]` lists. Any `Stitch` dataclass must remain mutable and support `[]` indexing.
- **Thread changes ≠ color changes** — Some formats use `NEEDLE_SET` instead of `COLOR_CHANGE`. The encoder handles this, but don't assume they're interchangeable.
- **Y-axis is flipped** — Internal convention is +Y down. Format readers/writers flip as needed.
- **Format handlers are the crown jewels** — The readers/writers encode years of reverse-engineering work on proprietary binary formats. Refactor them cautiously and always verify with round-trip tests.
