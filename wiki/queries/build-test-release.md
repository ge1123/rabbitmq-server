# Build, Test, Release, and Packaging Routing

## Purpose

Route investigations about the Make build, dependency pins, source-run nodes,
Common Test, packaging, Selenium, CI, compatibility, and release dispatch
without copying target inventories, workflow matrices, or release-note content
that will drift.

This area is mostly Make, shell, YAML, JSON, Markdown, package metadata, and
generated artifacts. Confirm the graph project first, use graph tools for
orientation, then treat targeted source and text reads as the behavioral
authority.

## Entry Points

 * `Makefile`: top-level release project, source distributions, source bundles,
   package delegation, install targets, generated workflow targets, source-run
   plugin defaults, and `PACKAGES_DIR`
 * `plugins.mk`: tier-1 plugins included by the Make-based release path, with
   the explicit warning that the server-release Concourse pipeline overrides
   this list for actual release builds
 * `rabbitmq-components.mk`: project version derivation, third-party dependency
   pins, built-in and community component lists, forced component rebuilds,
   shared `DEPS_DIR`, and shared `ERLANG_MK_TMP`
 * `erlang.mk`: inherited target machinery for dependency build, `test-build`,
   Common Test, `ct-*` suite targets, `t=`/`c=` selection, Dialyzer, Xref,
   cover, relx, escript, and Hex publishing support
 * `deps/rabbit_common/mk/rabbitmq-early-plugin.mk`: RabbitMQ-specific Common
   Test defaults, hidden CT nodes, CI skip-as-error behavior, local styled
   output, and `SECONDARY_DIST_VSN` download/setup
 * `deps/rabbit_common/mk/rabbitmq-run.mk`: source-run node environment,
   `TEST_TMPDIR`, generated test config, enabled-plugin-file preservation,
   `run-broker`, `start-cluster`, `stop-cluster`, and rolling
   `restart-cluster`
 * `deps/rabbit_common/mk/rabbitmq-dist.mk`: `dist` and `test-dist` plugin
   archive construction, CLI script/escript installation, and the difference
   between runtime and test dependency distributions
 * `deps/rabbit/Makefile`: broker-specific Bats integration, fast/slow CT
   groupings, parallel CT shard sets, sequential CT suites, and
   `parallel-ct-sanity-check`
 * `deps/rabbitmq_mqtt/Makefile`: plugin-specific parallel CT sharding pattern
   mirroring the broker shape
 * Selected plugin `Makefile`s: component-local dependencies and test fixtures,
   such as `deps/amqp10_client/Makefile` fetching ActiveMQ before `ct` and
   `ct-system`
 * `packaging/Makefile`: repository-local packaging dispatcher that validates a
   single source tarball before delegating to generic Unix or Docker image
   subdirectories
 * `packaging/generic-unix/Makefile`: generic Unix package construction from a
   source archive, including manpages, install targets, defaults rewriting,
   manifest generation, and tarball output
 * `packaging/docker-image/Makefile` and `packaging/docker-image/Dockerfile`:
   Docker image construction from the generic Unix archive
 * `CONTRIBUTING.md`: contributor-facing build/test examples, single-suite CT
   usage, `t=` group/case syntax, mixed-version local setup, and source-run
   node examples
 * `docs/compatibility.json` and `docs/COMPATIBILITY.md`: release-to-Erlang and
   release-to-Elixir compatibility evidence and maintenance procedure
 * `.github/workflows/test-make*.yaml`: Make build, xref, CT matrix, mixed
   cluster, and Dialyzer routing
 * `.github/workflows/test-management-ui*.yaml` and
   `.github/workflows/test-authnz.yaml`: Selenium CI that first builds a
   generic Unix package and Docker image, then runs suite-list files
 * `.github/workflows/test-upgrades.yaml` and
   `.github/workflows/test-upgrades.sh`: old/new distribution build and upgrade
   scenario verification
 * `.github/workflows/release-*-alphas.yaml`: branch-specific alpha release
   dispatch to `rabbitmq/server-packages`
 * `selenium/README.md`, `selenium/run-suites.sh`,
   `selenium/bin/suite_template`, and `selenium/test/utils.js`: browser
   end-to-end architecture, suite list execution, generated config/log/screen
   layout, local-vs-Docker modes, and WebDriver setup
 * `release-notes/`: flat per-version Markdown and historical notes, searched
   by version, component, issue number, or feature name

