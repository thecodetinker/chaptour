# CI blocks on automated dependency vulnerability scanning

`security-review` stays a deliberate, human-triggered deeper pass for anything touching auth or user input. Baseline dependency vulnerability scanning (GitHub Dependabot/CodeQL) is a separate, always-on blocking CI check instead — cheap, needs no new infrastructure, and shouldn't depend on someone remembering to run it. Consistent with ADR-0014: "checks pass" is meant to be a real guarantee, not a partial one.
