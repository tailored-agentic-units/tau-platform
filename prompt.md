We're going to be conducting a final planning and preparation session that will provide use with everything we need to lay out the requirements, initial details, project management infrastructure, and repository for Phase 1 - Foundation of the TAU Project (use /project-management and /github-cli for details).

My entire goal for Phase 1 - Foundation is to get the minimum viable `tau-runtime` setup, along with the extensions that would be required to get the `tau-runtime` to be able to run on a local dev machine. All of the current details for `tau-runtime` can be found in the concepts/tau-runtime directory.

Beyond this, the current `tau-core` and `tau-orchestrate` libraries can be found at ~/tau/tau-core and ~/tau/tau-orchestrate.

## Additional Concept Details

Last night, I had some final thoughts about how to approach the `tau-runtime` development that I believe should be considered and merged into concepts/tau-runtime/README.md. They are as follows:

Standards for the tau-runtime

Inside the runtime, the only concern is managing receipt of the context, processing the workflow with additional inputs / outputs as needed, and returning the result.

The interface to the runtime will provide integration points for:
- Bootstrapping a session, with context being provided from external persistence sources
- sending and receiving additional context in various formats
- outputting metadata
- administrative / diagnostic / validation features

Any concerns beyond that are outside of the concern of the runtime (i.e. - data persistence, integration with IAM, containerization standards, etc.). Those are layers of complexity that should reside outside of the runtime. The enables the establishment of an extensible ecosystem of components that can be layered on top of the runtime (similar to the extensions that turn the Linux kernel into an embedded device OS, a Desktop OS, a server OS, or any other kind of operating system that could potentially be needed. This way, we’re not locked into accessing the runtime from a singular, restrictive platform.

We should investigate leveraging gRPC or ConnectRPC for the runtime interface.

Perform a preparation session tomorrow where the concept doc is used to initialize all of the library repositories, fill in / adjust all of the GitHub projects management infrastructure, and plan out the initial concept documents for each library. That way, we make sure that we’ve considered how the ecosystem will interconnect, work through any pitfalls we might have missed if we just started with the first adjustments, and make sure our holistic long-term vision is aligned and cohesively laid out.

Build each component external to the runtime as if it’s a sub-system component of an operating system: setup the local dev environment to compose a tau-runtime container with locally run extensions, and transition to cloud / self-hosted production services on deployment.

Setup the tau-runtime as a single repository with a sub-module (library) structure with tagged releases in the format: tau-[lib]@v[version]

The tau-runtime should have a single skill with references to each sub-component. The main skill.md will focus on describing the general runtime architecture and how to reference its sub-components.

## Instructions

Review the `tau-runtime` concept, the current library state, and the current GitHub project management infrastructure and establish a strategy that will:

1. Allow you to finalize the concepts/tau-runtime/README.md concept document
2. Determine any adjustments that need to be made to the ~/tau/tau-marketplace plugin skills
3. Determine the initial details for each of the `tau-runtime` libraries.
4. Establish a `tau-runtime` repository with each of the supporting libraries established as sub-modules within the `tau-runtime` monorepo (including the existing `tau-core` and `tau-orchestrate` libraries). The repository should include a PROJECT.md that provides the long-term vision for the `tau-runtime` along with a roadmap derived from the `tau-runtime` concept document.
5. Determine any adjustments that need to be made to the existing GitHub project management infrastructure. This includes labels, issues, milestones, repositories, and the TAU Platform project details.

This session will be very heavily focused on planning and ensuring we have all of the details clearly laid out and aligned so that our long term vision is organized with details that pave a path towards its successful execution. By the time we complete it, we should be setup with everything we need to dive into a focused, productive development workflow with minimal distractions or requirements to continually revisit and tweak these concerns.