## Graph Query Strategy

Always begin with:

```text
list_projects
index_status(project="Users-wuchuni-Desktop-system_design-rabbitmq-server")
```

Useful graph orientation queries:

```text
get_architecture(
  project="Users-wuchuni-Desktop-system_design-rabbitmq-server",
  aspects=["entry_points", "clusters"])

search_graph(
  project="Users-wuchuni-Desktop-system_design-rabbitmq-server",
  query="make common test release package",
  limit=20)

search_graph(
  project="Users-wuchuni-Desktop-system_design-rabbitmq-server",
  query="package generic unix docker image source dist",
  file_pattern="Makefile|packaging/.*Makefile",
  limit=30)

search_graph(
  project="Users-wuchuni-Desktop-system_design-rabbitmq-server",
  query="common test parallel ct sanity check",
  file_pattern="deps/.*/Makefile|erlang.mk|deps/rabbit_common/mk/.*",
  limit=50)
```

Use graph-augmented literal search for Make/YAML/config evidence:

```text
search_code(
  project="Users-wuchuni-Desktop-system_design-rabbitmq-server",
  pattern="ENABLED_PLUGINS|RABBITMQ_ENABLED_PLUGINS",
  path_filter="^(Makefile|deps/|packaging/)",
  regex=true)

search_code(
  project="Users-wuchuni-Desktop-system_design-rabbitmq-server",
  pattern="parallel-ct|ct-\\$1|CT_OPTS|CT_SUITES|RABBITMQ_CT_SKIP_AS_ERROR|SECONDARY_DIST",
  path_filter="^(erlang\\.mk|deps/rabbit_common/mk/|deps/[^/]+/Makefile|\\.github/workflows/)",
  regex=true)

search_code(
  project="Users-wuchuni-Desktop-system_design-rabbitmq-server",
  pattern="RABBITMQ_PACKAGING_TARGETS|PACKAGES_SOURCE_DIST_FILE|SOURCE_DIST_FILE|package-generic-unix|docker-image|RABBITMQ_PACKAGING_REPO|generic-unix",
  path_filter="^(Makefile|packaging/|\\.github/workflows/)",
  regex=true)
```

Graph results are good at finding Makefile targets represented as nodes, such
as top-level `package-generic-unix`, `source-dist`, packaging `dist`, and
component `parallel-ct-sanity-check`. They are not enough to answer behavior
questions in this area because recipes, variables, workflow inputs, shell
branches, and comments carry much of the contract.

## Behavioral Notes

Top-level build and release routing:

 * The root `Makefile` is the release project `rabbitmq_server_release`; it
   includes `plugins.mk`, `rabbitmq-components.mk`, `erlang.mk`, and
   `mk/github-actions.mk`
 * `rabbitmq-components.mk` derives `PROJECT_VERSION` from `RABBITMQ_VERSION`,
   `git-revisions.txt`, `git describe`, or `0.0.0`, then pins shared
   third-party dependency versions for all components
 * `RABBITMQ_BUILTIN`, `RABBITMQ_COMMUNITY`, and `RABBITMQ_COMPONENTS` are
   component lists, while `FORCE_REBUILD = $(RABBITMQ_COMPONENTS)` makes
   RabbitMQ components rebuild rather than relying on erlang.mk's dependency
   cache
 * Source archives are produced by `source-dist` from `SOURCE_DIST_FILES` under
   `PACKAGES_DIR`; `source-bundle` is a separate source bundle path
 * Generated workflow targets live near the bottom of the top-level `Makefile`
   and use `ytt` templates, so generated workflow diffs should be routed back
   through `mk/github-actions.mk` and `.github/workflows/templates/`

