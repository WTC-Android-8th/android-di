# Learning DI by Building It: Development and Code Review Guidelines

This repository is an Android learning mission with five stages, covering building a DI container and replacing it with a library. First identify the stage of the current task or PR. Apply the current stage's requirements and earlier functional guarantees that still apply. Do not carry forward implementation restrictions that a later stage explicitly replaces, or require later-stage designs in advance. Stage 5 is optional; apply its requirements only when that stage is in progress.

## Development Principles

- Before implementing features, record the feature list for the current stage in `README.md` and organize changes by feature. Document design decisions when the stage requires a rationale.
- When the mission allows alternative designs, do not assume there is only one correct solution. Examine the reasoning and actual behavior.
- Avoid adding unnecessary complexity to educational code merely to match conventions used by production DI libraries.

## Identifying the PR Stage

- For PR reviews, use the `미션 단계` (mission stage) section in the PR description as the source of truth. The author should specify a stage from 1 to 5 in the PR template.
- If the section is empty, check whether the changes and repository state clearly identify the stage. If the stage remains unclear, do not guess which stage requirements apply. Review only the common guidelines and state that the stage is unclear.
- Apply the identified stage's mandatory requirements and earlier requirements that remain applicable. For example, Stage 1 prohibits annotations, but Stage 2 requires them; Stage 5 preserves behavior without requiring the custom container's implementation.
- The Stage 1 mission page also includes a preparatory Stage 0.5. If a PR explicitly covers manual DI at Stage 0.5, review manual factory wiring and the provided activity test; do not require an automatic DI container yet.

## Learning Review Trigger

When a PR comment invokes the standard review with `@codex review`:

- Review the entire PR diff at the current head commit, including relevant surrounding code. Do not limit the review to the latest commit's changes.
- Determine the mission stage using the PR stage rules above and apply the relevant guidelines in `Code Review Rules` below.
- Respond in Korean. State the mission stage and reviewed head commit so the review scope is clear.
- Report correctness issues with concrete evidence and code locations. If none are found, say so without claiming that the code is free of defects.
- Give relevant design feedback on architecture, responsibility separation, Android DI and lifecycle behavior, and naming. Explain the reasoning and alternatives without presenting preferences as defects.
- Include 1–3 meaningful learning questions grounded in the PR, even if the code works correctly. Suggest a small test or experiment where useful.
- Return review feedback only. Do not modify code, create commits, or push changes.

The `@codex review` phrase is the GitHub review trigger. The repository rules above define the review style, so the author can use only that trigger. Add a short focus after it only when a one-off review needs a narrower scope.

## Code Review Rules

### Presenting Review Feedback

- **Correctness issues**: Identify the requirement that actually fails, a reproduction path or failing test, and the relevant code. Do not label speculation as a defect.
- **Design feedback**: Explain concerns about responsibilities or dependency direction using concrete code flows, and explain why they matter at the current stage.
- **Learning questions**: These can apply to working code. Ask about the reasoning behind a design and its alternatives, and suggest a small test or experiment to explore the question. Do not present learning questions as defects or mandatory fixes.
- Choose the 1–3 most meaningful learning questions for the PR. Briefly explain how answering them helps with design decisions or understanding Android behavior.
- Generate questions from concrete design decisions, code paths, and tests in the PR using the stage's learning focus below. These topics are not a fixed question bank. Do not repeat questions already answered convincingly in the PR or `README.md`; build on that reasoning where useful.
- Connect each question to a code location or design choice, explain a plausible alternative and its tradeoff, and suggest a focused experiment when it would help. Do not invent defects or require speculative robustness features to produce learning feedback.
- Distinguish mandatory requirements from optional ones. Avoid filling the review with style preferences or requirements from stages that have not started.

### Architecture and Separation of Responsibilities

- Check whether the UI handles rendering and user input, ViewModels handle screen state and workflows, and repositories handle data access. Judge actual responsibilities and dependency direction rather than layer names or file locations.
- Check for unnecessary dependencies from domain models to implementation details such as Room entities, Compose, or Android `Context`. When mapping is needed, examine the boundary where it occurs.
- Examine whether abstractions actually help replace implementations or support testing. Using an interface alone does not establish dependency inversion.
- After extracting the `:di` module, examine whether the core needs direct knowledge of the shopping app's domain types or Android screen lifecycles. Check the reasoning behind where app registration code and Android integration code belong.
- Do not evaluate Clean Architecture by a fixed number of layers. Base feedback on observable effects, such as code with different reasons to change being modified together, or difficulty testing and replacing implementations.

### Naming and Abstraction Boundaries

