# Russ Cox method

Use this method when a change creates or alters something other code or stored
data must depend on. The judge rubric owns deductions; this file owns the
contract analysis.

## Review sequence

1. Inventory the contracts: exported functions and types, interfaces, command
   behavior, configuration, wire formats, disk formats, ordering, and error
   semantics.
2. For each changed contract, write one sentence describing what callers can
   rely on today. Separate that promise from incidental implementation detail.
3. Build the compatibility matrix that applies: old caller/new library,
   new caller/old library, old writer/new reader, new writer/old reader, and
   mixed-version processes.
4. Trace unknown fields, missing fields, zero values, version markers,
   ordering, and defaults. Verify that one deterministic rule decides each.
5. When the change moves a fact to a new representation, check whether the old
   one is still written or still read. Two live representations of one fact can
   disagree, and a reader left on the old one silently keeps the old meaning;
   locate that reader before accepting either as retired.
6. Search for every consumer and persisted sample. Check whether a local edit
   has silently changed behavior at a distant boundary.
7. Identify the next plausible evolution and ask whether it can be added
   without reinterpreting existing data or breaking existing callers.
8. Confirm the tests pin the promise rather than the current implementation. When
   the change deletes tests, check each deleted case against the behavior that
   survives: a promise whose only test was removed is no longer pinned.

## Evidence to seek

- A named public or persisted contract and the exact old/new behavior.
- The writer and the reader of each representation when a fact is being moved,
  so a retired form is shown retired rather than assumed so.
- The surviving test for a promise whose original test the change deletes.
- Compatibility tests or fixtures that cross versions, not only round-trip the
  same implementation against itself.
- Deterministic ordering only where a cited consumer or the documented
  contract relies on stable bytes; a contract documented as non-canonical (not
  to be hashed, compared, or persisted for equality) owes none, and adding
  order it disclaims is cost without a promised benefit.
- Explicit ownership of defaulting and version selection rather than ambient
  global or build-time state.

## Stop condition

Stop when the current promise is clear, old and new participants have defined
behavior, and the next extension has somewhere unambiguous to go. Return N/A
when no external or persisted contract changed.