Source-run routing:

 * Top-level `ENABLED_PLUGINS` defaults to `rabbitmq_management` for
   `run-broker`, `start-cluster`, and related source-run targets, then becomes
   comma-separated `RABBITMQ_ENABLED_PLUGINS`
 * `rabbitmq-run.mk` only exports `RABBITMQ_ENABLED_PLUGINS` when the node's
   `enabled_plugins` file does not already exist, so changing
   `ENABLED_PLUGINS` affects fresh source-run nodes only
 * Use `virgin-test-tmpdir`, remove the relevant
   `/tmp/rabbitmq-test-instances/rabbit@*/enabled_plugins` file, or use
   `rabbitmq-plugins` when investigating plugin changes on an existing
   source-run node
 * `run-broker` starts one foreground broker with generated test config,
   `start-background-broker` waits for startup, `start-cluster` starts numbered
   nodes in parallel, and `restart-cluster` drains and restarts nodes in reverse
   order
 * Source-run node state defaults under `TEST_TMPDIR`, with per-node log,
   Mnesia, quorum, stream, plugin expansion, feature-flag, and enabled-plugin
   paths derived in `rabbitmq-run.mk`

Common Test routing:

 * `erlang.mk` discovers `*_SUITE.erl` files into `CT_SUITES`, makes `tests::`
   depend on `ct`, and generates per-suite `ct-<suite>` targets
 * `ct-<suite> t=<group>` becomes a Common Test `-group`, and
   `t=<group>:<case>` becomes `-group <group> -case <case>`; `c=<case>` maps
   directly to `-case`
 * `rabbitmq-early-plugin.mk` sets CT logs to the repository `logs/` directory
   for components, adds hidden CT nodes and low net tick time, and sets
   `RABBITMQ_CT_SKIP_AS_ERROR=true` on GitHub Actions
 * Local CT output uses `cth_styledout` unless running on GitHub Actions; do not
   assume local and CI output hooks match
 * Mixed-version component tests can use `SECONDARY_DIST` directly or
   `SECONDARY_DIST_VSN`, which downloads a generic Unix archive and exports the
   extracted path
 * Broker and MQTT parallel CT targets render `ct.set-<N>.spec`, start shard
   nodes with fixed port-base offsets, and run `ct_master_fork`
 * `parallel-ct-sanity-check` fails when a suite is not assigned to either a
   parallel shard set or a sequential suite list, so route new-suite CI coverage
   questions to component `Makefile` sharding variables
 * Component `Makefile`s can add fixture setup to generic CT targets; for
   example `amqp10_client` fetches ActiveMQ before `tests`, `ct`, and
   `ct-system`

Packaging and image routing:

 * External OS package targets such as `package-deb`, `package-rpm`, and
   `package-windows` require `RABBITMQ_PACKAGING_REPO` and delegate to the
   external `rabbitmq-packaging` repository with `SOURCE_DIST_FILE`
 * Repository-local `package-generic-unix` and `docker-image` depend on the
   source distribution and delegate under `packaging/`
 * `packaging/Makefile` validates that exactly one existing `.tar.xz` or `.txz`
   source archive is selected before running non-clean targets
 * `packaging/generic-unix/Makefile` unpacks the source archive, runs install
   and manpage targets with package layout variables, rewrites
   `rabbitmq-defaults`, creates `etc/rabbitmq`, writes a manifest, and emits
   `rabbitmq-server-generic-unix-<version>.tar.xz`
 * `packaging/docker-image/Makefile` expects a generic Unix archive and copies
   it to `package-generic-unix.tar.xz` for the Docker build
 * `packaging/docker-image/Dockerfile` is an image path, not the OS package
   release path; it unpacks the generic Unix archive under `RABBITMQ_HOME`

