+++
title = 'How I Use Mutation Testing'
date = 2025-04-12T10:14:05Z
+++

I have recently began using mutation testing as a **Pre-Refactor Safety Net**.

The process looks like this;

1. Choose the package that I want to refactor.
2. Run mutation testing on that module.
    1. OPTIONAL: save this report for comparison later.
3. If any mutants survive the test suite is not tight enough. Write additional unit tests to cover
    1. Continue this until all mutants are killed or that kill rate is suitably high.
4. Open PR. Something in the realm of  `test(auth): add unit tests for token validation edge cases`
5. After approval merge and make coffee.
6. Open another branch and refactor code.
7. Ensure all Unit Tests pass
8. Run Mutation testing ensure all mutants are killed or that kill rate is suitably high.
9. Open PR. Something in the realm of  `refactor(auth): simplify token validation logic`.

### Why this works so well;

1. It keeps my PRs small and quick to review.
2. It keep them focused on a specific area. The test suite or the logic.
3. Seeing a logical refactor and NO changes to the test suite removes any worry of the refactor altering behaviour.
