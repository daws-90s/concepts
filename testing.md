# Testing Types

| Type | What it checks | Owner |
|---|---|---|
| **Unit testing** | The smallest piece of code (a function/method/class) in isolation — all external dependencies mocked. Fast, runs on every commit. | Developers |
| **Component / module / API testing** | One service's own interface (its API), as a black box, against its *real* dependencies (e.g. its own database) but with no other services involved. Confirms the module works correctly on its own. | Testers (SDET) |
| **Integration testing** | Real calls **between** two or more services/modules. Doesn't re-check each module's own logic — proves the wiring between them actually works. | Testers (SDET) |
| **Regression testing** | Broad re-run across the whole system (base flows + edge/negative cases) to catch cases where a new change broke something that used to work. Run before a release/sign-off. | Testers (SDET) |
| **System testing** | The fully integrated, real system exercised end-to-end as an actual user would (often via the UI), validated against overall requirements rather than any one module. | Testers (SDET) |

Developers own unit tests only, since those live next to the code and change with it.
Everything from component/API testing upward is black-box and owned by testers (SDET) —
they test the system from the outside, independent of implementation details.