- Check whether names in public interfaces and names used by callers describe their roles and guaranteed behavior. If names expose internal details that callers do not need, such as storage mechanisms, creation order, or reflection use, examine which callers would be affected by changes to those details.
- Implementation names may include details needed to distinguish implementations. Do not require hiding useful selection criteria such as `Room` or `InMemory` in every case.
- Check for mismatches between names and behavior. Pay particular attention to a list position called an `id`, an object created on every call named as though it were shared, or an object scoped to a screen named as though it were application-wide.
- Support naming feedback with usage examples that reveal the misunderstanding and explain the impact of a rename. Do not report word preferences as defects.

### Required Checks and Learning Topics by Stage

#### Stage 1: Automatic DI

Source: [Stage 1 mission](https://techcourse-lms-plus-web.woowahan.com/missions/43?step=321).

Learning objective: Move dependency construction out of consumers and replace repeated manual wiring with reusable creation logic.

- **Required**: `ProductsViewModel` and `CartViewModel` use the same automatic injection logic. Adding another ViewModel must not require a factory or `when` branch for that specific ViewModel type.
- **Required**: Both screens share the intended `CartRepository` instance. Entering the cart screen must not create a fresh repository and lose in-memory cart contents. The provided tests must pass, and annotations must not be used at this stage.
- **Learning focus — responsibility**: Trace who constructs, resolves, and consumes dependencies. Examine whether replacing a repository affects the composition/wiring boundary or forces changes inside its consumer. A factory name alone does not establish this separation.
- **Learning focus — Android ownership**: Examine where the container is created, how Compose recomposition affects it, and how `viewModel(factory = ...)` delegates ViewModel ownership. Keep the discussion proportional to this stage; do not require Stage 4 scope infrastructure.
- **Learning focus — reflection and tests**: Examine how constructor metadata leads to dependency resolution and construction, which assumptions the implementation makes, and what the activity startup test actually proves. Use adding a new ViewModel or comparing repository identity across screens as focused experiments.
- **Optional**: TDD, additional Robolectric feature tests, ViewModel tests, and domain/repository unit tests are extensions. Do not treat the absence of every optional test category as a mission failure.

#### Stage 2: Annotations and Recursive Injection

Source: [Stage 2 mission](https://techcourse-lms-plus-web.woowahan.com/missions/43?step=325).

Learning objective: Make injection targets explicit and resolve a dependency graph while preserving correct data identity and coroutine behavior.

- **Required**: Implement field injection in ViewModels and inject only annotated fields. Verify runtime retention and whether the actual reflection path reads Kotlin properties or JVM fields, including use-site targets. Test both annotated and unannotated fields.
- **Required**: Resolve dependencies recursively through ViewModel → repository → DAO. Identify the boundary between automatic construction and externally supplied objects such as the Room database. Tests must cover field injection and recursive resolution.
- **Required**: Add the `CartProduct` domain model and entity-to-domain mapping. Show product names and added timestamps, call repository `suspend` operations from the ViewModel coroutine scope, and delete the intended item using its persistent `Long` ID rather than its list position. Keep provided tests passing with justified changes for the new model.
- **Learning focus — injection timing**: Examine the period between object construction and field initialization, how consumers become safe to use, and how that differs from constructor injection. ViewModel field injection is deliberate practice in this stage; do not reject it merely because constructor injection is generally preferable. Relate the tradeoff to OS-created Android components when relevant.
- **Learning focus — dependency graph**: Examine recursion, shared dependencies, error messages, and registration boundaries in the actual implementation. Circular dependencies and unsupported constructors are exploration topics unless the PR demonstrates a concrete failure; do not impose a production-grade container as a prerequisite.
- **Learning focus — data and UI boundaries**: Trace persistent identity through state and deletion, keep suspending work outside non-suspending state-update lambdas, and examine where `DateFormatter` or formatted text belongs. Assess naming by whether it distinguishes persistent IDs from positions and communicates responsibilities.
- **Optional**: Stable `LazyColumn` keys and avoiding direct UI references to `CartProductEntity` are optional mission items. They remain useful design discussion topics; distinguish such feedback from unmet mandatory requirements.

#### Stage 3: Qualifiers and Module Separation

Source: [Stage 3 mission](https://techcourse-lms-plus-web.woowahan.com/missions/43?step=326).

Learning objective: Separate dependency selection from consumers and separate a reusable DI engine from app-specific registration.

- **Required**: Explicitly select Room or in-memory implementations of the same interface. Ambiguous resolution without a qualifier must produce a clear error rather than choosing an arbitrary registration. Test qualifier selection and the missing-qualifier failure.
- **Required**: Extract the DI library into a separate module that does not depend on shopping app types. Inspect both imports and Gradle dependencies. Record the rationale for qualifier representation and an Android versus pure JVM module in `README.md`.
- **Learning focus — keys and resolution**: Examine whether registration and lookup use consistent type-and-qualifier identity, whether different bindings can overwrite each other, and when mistakes in string keys or annotation types become visible. Evaluate the chosen representation instead of prescribing one universal choice.
- **Learning focus — dependency direction**: Trace `:app` → `:di`, app-owned registration, and generic core resolution. Passing a type or provider into the core is different from hardcoding `CartRepository` in the core; evaluate the actual coupling.
- **Learning focus — Android integration and naming**: Examine which layer supplies `Context` and will later receive lifecycle signals. An Android DI module is allowed; a pure JVM core is not mandatory. Check whether consumer-facing names expose irrelevant storage details while retaining implementation names that genuinely help select a binding.
- **Optional**: A registration DSL and publishing the library are extensions. Do not require them to satisfy module separation or qualifier support.

#### Stage 4: Lifecycles and Scopes

Source: [Stage 4 mission](https://techcourse-lms-plus-web.woowahan.com/missions/43?step=330).

Learning objective: Model instance identity and ownership explicitly, and release references when the owning scope ends.

- **Required**: Keep `CartRepository` for the application lifetime, `ProductRepository` for its ViewModel lifetime, and `DateFormatter` for a navigation destination's lifetime. Reuse the formatter within that screen and create a new one after the destination is popped and entered again. An Activity scope is not a substitute for a destination scope in this single-Activity app.
- **Required**: Drop scope-held instance references on scope closure, prevent accumulation across repeated navigation, and test both creation and disposal for each scope. Review the design for adding scope kinds without multiplying internal `when` branches.
- **Required**: Include the link to the learner's “Why Reinvent the Wheel” write-up in the PR. In `README.md`, explain DI versus service locator using concrete code and distinguish that judgment from DIP and IoC. Explain what a migration to KSP would need to redesign, or justify retaining runtime resolution.
- **Learning focus — ownership and signals**: Trace the scope key, owner, creation, reuse, and cleanup paths. Distinguish recomposition, leaving composition, configuration changes, and final back-stack removal. Evaluate `DisposableEffect`, lifecycle callbacks, and ViewModel cleanup against the behavior actually promised.
- **Learning focus — boundaries and retention**: Examine how the Android integration layer signals a generic core, and whether longer-lived objects retain shorter-lived dependencies or an inappropriate `Context`. Use concrete reference paths rather than labeling every global container a memory leak.
- **Learning focus — verification and API design**: Examine what a cleanup test observes: removal from the container, a disposal callback, or owner teardown. Dropping container references does not prove immediate garbage collection. Assess whether public testing hooks expose internal state unnecessarily, and use repeated navigation or same/different owner identity as targeted experiments.
- **Optional**: Keeping `DateFormatter` across configuration changes and fully registering arbitrary scope kinds from outside the core are extensions. Do not report the lack of configuration-change retention as a mandatory failure; evaluate it if the PR claims to support it.

#### Stage 5: Replacing the Container with Hilt or Koin

Source: [Stage 5 mission](https://techcourse-lms-plus-web.woowahan.com/missions/43?step=332).

Learning objective: Evaluate a library through the guarantees it preserves and the tradeoffs it introduces, using experience from the custom container.

- **Required when undertaking Stage 5**: Replace app DI with Hilt or Koin while preserving repository injection, explicit implementation selection, all three Stage 4 lifetime guarantees, and existing tests. Keeping the old module and tests is allowed; do not require their deletion.
- **Required**: Document the library choice against the criteria from the learner's “Why Reinvent the Wheel” write-up, unsupported-scope workarounds, and reasons for any test changes in `README.md`. Verify the actual application startup, registration, ViewModel integration, and testing setup for the selected library.
- **Learning focus — scope equivalence**: Evaluate whether the library binding's actual owner matches the required lifetime. Do not equate a factory or application singleton with one instance per live destination. For Hilt, examine the destination-scope workaround; for Koin, examine scope identity and closure. Judge observed behavior rather than library-specific annotation or DSL names alone.
- **Learning focus — migration and tests**: Examine which app boundaries remain independent of the DI framework and which tests change because of framework setup versus coupling to the old implementation. The mission preserves behavior, not every custom injection mechanism.
- **Learning focus — failure timing and tradeoffs**: Examine when a missing binding is detected in the chosen configuration and compare that with the custom container. Discuss lifecycle integration, graph validation, testability, generated code, build cost, and runtime work using the PR's evidence. Do not assert performance improvements without measurements.
- **Optional**: Trying both libraries and measuring startup or build time are extensions. If undertaken, review measurement conditions and the comparison; do not require benchmarks or a second implementation for completion.