CI and release routing:

 * `.github/workflows/test-make.yaml` runs build/xref on OTP 27 and 28,
   performs broker and MQTT parallel CT sanity checks, then delegates tests and
   mixed-cluster tests to reusable workflows
 * `.github/workflows/test-make-tests.yaml` maps make targets and plugins to
   `.github/workflows/test-make-target.yaml`; use it to understand CI sharding
   but avoid copying its matrix into wiki pages
 * `.github/workflows/test-make-target.yaml` checks out an optional ref, sets up
   OTP/Elixir, handles mixed-cluster `SECONDARY_DIST`, applies special
   component setup such as ActiveMQ and LDAP, runs `make -C deps/<plugin>`, and
   uploads CT logs
 * `.github/workflows/test-make-type-check.yaml` reuses the make-target
   workflow with `make_target: dialyze`
 * Management UI and authnz Selenium workflows build `package-generic-unix` and
   `docker-image` before building the mocha test image and running suite-list
   files
 * `test-upgrades.yaml` builds an old distribution from `v4.2.x`, builds a new
   distribution from the current checkout, and runs `test-upgrades.sh` scenarios
   against downloaded artifacts
 * `release-*-alphas.yaml` workflows do not build packages in this repository;
   they compute a prerelease identifier and dispatch branch-specific alpha
   release requests to `rabbitmq/server-packages`

Selenium routing:

 * `selenium/run-suites.sh` defaults to `full-suite-management-ui`, sorts suite
   names from the suite-list file, runs each with `ENV_MODES=docker`, records
   success/failure arrays, and exits with the first non-zero suite result
 * `selenium/bin/suite_template` owns generated config, `CONF_DIR_PREFIX`,
   logs, screenshots, Docker network setup, component orchestration, and command
   modes such as `start-rabbitmq`, `start-others`, and `test`
 * `selenium/test/utils.js` chooses local versus remote WebDriver from
   `RUN_LOCAL`, `SELENIUM_URL`, `RABBITMQ_URL`, and related environment
   variables; non-local runs add headless Chrome arguments and screenshots go
   under `SCREENSHOTS_DIR`

Compatibility routing:

 * Use `docs/compatibility.json` for release-specific Erlang and Elixir support
   ranges
 * Use `docs/COMPATIBILITY.md` for how the matrix was built and updated; it
   distinguishes compatibility evidence from `.github/versions.json`, which is
   for workflow population
 * CI workflow OTP/Elixir versions are test coverage choices, not necessarily
   the release compatibility matrix

## Cross-Component Links

 * Build and package entry points affect all components through
   `rabbitmq-components.mk`, `erlang.mk`, and shared `deps/rabbit_common/mk`
   includes
 * Broker source-run behavior in `rabbitmq-run.mk` depends on scripts copied
   from `deps/rabbit/scripts` and escripts built from `deps/rabbitmq_cli`
 * Component tests often cross into `deps/rabbitmq_ct_helpers` and
   `deps/rabbitmq_ct_client_helpers`; use graph tools for helper behavior after
   routing through the build/test page
 * Management UI Selenium failures may involve `deps/rabbitmq_management`,
   `deps/rabbitmq_web_dispatch`, Docker packaging, generated Selenium config,
   browser screenshots, and workflow artifacts
 * Upgrade tests bridge package distribution targets, source-run cluster
   targets, CLI commands, and broker feature flags
 * Release-note questions route to `release-notes/` for historical text and to
   component pages only when the note implies behavior that needs source
   verification

## Investigation Pitfalls

 * Do not use broad `release` graph searches as proof; they mix runtime release
   code, Make targets, Hex support, workflow variables, release notes, and
   unrelated functions
 * Do not infer complete CI behavior from graph nodes; workflow matrices,
   `uses:` indirection, path filters, artifact names, and shell steps require
   YAML reads
 * Do not copy complete target lists, CT suite inventories, workflow matrices,
   dependency lists, or release-note summaries into wiki pages
 * Do not treat `plugins.mk` as the final release-inclusion authority without
   checking the server-release pipeline override noted in that file
 * Do not expect `ENABLED_PLUGINS` to change an existing source-run node unless
   the node's `enabled_plugins` file is removed or the test tmpdir is reset
 * Do not assume OS packages are built locally in this repo; only generic Unix
   and Docker image package paths are repository-local
 * Do not treat local Selenium interactive mode and CI Docker mode as equivalent;
   `ENV_MODES`, `RUN_LOCAL`, `SELENIUM_URL`, generated config, and artifact
   paths change the failure surface
 * Do not combine `docs/compatibility.json` with `.github/versions.json`; they
   answer different questions

## Related Wiki Pages

 * `wiki/index.md`
 * `wiki/components/index.md`
 * `wiki/queries/index.md`
 * `wiki/source.md`
