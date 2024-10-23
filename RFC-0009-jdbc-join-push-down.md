# **RFC0 for Presto**

See [CONTRIBUTING.md](CONTRIBUTING.md) for instructions on creating your RFC and the process surrounding it.

## [Title]

Proposers

*
*

## [Related Issues]

Related issues may include Github issues, PRs or other RFCs.

## Summary

Briefly describe what you intend to propose

## Background

Brief description of any existing issues, user requests or feature comparison with competitors which explains why the proposed feature might be needed. How the proposed feature helps our users? Explain the impact and value of this feature.

### [Optional] Goals

### [Optional] Non-goals

## Proposed Implementation

How do you intend to implement the feature? This section can be as detailed as possible with large subsections of its own, or may be a few sentences depending on the scope of the feature proposed. Explain the design in enough detail for existing users/contributors to understand. Design should include all the corner cases you can think of. Feel free to include any new SPI method signatures, class hierarchies or system contracts here. It is recommended to mention any methods, variables, classes, or SQL language additions which you think are needed to provide a broader view of the code changes. Please mention/describe the below on a high level -

1. What modules are involved
2. Any new terminologies/concepts/SQL language additions
3. Method/class/interface contracts which you deem fit for implementation.
4. Code flow using bullet points or pseudo code as applicable
5. Any new user facing metrics that can be shown on CLI or UI.

## [Optional] Metrics

How can we measure the impact of this feature?

## [Optional] Other Approaches Considered

Based on the discussion, this may need to be updated with feedback from reviewers.

## Adoption Plan

- What impact (if any) will there be on existing users? Are there any new session parameters, configurations, SPI updates, client API updates, or SQL grammar?
- If we are changing behaviour how will we phase out the older behaviour?
- If we need special migration tools, describe them here.
- When will we remove the existing behaviour, if applicable.
- How should this feature be taught to new and existing users? Basically mention if documentation changes/new blog are needed?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

## Test Plan

How do we ensure the feature works as expected? Mention if any functional tests/integration tests are needed. Special mention for product-test changes. If any PoC has been done already, please mention the relevant test results here that you think will bolster your case of getting this RFC approved.