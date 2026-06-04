# poetry_constraints-txt-multi-pin

**What this probe proves:** Mend applies ALL constraints in `constraints.txt` for a Poetry project, not just the first one. Two transitives (urllib3 + idna) are pinned simultaneously; both must land at their constrained versions in Mend's scan output, even though `poetry.lock` resolves them to their unconstrained latest versions.

## Real-world scenario

Corp environments rarely pin just one transitive. Typical `constraints.txt` files lock 3–10 transitives at once for compatibility / security policy. Without this probe, our Poetry suite would only ever exercise the single-pin code path.

## Files

- `pyproject.toml` — Poetry manifest pulling `requests = "2.32.0"` (one direct, pulls 4 transitives)
- `poetry.lock` — **NOT YET GENERATED.** Run `poetry lock` on a real machine. Will reflect Poetry's unconstrained resolution (urllib3 at latest 2.x, idna at latest 3.x). Do NOT hand-craft.
- `constraints.txt` — pins TWO transitives: `urllib3==1.26.18` AND `idna==3.7`
- `whitesource.config` — `python.applyConstraints=true`

## Expected scan behavior (post SCA-5154)

Dep tree contains:
- Direct: `requests` 2.32.0
- Transitive `certifi` 2026.4.22 (unconstrained — resolver picks latest)
- Transitive `charset-normalizer` 3.4.7 (unconstrained)
- Transitive `idna` **3.7** (constrained — resolver picks 3.7 not 3.15)
- Transitive `urllib3` **1.26.18** (constrained — resolver picks 1.26.18 not 2.7.0)

The divergence between `poetry.lock` (urllib3==2.7.0, idna==3.15) and Mend's scan output (urllib3==1.26.18, idna==3.7) is the load-bearing signal — Poetry does NOT read constraints.txt natively, so the divergence proves Mend applied the constraints independently.

## Load-bearing assertion

`expected_dependency_pairs` contains TWO entries (one per constrained dep). If Mend applies only the urllib3 constraint and ignores the idna one (or vice versa), exactly one of the two pair assertions fails — pinpointing exactly which constraint was dropped.

## Why this isn't redundant with poetry_constraints-txt-overlay

`poetry_constraints-txt-overlay` pins one transitive. This probe pins two. Different code-path coverage — exercises the constraint-file-iteration path, not just constraint-application.