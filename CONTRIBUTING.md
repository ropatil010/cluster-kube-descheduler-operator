# Contributing to the OpenShift Kube Descheduler Operator

This document serves as a guide for contributing to the OpenShift Kube Descheduler Operator, maintained by the OpenShift Control Plane group.

The OpenShift Kube Descheduler Operator manages the Kubernetes Descheduler in OpenShift clusters, evicting pods based on descheduling strategies to optimize cluster resource utilization. The descheduler evicts pods; the scheduler then decides where to reschedule them.

This document is explicitly for contributions to this component repository and not for high-level feature proposals within OpenShift.

Feature proposals should follow the OpenShift Enhancement Proposal process outlined in https://github.com/openshift/enhancements/blob/master/dev-guide/feature-zero-to-hero.md#openshift-feature-development-zero-to-hero-guide. If you are looking for a review on an OpenShift Enhancement Proposal that involves changes to components maintained by the control plane group, please request a review in the [`#forum-ocp-workloads`](https://redhat.enterprise.slack.com/archives/CKJR6200N) Slack channel.

This document contains the following sections:

- [Code conventions](#code-conventions) - A collection of guidelines, style suggestions, and tips for writing code.
- [Testing guidelines](#testing-guidelines) - Guidelines and expectations for testing of contributions.
- [Pull Request process/guidelines](#pull-request-process-and-guidelines) - Guidelines and expectations of pull requests containing contributions.
- [Review expectations](#review-expectations) - Guidelines and expectations for requesting reviews and interacting with reviewers.

## Code Conventions

We largely follow the [Kubernetes Code Conventions](https://github.com/kubernetes/community/blob/main/contributors/guide/coding-conventions.md#code-conventions).

Review both the Kubernetes Code Conventions and the ones specified here. There will be some overlap. If any conventions are at odds with one another, prefer the conventions explicitly documented here.

### Bash

- Follow the [shell styleguide](https://google.github.io/styleguide/shellguide.html).
- Use [`shellcheck`](https://github.com/koalaman/shellcheck) to identify common mistakes or caveats.
- Ensure that all scripts run consistently across Linux and MacOS.

### Golang (Go)

- Review [Effective Go](https://go.dev/doc/effective_go).
- Review common [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments).
- Review and avoid [Go Landmines](https://gist.github.com/lavalamp/4bd23295a9f32706a48f)
- Comment your code following the [Go comment conventions](https://go.dev/doc/comment).
    - Comments should be meaningful and add context and/or explain choices that cannot be expressed through clear code.
    - All exported types, functions, and methods must have descriptive comments.
    - All unexported types, functions, and methods should have descriptive comments.
- When adding command-line flags, use dashes/hyphens (`-`) and not underscores (`_`).
- Naming
    - Please consider package name when selecting an interface name, and avoid redundancy. For example, `storage.Interface` is better than `storage.StorageInterface`.
    - Do not use uppercase characters, underscores, or dashes in package names.
    - Please consider parent directory name when choosing a package name. For example, `pkg/controllers/autoscaler/foo.go` should say `package autoscaler` not `package autoscalercontroller`.
        - Unless there's a good reason, the package foo line should match the name of the directory in which the .go file exists.
        - Importers can use a different name if they need to disambiguate.
    - Locks should be called `lock` and should never be embedded (always `lock sync.Mutex`). When multiple locks are present, give each lock a distinct name following Go conventions: `stateLock`, `mapLock` etc.
- Context propagation
    - Never discard a context or substitute `context.Background()`/`context.TODO()` when a context is already available
    - Pass contexts through call chains to enable cancellation, deadlines, and request-scoped values
- Error handling
    - Wrap errors with meaningful context before returning or logging them.
- When logging, follow the [Kubernetes Logging Conventions](https://github.com/kubernetes/community/blob/main/contributors/devel/sig-instrumentation/logging.md).
- Dependencies
    - All dependencies must be vendored using `go mod vendor`
    - After adding/updating dependencies, always run `go mod tidy && go mod vendor`
    - Prefer using existing libraries from library-go, kubernetes, and other established dependencies
    - Minimize introduction of new dependencies
- When patching OpenShift-maintained forks of "upstream" repositories, patches should be as small as reasonably possible and should minimize touch points with code that is likely to change and impact the rebasing process.

### General

Regardless of the programming language, make sure to take the following into consideration:
- Keep readability / maintainability in mind when writing code.
    - Clever code and abstractions are often harder to reason about after the fact. Keep clever code and abstractions to the minimum necessary to accomplish the end-goal.
- Testing philosophy
    - Test your code's functional behavior, not standard library functionality
    - Do not write tests that merely verify Go's built-in features work correctly

### Directory and File Conventions

- Avoid package sprawl. Find an appropriate subdirectory for new packages.
    - Libraries with no appropriate home belong in new package subdirectories of `pkg/util`.
- Avoid general utility packages. Packages called "util" are suspect. Instead, derive a name that describes your desired function. For example, the utility functions dealing with waiting for operations are in the `wait` package and include functionality like `Poll`. The full name is `wait.Poll`.
- All filenames should be lowercase.
- Go source files and directories use underscores, not dashes.
    - Package directories should generally avoid using separators as much as possible. When package names are multiple words, they usually should be in nested subdirectories.

### Controller Patterns

All controllers must follow the **library-go factory pattern**.

For the complete controller pattern with code examples and implementation details, see [AGENTS.md - Controller Pattern](./AGENTS.md#controller-pattern-library-go-factory).

**Key requirements for contributors:**
- Use informers/listers for reading resources in sync loops
  - Never make direct API calls for Get/List operations on watched resources
  - Write operations (Create/Update/Delete/Patch) require direct API calls
- Handle errors in sync() properly
  - Return transient errors (API failures, network issues) to trigger rate-limited retry
  - Wrap errors with context using `fmt.Errorf("context: %w", err)` for debugging
  - For validation errors, handle appropriately (e.g., scale down deployment, update status) before returning
- Use `resourceapply` helpers from library-go when available (e.g., ApplyDeployment, ApplyServiceAccount)
  - For CRD resources, use `ApplyKnownUnstructured` if supported (ServiceMonitor, PrometheusRule, etc.)
  - For operations without helpers (e.g., UpdateScale, UpdateStatus), use direct API calls or v1helpers with proper error handling
- Register new controllers in `pkg/operator/starter.go`

## Testing Guidelines

These are high-level testing guidelines. Individual component repositories may have additional testing guidelines to follow when making contributions.

### Unit Tests

- **Required**: All changes must include unit test additions/changes (exceptions at reviewer/approver discretion)
- Table-driven tests are preferred for testing multiple scenarios/inputs. For an example, see https://github.com/openshift/cluster-authentication-operator/blob/a493799952e9b6838021ccc7d15d3d37d7ad3508/pkg/controllers/externaloidc/externaloidc_controller_test.go#L108
- Tests must pass on all platforms (at the very least, Linux + MacOS)
- Do not expect asynchronous operations to happen immediately - use wait and retry patterns instead
- Focus on testing your code's functional behavior, not standard library functionality

Example:

```go
func TestMyController_Sync(t *testing.T) {
    tests := []struct {
        name    string
        setup   func(*fakeClient)
        wantErr bool
    }{
        {
            name: "successful sync",
            setup: func(c *fakeClient) {
                // Setup test state
            },
            wantErr: false,
        },
        // More test cases...
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Test implementation
        })
    }
}
```

### E2E Tests

- **Required for**: New descheduler profiles, mode changes, major features
- Significant features should come with integration and/or end-to-end tests where appropriate
    - End-to-end tests _may_ be scoped as a separate work item when they must be added to the openshift/origin repository (at reviewer/approver discretion)
- Use OpenShift Tests Extension (OTE) framework - see [Using OpenShift Tests Extension](#using-openshift-tests-extension-ote) section
- Place tests in `test/e2e/` directory
- Follow Ginkgo v2 conventions

**Topology compatibility requirements**:
- Add `[Skipped:MicroShift]` if not applicable to MicroShift
- Add `[Skipped:SingleReplicaTopology]` if requires multiple nodes
- Use `[apigroup:...]` labels to indicate API dependencies

**Manual testing**: For manual verification, use the [`Cluster Bot` Slack App](https://redhat.enterprise.slack.com/archives/D03KX7M1CRJ) to create test clusters. See [Verifying Your Changes](#verifying-your-changes--creating-an-openshift-cluster-from-a-pr) for details on using Cluster Bot and [Getting an OpenShift Cluster](#getting-an-openshift-cluster) for cluster setup. Follow https://github.com/openshift/enhancements/blob/master/dev-guide/operators.md for guidance on building component images and modifying cluster-operators.

## Pull Request Process and Guidelines

This section assumes that you have a functional understanding of `git` and how to create a pull request on GitHub.

If you do not, start with [GitHub's "Getting Started" guide](https://docs.github.com/en/get-started/start-your-journey).

### Prerequisites

Before you commit any changes or create any pull requests, you must adhere to OpenShift contribution policies. Currently, that means enabling commit signature verification.

See https://docs.google.com/document/d/1184EPSGunUkcSQYUK8T4a6iyawwi6f2zxdbB2jtG9nQ/edit?usp=sharing for details on enabling commit signature verification (requires Red Hat authentication).

### Pre-Submit Checklist

Before creating a pull request, run the following commands locally to ensure your changes meet quality standards:

```bash
# Verify code formatting and linting
make verify

# Run unit tests
make test-unit

# Build to ensure no compilation errors
make build
```

All checks must pass before requesting review. If any check fails, fix the issues before creating your PR.

### Creating a Pull Request

When creating a pull request, include the following:

- A brief, but descriptive, title.
    - All pull requests _should_ link to a Jira ticket associated with the work. There is automation that performs this linking when prefixing the title with the Jira ticket identifier like: `CNTRLPLANE-XXXX: my pull request title`. For pull requests that have no Jira ticket associated with it, you can prefix it with `NO-JIRA:` to signal that there is not a Jira ticket associated with it.
- A useful description of the changes being made and why they are important. Include links to supporting documents and any additional context that reviewers may need.

### CI / CD

For CI/CD, OpenShift uses Prow to run various checks. This can include unit tests, e2e tests, linters, etc.

The jobs configured for each repository are in https://github.com/openshift/release/tree/main/ci-operator/config/openshift.

There are often a mixture of required and optional checks as well as merge criteria that must be met before a pull request can merge. When any of these checks fail, the GitHub Prow bot will leave a comment on the PR with links to the run of that check that failed.

As the PR author, it is your responsibility to evaluate the failed checks and determine if there are any changes necessary to pass the checks. If you suspect that the check failure was a flake, you can trigger retests by commenting `/retest` (or `/retest-required` for retesting only the required checks) on the PR.

### Verifying Your Changes / Creating an OpenShift Cluster from a PR

As part of merging a PR, there is a requirement to verify that the changes you've made are working as expected using the `/verified` comment command.

While there are a lot of scenarios where the existing CI/CD checks may be sufficient to verify your changes are working (and can be denoted by commenting `/verified by ci`), there may be scenarios where manual verification is required.

You can use the `Cluster Bot` Slack App to create a cluster from a PR by sending it a message in the format of `launch ${OCP_VERSION},${PR_LINK} ${PLATFORM},${VARIANT}`. As an example, `launch 4.23,https://github.com/openshift/cluster-kube-descheduler-operator/pull/123 aws,techpreview` would launch an OpenShift 4.23 cluster with the changes made in openshift/cluster-kube-descheduler-operator#123 running on AWS with the TechPreviewNoUpgrade feature-set enabled. For more information on what `Cluster Bot` can do, you can send it a message saying `help` and it will respond with additional documentation on how it can be used.

Once you've verified your changes work as expected, you can mark the PR as verified by commenting `/verified by @{your_github_handle}` on the PR.

## Review Expectations

### Requesting a Review

If you are not a member of the OpenShift control plane team and you need a review on a PR, post it in the [#forum-ocp-workloads](https://redhat.enterprise.slack.com/archives/CKJR6200N) Slack channel or reach out to folks outlined in the OWNERS file directly.

If you are a member of the OpenShift control plane team, reviews should come from your feature team. In the event your feature team does not have someone that can approve a PR, post it in the [#control-plane](https://redhat.enterprise.slack.com/archives/CC3CZCQHM) Slack channel.

OpenShift uses AI code review tools as part of the code review process. Before requesting a review, address all feedback from the code review agent(s). It is up to your discretion as the contributor how you would like to address that feedback. Responding with an explanation as to why you are not going to take action on a comment made by the agent is an acceptable way to "address" its feedback.

### Interacting with Reviewers

When interacting with reviewers/approvers:

- Be professional.
- Be respectful of differing opinions, viewpoints, and experiences.
- Gracefully give and receive constructive feedback.
- Focus on what is best for the product/organization, not just us as individuals.

A special note on the usage of AI - to respect the time of those that are reviewing your contribution, please do not use AI to respond to review comments.

**Review timeline**: Most PRs are reviewed within 2-3 business days. PRs go through automated checks (unit tests, linters, verifications), code review (at least one maintainer approval required), and E2E tests before merging.

---
---

# Specific Guidelines for `cluster-kube-descheduler-operator`

## Pre-Submit Checks

Before pushing changes, execute:

```bash
make verify
```

This command runs verification checks (gofmt, govet, golang version validation). All must pass for CI success.

Before creating a pull request, also run:

```bash
make test-unit
make build
```

## Dependency Management

All Go dependencies are vendored in this repository. After modifying dependencies (adding, removing, or updating):

```bash
go mod tidy && go mod vendor
```

CI validates this using `make verify`.

## CRD Changes

If you modify CRD types in `pkg/apis/descheduler/v1/types_descheduler.go`, regenerate CRD manifests:

```bash
make regen-crd
```

This updates:
- `manifests/kube-descheduler-operator.crd.yaml`
- `deploy/0000_00_kube-descheduler-operator.crd.yaml`

After regeneration, run `make verify` to ensure consistency.

**Note**: `make regen-crd` handles most common cases, but is not foolproof. For complex API changes, the underlying generation script may need updates. Always verify the generated CRD manifests match your intended changes.

## API Changes

If you add new API versions or modify clientset/informers:

```bash
# Regenerate clientset, listers, and informers
make generate-clients
```

## E2E Tests

For comprehensive guidance on writing, running, and debugging E2E tests, see [TESTING.md](./TESTING.md).

Key test locations:
- Unit tests: `pkg/*/` directories (`*_test.go` files)
- E2E tests: `test/e2e/` (using OpenShift Tests Extension framework)

To run E2E tests locally (requires OpenShift cluster access):

```bash
export KUBECONFIG=/path/to/kubeconfig
export RELEASE_IMAGE_LATEST=<registry>/ocp/release:latest
export NAMESPACE=<ci-namespace>
make test-e2e
```

**Environment variables:**
- `KUBECONFIG` - Path to kubeconfig file (required)
- `RELEASE_IMAGE_LATEST` - CI release image URL (required)
- `NAMESPACE` - CI namespace for image resolution (required)
- `ARTIFACT_DIR` - Directory for test artifacts (optional, default: `/tmp/artifacts`)
