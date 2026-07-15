# Contributing

We welcome feedback on Singulisp. This project uses its issue tracker as the single channel for bug
reports, suggestions for improvement, and feature requests.

## Pull Requests

This project is not currently accepting pull requests.

This is not because implementation contributions from outside the project are unwelcome. We
currently lack the capacity to review changes responsibly and maintain them afterward. If a pull
request is opened, it may be closed without review.

If you find a bug or have a proposal, please open an issue instead of a pull request. This policy
does not restrict forks or independent modifications.

## Reporting Bugs

If you find a bug, please report it in an issue. If we can confirm the bug, we will investigate its
cause and fix it. We cannot, however, guarantee when the fix will be available.

The following information will help us investigate:

- A minimal code sample that reproduces the bug
- The expected result
- The actual result
- The Singulisp version and execution environment you used

Code snippets, input data, expected results, and similar material posted in an issue may be
consulted while implementing a fix or, after simplification or modification, used in regression
tests. We may not use submitted material as-is, and we may choose not to use it at all.

If you attach code or data, submit only material that you have the right to provide and that the
project is permitted to use and modify under the Apache License 2.0. Do not submit code subject to
third-party rights. Do not include confidential information or data that cannot be made public.

## Feature Requests

Feature requests are also welcome in the issue tracker. You do not need to provide a finished
specification or implementation. A description of what you want to accomplish, what is currently
causing difficulty, and the use case you have in mind is enough.

We prioritize evaluating requests for capabilities that cannot be expressed in Singulisp in
principle, or that cannot be expressed while preserving the required performance characteristics.
If the same result can be achieved by combining existing language features or standard-library
facilities, we may suggest an alternative approach or give the request a lower priority.

Singulisp is designed around execution speed and predictable costs. As a result, even a proposal
that improves convenience may not be accepted if it would degrade performance on existing fast
paths. Designs are easier to evaluate if their costs can be made explicit, or if they can be
selected only when needed without affecting existing execution paths.

Submitting an issue does not guarantee that a proposal will be accepted or implemented. Decisions
about acceptance and concrete design are based on consistency with the language as a whole and the
impact on performance.
