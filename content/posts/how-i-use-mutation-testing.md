+++
title = 'How I Use Mutation Testing'
date = 2025-04-12T10:14:05Z
+++
## How I integrated Mutation Testing into my workflow?

When I want to refactor some code my biggest fear is changing the functionality. A strong test suite can protect against this but how do I know if my test suit has all of the possible outcomes covered?

I have recently began using mutation testing as a **Pre-Refactor Safety Net**. I have began leveraging mutation testing for updating the quality of my test suite before touch any of the production code.

My process looks like thus;

1. Choose the package that I want to refactor.
2. Run mutation testing on that module.
   1. OPTIONAL: save this report for comparison later. It can be reassuring to have a solid metric for before and after.
3. If any mutants survive the test suite is not tight enough. Write additional unit tests to cover
   1. Continue this until all mutants are killed or that kill rate is suitably high.
4. Open PR. Something in the realm of  `test(auth): add unit tests for token validation edge cases`
5. After approval merge and make coffee.
6. Open another branch and refactor code.
7. Ensure all Unit Tests pass
8. Run Mutation testing ensure all mutants are killed or that kill rate is suitably high.
9. Open PR. Something in the realm of  `refactor(auth): simplify token validation logic`.

### The outcome

I have found that my Pull Requests have been smaller and more focused with a perceived smaller chance of negative side effects. This has led to less time to receive an Approval. I put this down to the confidence reviewers have in the stability of the change, especially when the unit tests remain unchanged and continue to pass, indicating that existing functionality is preserved."