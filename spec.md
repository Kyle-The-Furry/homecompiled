# Technical Specification: homecompiled

## Section 1: Introduction

`homecompiled` is a Linux-only Python CLI tool that automates the download, verification, extraction, optimization, build, installation, and shell exposure of a curated set of performance-sensitive programs and libraries. Its primary purpose is to make “locally optimized from source” builds practical for end users without asking them to manually research per-package build quirks, compiler flags, dependency ordering, or shell integration. The tool does not attempt to be a general-purpose package manager. Instead, it is intentionally opinionated, deterministic, and hard-coded around a finite registry of supported packages.

The central technical model is a package registry backed by one Python module per supported package. Each package module contains the pinned version metadata, source archive URL, optional checksum metadata, declared vendored dependencies within the supported package set, supported binary aliases, library environment wrappers, and package-specific build logic. This is a deliberate design choice: the supported packages in this domain have too many package-specific build differences for a purely declarative or generic “build recipe” system to remain reliable. The hard-coded approach trades extensibility for correctness and maintainability.

The tool installs all artifacts beneath a dedicated `~/homecompiled/` directory, while generating a shell file at `~/.hc-aliases`. Sourcing `~/.hc-aliases` gives the user two kinds of integration points: standard command aliases that redirect common executable names to the locally built binaries, and composable shell functions prefixed with `hc-` that temporarily prepend the necessary environment variables so other commands can run or link against selected homecompiled libraries. This design keeps the installation isolated from system directories while still making the optimized builds convenient to use.

A major design requirement is “maximize optimization without breaking functionality.” The default toolchain preference is `clang` for C, `clang++` for C++, `gfortran` for Fortran, and `ld.lld` for linking where supported by the package’s build system. The default optimization baseline for C and C++ is `-O3 -march=native -mtune=native -flto=thin`, with a matching Fortran baseline unless a package must opt out of specific flags for correctness or build compatibility. Package-specific builders are responsible for injecting additional safe optimizations, disabling unsafe ones, or adapting the global settings into the idioms of Autotools, CMake, Meson, Cargo, and custom build systems.

The orchestration model is intentionally conservative. Builds are globally sequential, but each selected package first expands to include its explicitly declared vendored dependencies from the supported package registry. Those dependencies are built before the package that needs them, using a topologically sorted order. Build failures do not abort the full run; `homecompiled` continues with the remaining buildable packages and emits a final summary of successes, skips, warnings, and failures. This behavior is essential because long source builds should yield as much useful output as possible from a single invocation.

## Section 2: Tech Stack

The implementation language is Python 3.12+. The CLI framework is `typer`, chosen for strong ergonomics, standard type-driven option parsing, and straightforward future growth into additional commands such as benchmarking. Terminal presentation uses `rich` for readable progress, warnings, summaries, and structured error output. Testing uses `pytest`. Static and style tooling should use `ruff` and `mypy`.

The runtime implementation should prefer the Python standard library wherever practical:
- `pathlib` for filesystem paths
- `subprocess` for build command execution
- `hashlib` for checksum verification
- `shlex` for safe shell quoting and command formatting
- `json` for manifest persistence
- `tarfile` and `zipfile` where sufficient, with fallback to external tools only if required by unsupported archive formats
- `tempfile` for staging and atomic replacement workflows
- `dataclasses` and `typing` for package metadata and interfaces

`pydantic` is **not required in v1**. The package registry is hard-coded and not user-authored, so runtime schema validation through a separate dependency is unnecessary. Use frozen dataclasses plus clear type hints and unit tests instead. This keeps startup light and reduces dependency surface area.

The build execution layer must support the following build system families because the package set spans multiple ecosystems:
- Autotools / `configure` + `make`
- CMake
- Meson + Ninja
- Cargo
- Custom Makefiles / bespoke upstream scripts

The shell integration layer targets Bourne-style shells available on Linux, specifically Bash and Zsh. The generated `~/.hc-aliases` file must use syntax compatible with both. Fish shell support is out of scope for v1.

The development toolchain should be:
- Python 3.12+
- `typer`
- `rich`
- `pytest`
- `ruff`
- `mypy`

The project should be shipped as a standard Python package with a `pyproject.toml` entry point exposing the CLI executable name `homecompiled`.

## Section 3: Code Structure

```text
homecompiled/
├── pyproject.toml
├── README.md
├── src/
│   └── homecompiled/
│       ├── __init__.py
│       ├── __main__.py
│       ├── cli.py
│       ├── constants.py
│       ├── context.py
│       ├── models.py
│       ├── resolver.py
│       ├── manifests.py
│       ├── aliases.py
│       ├── shell.py
│       ├── downloads.py
│       ├── verify.py
│       ├── extract.py
│       ├── commands.py
│       ├── execution.py
│       ├── env.py
│       ├── toolchains.py
│       ├── buildsystems/
│       │   ├── __init__.py
│       │   ├── autotools.py
│       │   ├── cmake.py
│       │   ├── meson.py
│       │   ├── cargo.py
│       │   └── make.py
│       ├── packages/
│       │   ├── __init__.py
│       │   ├── registry.py
│       │   ├── base.py
│       │   ├── grep.py
│       │   ├── ripgrep.py
│       │   ├── tar.py
│       │   ├── bzip2.py
│       │   ├── xz.py
│       │   ├── zstd.py
│       │   ├── pigz.py
│       │   ├── zlib_ng.py
│       │   ├── ffmpeg.py
│       │   ├── libsvtav1.py
│       │   ├── dav1d.py
│       │   ├── libsvtvp9.py
│       │   ├── x265.py
│       │   ├── x264.py
│       │   ├── flac.py
│       │   ├── libopus.py
│       │   ├── libjpeg_turbo.py
│       │   ├── libvips.py
│       │   ├── libjxl.py
│       │   ├── mpv.py
│       │   ├── handbrake.py
│       │   ├── qbittorrent.py
│       │   ├── obs_studio.py
│       │   ├── darktable.py
│       │   ├── audacity.py
│       │   ├── openblas.py
│       │   └── fftw.py
│       └── bench/
│           ├── __init__.py
│           └── README.md
└── tests/
    ├── test_cli.py
    ├── test_resolver.py
    ├── test_aliases.py
    ├── test_verify.py
    ├── test_manifests.py
    ├── test_toolchains.py
    ├── test_buildsystems.py
    └── packages/
        └── test_registry.py
```

The package should be organized around three layers: orchestration, reusable build helpers, and package-specific recipes. The orchestration layer parses CLI options, resolves the selected package graph, manages per-run context, records results, and regenerates shell integration files. The reusable build helper layer contains shared behavior for downloads, archive extraction, environment construction, command execution, and adapters for common build systems. The package-specific recipe layer contains one Python module per package and is the only place where package-specific versions, URLs, flags, dependency declarations, binary exports, and custom build logic live.

The `packages/` directory is the authoritative registry. Each module in that directory exports one `PackageBuilder` instance or subclass. The registry module imports them and exposes a mapping from user-facing package IDs such as `zlib-ng` or `obs-studio` to builder objects. This split makes the hard-coded package set explicit and easy to audit, while keeping the resolver and executor generic.

The `bench/` directory exists in v1 only as a reserved namespace for the future benchmark feature. No benchmark orchestration is implemented in v1, but the package layout must make it easy to introduce benchmark definitions later without restructuring the project.

## Section 4: Project Specific Considerations

1. **Naming and command identity**

   * The project name, Python package name, and CLI executable name are all `homecompiled`.
   * Treat any occurrence of `homecompile` as a typo and do not encode it into public interfaces.
   * User-facing package IDs are lowercase kebab-case, for example `ripgrep`, `zlib-ng`, `obs-studio`, `libjpeg-turbo`.
   * Python module filenames use snake_case, for example `zlib_ng.py`, `obs_studio.py`, `libjpeg_turbo.py`.

2. **Supported package set**

   * The following packages must be present in the default hard-coded registry:

     * System Utils: `grep`, `ripgrep`, `tar`
     * Compression Utils: `bzip2`, `xz`, `zstd`, `pigz`, `zlib-ng`
     * Multimedia Utils: `ffmpeg`, `libsvtav1`, `dav1d`, `libsvtvp9`, `x265`, `x264`, `flac`, `libopus`, `libjpeg-turbo`, `libvips`, `libjxl`
     * GUI Programs: `mpv`, `handbrake`, `qbittorrent`, `obs-studio`, `darktable`, `audacity`
     * Misc: `openblas`, `fftw`

3. **Non-goals**

   * `homecompiled` is not a general-purpose package manager.
   * It does not support user-defined package recipes in v1.
   * It does not install distro-level prerequisites via `apt`, `dnf`, `pacman`, or similar tools.
   * It does not install into `/usr`, `/usr/local`, or any system-owned prefix.
   * It does not support non-Linux platforms in v1.
   * It does not implement benchmarking in v1.

4. **CLI surface**

   * The v1 CLI entry point is `homecompiled [OPTIONS]`.
   * Supported options:

     * `--only [package1 package2 ... packageN]`
     * `--cc <command-name>`
     * `--cxx <command-name>`
     * `--fc <command-name>`
     * `--ld <command-name>`
     * `--cflags <flags-string>`
     * `--fcflags <flags-string>`
     * `--ldflags <flags-string>`
   * `--only` behavior:

     * Accept zero or more package names after the option as a repeated list.
     * Normalize names exactly against the registry’s user-facing package IDs.
     * Invalid names do not stop execution.
     * Invalid names produce warnings and are reported in the final summary.
     * If `--only` is omitted, the tool targets all registry packages.
     * If `--only` contains no valid package names, the tool performs no builds, regenerates aliases from the currently installed manifests, prints warnings, and exits successfully.
   * There is no generic “add package by URL” feature.

5. **Filesystem layout**

   * Default root: `~/homecompiled/`
   * Required subdirectories:

     * `~/homecompiled/sources/`
     * `~/homecompiled/build/`
     * `~/homecompiled/install/`
     * `~/homecompiled/cache/`
     * `~/homecompiled/logs/`
   * Shell integration file:

     * `~/.hc-aliases`
   * The install prefix for each package is isolated:

     * `~/homecompiled/install/<package-id>/`
   * Each package install prefix must contain a manifest file written by `homecompiled`, for example:

     * `~/homecompiled/install/<package-id>/.homecompiled-manifest.json`

6. **Atomicity and preservation of working installs**

   * A failed rebuild must not destroy the last known-good install for a package.
   * Build and install into a staging directory first.
   * Replace the existing install prefix only after a successful full package build and manifest generation.
   * Use atomic rename where the filesystem permits it.
   * If replacement cannot be atomic, perform a safe fallback with rollback handling.

7. **Pinned source policy**

   * Every package has a pinned version hard-coded in its module.
   * Every package has a hard-coded, trusted source archive URL.
   * Packages may optionally define checksum metadata.
   * If checksum metadata is present, `homecompiled` must verify the downloaded archive before extraction.
   * If checksum metadata is absent, the download proceeds with an explicit informational note that verification was unavailable for that package.
   * Verified checksums should support at least SHA-256.
   * Signature verification is not required in v1.

8. **Archive handling**

   * The downloader stores archives under `~/homecompiled/cache/`.
   * Extraction occurs under a package/version-specific path beneath `~/homecompiled/build/`.
   * The extractor must support common release formats used by the supported package set, including `.tar.gz`, `.tar.xz`, `.tar.zst`, and `.zip` where necessary.
   * Extraction must reject path traversal entries and unsafe archive members.

9. **Build orchestration**

   * The global orchestration model is strictly sequential.
   * Before building, expand the selected package list to include each package’s explicitly declared vendored dependencies from the supported registry.
   * Resolve build order with a topological sort.
   * Cycles in the registry are treated as developer errors and must abort the run before any package build starts.
   * External distro dependencies are not part of this resolver. Package builders must check for them in preflight and fail with actionable messages if missing.

10. **Definition of vendored dependency in this project**

    * A vendored dependency is any other supported `homecompiled` package that a package recipe explicitly chooses to build against from the local `~/homecompiled/install/` tree instead of relying on the system version.
    * Vendored dependencies must be declared explicitly in the package module.
    * No implicit dependency inference is allowed.
    * Package-specific dependency declarations are authoritative even if upstream supports many more optional libraries.

11. **Per-package module contract**

    * Each package module must export one builder object implementing a shared interface.
    * Minimum required metadata per package:

      * `id`
      * `display_name`
      * `category`
      * `version`
      * `source_url`
      * `checksum` (optional)
      * `build_system`
      * `dependencies`
      * `binary_aliases`
      * `library_exports`
      * `safe_link_exports`
    * Minimum required behaviors per package:

      * `preflight(context)`
      * `fetch(context)`
      * `verify(context)`
      * `extract(context)`
      * `configure(context)`
      * `build(context)`
      * `install(context)`
      * `post_install(context)`
      * `create_manifest(context)`
    * Shared helper methods may implement most of these steps for standard build systems, but the package module remains the owner of package-specific arguments and quirks.

12. **Toolchain defaults**

    * Default C compiler: `clang`
    * Default C++ compiler: `clang++`
    * Default Fortran compiler: `gfortran`
    * Default linker command: `ld.lld`
    * Default C/C++ optimization flags:

      * `-O3 -march=native -mtune=native -flto=thin`
    * Default Fortran optimization flags:

      * `-O3 -march=native -mtune=native -flto=thin`
    * Default linker flags:

      * empty baseline unless the package adapter requires a minimal correctness flag
    * Package builders may add safe package-specific flags on top of the baseline.

13. **Override semantics**

    * `--cc`, `--cxx`, `--fc`, and `--ld` replace the corresponding default tool selections for the current invocation only.
    * `--cflags` replaces the baseline C and C++ optimization flag string for the current invocation only.
    * `--fcflags` replaces the baseline Fortran optimization flag string for the current invocation only.
    * `--ldflags` replaces the baseline linker flag string for the current invocation only.
    * These overrides do **not** suppress package-specific mandatory correctness flags, position-independent code flags, or build-system-required flags.
    * Package builders may still append package-specific flags after the override if necessary for the package to compile or function correctly.

14. **Linker injection strategy**

    * The linker command is advisory and build-system-aware, not blindly appended everywhere.
    * Build system adapters must translate the selected linker into the relevant mechanism for that ecosystem when supported:

      * CMake: prefer `-DCMAKE_*_LINKER_FLAGS=` and/or linker-related toolchain variables
      * Meson: use appropriate link arguments or environment variables
      * Autotools / configure / Make: use `LD=...`, compiler driver flags, or package-specific configure variables where supported
    * If a package build system does not reliably support the requested linker override, emit a warning and continue with the package’s default linker behavior rather than forcing a likely-broken configuration.

15. **Optimization philosophy**

    * The baseline target is maximum optimization on the current host CPU without knowingly breaking functionality.
    * Package builders may remove or alter individual optimization flags if a package is known to miscompile, fail LTO, or become unstable under a baseline flag.
    * The global optimization baseline is not absolute; correctness wins over aggressive optimization.
    * Builders for non-C/C++ ecosystems may inject analogous release optimizations. Example: a Cargo-based build may set `--release` and appropriate `RUSTFLAGS` where safe.

16. **Release-build requirement**

    * All supported package builds must target release/optimized modes rather than debug modes.
    * Examples:

      * CMake: `Release`
      * Meson: `release`
      * Cargo: `--release`
    * Package builders are responsible for using the canonical upstream mechanism for release builds.

17. **Environment propagation for vendored dependencies**

    * When building a package against vendored dependencies from `~/homecompiled/install/`, package builders must construct a dependency environment using the manifests of those dependency packages.
    * Supported environment variables for dependency propagation may include:

      * `PKG_CONFIG_PATH`
      * `CMAKE_PREFIX_PATH`
      * `PATH`
      * `LD_LIBRARY_PATH`
      * `LIBRARY_PATH`
      * `CPATH`
      * `C_INCLUDE_PATH`
      * `CPLUS_INCLUDE_PATH`
    * The exact subset is controlled by each dependency package’s declared safe link exports.

18. **Shell integration design**

    * `~/.hc-aliases` must be generated from installed manifests, not from the current run’s in-memory results alone.
    * This allows previously successful installs to remain available even if a later run fails.
    * The file must be fully regenerated on each invocation after all build attempts finish.
    * Regeneration must be atomic: write to a temporary file and rename into place.

19. **Executable aliases**

    * For packages that install end-user binaries, generate shell aliases that redirect standard command names to the homecompiled binaries.
    * Only generate aliases for binaries that actually exist in the installed prefix and are declared by that package’s manifest.
    * Examples include:

      * `alias grep='~/homecompiled/install/grep/bin/grep'`
      * `alias rg='~/homecompiled/install/ripgrep/bin/rg'`
      * `alias ffmpeg='~/homecompiled/install/ffmpeg/bin/ffmpeg'`
    * Do not generate aliases for binaries that were not produced successfully.
    * Do not attempt to alias every executable in a prefix automatically; use the package manifest’s explicit list.

20. **Library wrapper functions**

    * Library path wrappers must be shell functions, not aliases.
    * Function names use the library name prefixed with `hc-`, for example:

      * `hc-libopus`
      * `hc-libsvtav1`
      * `hc-zlib-ng`
    * These functions must be composable, so a command such as:

      * `hc-libopus hc-libsvtav1 ffmpeg`
        works by temporarily prepending the library-related environment variables for `libopus`, then invoking `hc-libsvtav1`, which temporarily prepends the variables for `libsvtav1`, then invokes `ffmpeg`.
    * Use shell temporary environment assignment syntax so the wrapper does not permanently mutate the current shell session.
    * The function body must invoke `"$@"` directly.

21. **Library export safety**

    * Not every library package is automatically safe to expose as a generic `hc-*` wrapper.
    * Each package manifest must explicitly declare whether it exports a safe library environment wrapper.
    * A safe export declaration includes:

      * wrapper name
      * prefix path
      * env vars to prepend
      * whether runtime lookup, build-time lookup, or both are appropriate
    * If a package does not declare a safe export, no `hc-*` function is generated for it.

22. **Build logs**

    * Each package build must produce a log file under `~/homecompiled/logs/`.
    * Use one log per package per run, preferably timestamped.
    * All executed commands should be logged in shell-escaped form.
    * Standard output and standard error of build steps must be captured in the log.
    * Console output may be summarized, but logs must remain complete enough for debugging.

23. **Preflight checks**

    * Before the first package build, verify that required host tools exist:

      * selected Python runtime
      * download tool path if external fallback is needed
      * requested compilers and linker commands
      * build-system-specific tools as required by a package, such as `cmake`, `meson`, `ninja`, `make`, `pkg-config`, `cargo`
    * Preflight must happen per package as well, because not all packages require the same host tooling.
    * Missing external host dependencies are package failures, not global hard failures, unless the missing tool is required for all selected packages.

24. **Failure handling**

    * A package failure does not abort the entire run.
    * A package whose vendored dependency failed must be marked as skipped due to failed dependency and not attempted.
    * The final summary must include:

      * successful builds
      * skipped packages due to invalid `--only`
      * skipped packages due to failed vendored dependencies
      * failed packages
      * packages for which checksum verification was unavailable
    * Exit code policy:

      * exit `0` if no valid package build failed
      * exit `1` if any valid selected package failed
      * invalid names in `--only` alone do not force a non-zero exit

25. **Manifest contract**

    * Each successful install must write a JSON manifest that includes:

      * package ID
      * version
      * source URL
      * checksum metadata if used
      * install prefix
      * binaries exported for aliasing
      * library export metadata for `hc-*` wrappers
      * dependency package IDs used during the build
      * effective toolchain commands
      * effective baseline and package-specific flags
      * build timestamp
    * The manifest is the sole source of truth for alias regeneration.

26. **Rebuild behavior**

    * In v1, selected packages are rebuilt on each invocation rather than skipped based on a cached success stamp.
    * This avoids incorrect reuse when compiler overrides or flag overrides change between runs.
    * Sources and archives may still be reused from cache.
    * Successful rebuilds replace prior installs atomically.

27. **Security considerations**

    * Never execute archive contents before checksum verification when checksum metadata exists.
    * Reject archive members with absolute paths, `..` traversal, or unsafe symlinks during extraction.
    * Never pass user-provided flag strings through a shell. Always pass them to `subprocess` as structured argument lists after `shlex.split`.
    * Never source arbitrary upstream scripts into the Python process.
    * Never trust installed manifests that fail schema/type validation when regenerating `~/.hc-aliases`; ignore them and emit a warning.

28. **Performance considerations**

    * Global build execution remains sequential by design.
    * Individual package builders may use the upstream build system’s local parallelism where appropriate, such as `make -jN`, `ninja -jN`, or equivalent.
    * The default local parallelism policy should be `os.cpu_count()` unless a package recipe requires a lower cap for stability or memory pressure.
    * Sequential global orchestration must still be preserved.

29. **Testing requirements**

    * Unit tests must cover:

      * package name resolution
      * dependency graph expansion
      * topological ordering
      * invalid `--only` handling
      * checksum verification
      * alias file generation
      * manifest read/write behavior
      * flag override merging
      * failure/skip summary logic
    * Integration-style tests should use fake package builders rather than invoking real upstream builds.
    * A small fixture registry with mock packages and dependency graphs should be used to exercise the resolver and failure semantics.

30. **Documentation requirements**

    * The README must explain:

      * Linux-only scope
      * dedicated installation directory
      * how to install the CLI
      * how to source `~/.hc-aliases`
      * what `hc-*` functions do
      * that the package registry is hard-coded
      * that system package-manager prerequisites are still required

## Section 5: Building, Testing, Profiling/Benchmarking and Running

Create a virtual environment and install the project in editable mode with development dependencies:

```bash
python3.12 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -e ".[dev]"
```

Suggested `pyproject.toml` development extras:

* runtime: `typer`, `rich`
* dev: `pytest`, `ruff`, `mypy`

Run formatting/linting/type checks:

```bash
ruff check .
ruff format .
mypy src/homecompiled
```

Run tests:

```bash
pytest
```

Run a focused test subset during development:

```bash
pytest tests/test_resolver.py tests/test_aliases.py -q
```

Run the CLI against all default packages:

```bash
python -m homecompiled
```

Run the CLI for only selected packages:

```bash
python -m homecompiled --only zlib-ng pigz ripgrep
```

Run with explicit compiler overrides:

```bash
python -m homecompiled --cc gcc --cxx g++ --fc gfortran --ld mold
```

Run with explicit flag overrides:

```bash
python -m homecompiled \
  --cflags "-O3 -march=native -mtune=native -flto=thin" \
  --fcflags "-O3 -march=native -mtune=native -flto=thin" \
  --ldflags "-Wl,-O2"
```

Source the generated shell integration file:

```bash
source ~/.hc-aliases
```

Use a homecompiled binary alias after sourcing:

```bash
ffmpeg -version
rg "pattern" .
```

Run a system binary with selected homecompiled libraries temporarily injected:

```bash
hc-libopus hc-libsvtav1 ffmpeg -encoders
```

Profiling and benchmarking:

* No benchmark orchestration is implemented in v1.
* No profiling command is required in v1.
* Reserve the `bench/` package namespace and a future CLI command namespace for v2.

## Section 6: v2 Features

The following items are explicitly reserved for v2 and must not be partially implemented in v1:

1. **Benchmark suite orchestration**

   * Add a dedicated benchmark command namespace, for example `homecompiled bench ...`.
   * Benchmark definitions remain package-specific, similar to build recipes.
   * The benchmark runner should be able to compare:

     * system binary/library behavior
     * homecompiled binary/library behavior
     * different compiler/flag variants where applicable

2. **Benchmark result persistence**

   * Store benchmark outputs under a dedicated directory, for example:

     * `~/homecompiled/benchmarks/`
   * Persist machine-readable JSON plus human-readable summaries.

3. **Package-specific benchmark contracts**

   * Each package module may optionally expose benchmark definitions separate from build definitions.
   * Benchmark inputs, commands, expected artifacts, and metric parsers should be hard-coded.

4. **Benchmark environment wrappers**

   * Reuse the manifest-driven library export mechanism so benchmarks can run against chosen homecompiled libraries without mutating the user’s shell permanently.

5. **Future CLI expansion**

   * Introduce subcommands cleanly without breaking the v1 default build workflow.
   * A likely future structure is:

     * `homecompiled build [OPTIONS]`
     * `homecompiled bench [OPTIONS]`

## Addendum A: Core Data Model

```text
BuildContext
- root_dir: Path                      # ~/homecompiled
- sources_dir: Path                   # ~/homecompiled/sources
- build_dir: Path                     # ~/homecompiled/build
- install_dir: Path                   # ~/homecompiled/install
- cache_dir: Path                     # ~/homecompiled/cache
- logs_dir: Path                      # ~/homecompiled/logs
- aliases_file: Path                  # ~/.hc-aliases
- cc: str
- cxx: str
- fc: str
- ld: str
- cflags: list[str]
- fcflags: list[str]
- ldflags: list[str]
- selected_packages: list[str]
- build_results: dict[str, BuildResult]
- run_timestamp: str
- jobs: int                           # derived internally, not exposed as CLI in v1

PackageSpec
- id: str
- display_name: str
- category: str
- version: str
- source_url: str
- checksum_algo: str | None
- checksum_value: str | None
- build_system: Literal["autotools", "cmake", "meson", "cargo", "make", "custom"]
- dependencies: list[str]
- binary_aliases: list[BinaryAlias]
- library_exports: list[LibraryExport]
- safe_link_exports: bool

BinaryAlias
- name: str                           # command name exposed in shell, e.g. rg
- relative_path: str                  # e.g. bin/rg

LibraryExport
- wrapper_name: str                   # e.g. hc-libopus
- prefix_subdir: str | None           # usually ""
- include_dirs: list[str]
- lib_dirs: list[str]
- pkgconfig_dirs: list[str]
- cmake_prefixes: list[str]
- path_dirs: list[str]
- runtime_enabled: bool
- build_enabled: bool

BuildResult
- package_id: str
- status: Literal["success", "failed", "skipped_invalid_only", "skipped_failed_dependency", "warning_only"]
- message: str
- log_file: Path | None
```

Implementation guidance:

* Use frozen dataclasses for `PackageSpec`, `BinaryAlias`, and `LibraryExport`.
* Use a base abstract class or Protocol for package builders.
* Keep `BuildContext` mutable for result accumulation and per-package execution state.

## Addendum B: Package Builder Interface

Each package module should follow this pattern:

```text
class PackageBuilder(ABC):
    spec: PackageSpec

    def preflight(self, ctx: BuildContext) -> None: ...
    def fetch(self, ctx: BuildContext) -> Path: ...
    def verify(self, ctx: BuildContext, archive_path: Path) -> None: ...
    def extract(self, ctx: BuildContext, archive_path: Path) -> Path: ...
    def configure(self, ctx: BuildContext, source_dir: Path, build_dir: Path, dep_env: dict[str, str]) -> list[list[str]]: ...
    def build(self, ctx: BuildContext, source_dir: Path, build_dir: Path, dep_env: dict[str, str]) -> list[list[str]]: ...
    def install(self, ctx: BuildContext, source_dir: Path, build_dir: Path, staging_prefix: Path, dep_env: dict[str, str]) -> list[list[str]]: ...
    def post_install(self, ctx: BuildContext, staging_prefix: Path) -> None: ...
    def create_manifest(self, ctx: BuildContext, staging_prefix: Path) -> dict: ...
```

Rules for implementers:

* `configure`, `build`, and `install` may return command lists to execute, or execute internally if the package genuinely requires bespoke flow. Prefer returning structured commands for standard cases.
* Package-specific modules should rely on shared adapters where possible, but they are allowed to fully customize the build when upstream requires it.
* Each builder is responsible for declaring which supported dependencies from the registry it consumes.
* Each builder must explicitly choose whether to expose library wrappers in the manifest.

## Addendum C: Alias File Contract

The generated `~/.hc-aliases` file must:

* begin with a generated-file header
* contain only aliases and shell functions
* avoid shell arrays or Bash-only features that break Zsh compatibility
* be regenerated from installed manifests

Illustrative structure:

```bash
# Generated by homecompiled. Do not edit by hand.

alias grep="$HOME/homecompiled/install/grep/bin/grep"
alias rg="$HOME/homecompiled/install/ripgrep/bin/rg"
alias ffmpeg="$HOME/homecompiled/install/ffmpeg/bin/ffmpeg"

hc-libopus() {
  LD_LIBRARY_PATH="$HOME/homecompiled/install/libopus/lib${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}" \
  LIBRARY_PATH="$HOME/homecompiled/install/libopus/lib${LIBRARY_PATH:+:$LIBRARY_PATH}" \
  PKG_CONFIG_PATH="$HOME/homecompiled/install/libopus/lib/pkgconfig${PKG_CONFIG_PATH:+:$PKG_CONFIG_PATH}" \
  CPATH="$HOME/homecompiled/install/libopus/include${CPATH:+:$CPATH}" \
  C_INCLUDE_PATH="$HOME/homecompiled/install/libopus/include${C_INCLUDE_PATH:+:$C_INCLUDE_PATH}" \
  CPLUS_INCLUDE_PATH="$HOME/homecompiled/install/libopus/include${CPLUS_INCLUDE_PATH:+:$CPLUS_INCLUDE_PATH}" \
  CMAKE_PREFIX_PATH="$HOME/homecompiled/install/libopus${CMAKE_PREFIX_PATH:+:$CMAKE_PREFIX_PATH}" \
  "$@"
}

hc-libsvtav1() {
  LD_LIBRARY_PATH="$HOME/homecompiled/install/libsvtav1/lib${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}" \
  LIBRARY_PATH="$HOME/homecompiled/install/libsvtav1/lib${LIBRARY_PATH:+:$LIBRARY_PATH}" \
  PKG_CONFIG_PATH="$HOME/homecompiled/install/libsvtav1/lib/pkgconfig${PKG_CONFIG_PATH:+:$PKG_CONFIG_PATH}" \
  CMAKE_PREFIX_PATH="$HOME/homecompiled/install/libsvtav1${CMAKE_PREFIX_PATH:+:$CMAKE_PREFIX_PATH}" \
  "$@"
}
```

Important behavior:

* Each function must end with `"$@"`.
* This allows nested wrappers such as `hc-libopus hc-libsvtav1 ffmpeg`.
* Functions must not `export` variables permanently into the caller’s shell.

## Addendum D: Initial Supported Package Registry and Dependency Intent

This addendum defines the required registry entries and the intended meaning of dependency declarations. The exact upstream build arguments remain package-module-specific.

```text
grep
- category: system-utils
- likely build system: autotools
- internal dependencies: []

ripgrep
- category: system-utils
- likely build system: cargo
- internal dependencies: []

tar
- category: system-utils
- likely build system: autotools
- internal dependencies: []

bzip2
- category: compression-utils
- likely build system: make/custom
- internal dependencies: []

xz
- category: compression-utils
- likely build system: autotools
- internal dependencies: []

zstd
- category: compression-utils
- likely build system: make/cmake
- internal dependencies: []

pigz
- category: compression-utils
- likely build system: make
- internal dependencies: ["zlib-ng"]   # when compat-mode linkage is enabled by the package builder

zlib-ng
- category: compression-utils
- likely build system: cmake
- internal dependencies: []
- special note: must build in zlib compatibility mode

libsvtav1
- category: multimedia-utils
- likely build system: cmake
- internal dependencies: []

dav1d
- category: multimedia-utils
- likely build system: meson
- internal dependencies: []

libsvtvp9
- category: multimedia-utils
- likely build system: cmake
- internal dependencies: []

x265
- category: multimedia-utils
- likely build system: cmake
- internal dependencies: []

x264
- category: multimedia-utils
- likely build system: custom/configure
- internal dependencies: []

flac
- category: multimedia-utils
- likely build system: autotools/cmake depending on pinned release
- internal dependencies: []

libopus
- category: multimedia-utils
- likely build system: autotools/meson depending on pinned release
- internal dependencies: []

libjpeg-turbo
- category: multimedia-utils
- likely build system: cmake
- internal dependencies: []

libjxl
- category: multimedia-utils
- likely build system: cmake
- internal dependencies: ["libjpeg-turbo"]   # only if the package builder elects to use the homecompiled turbojpeg path

libvips
- category: multimedia-utils
- likely build system: meson
- internal dependencies: ["libjpeg-turbo", "libjxl"]   # only where safe and available

ffmpeg
- category: multimedia-utils
- likely build system: custom/configure
- internal dependencies:
  - "zlib-ng"
  - "libsvtav1"
  - "dav1d"
  - "libsvtvp9"
  - "x265"
  - "x264"
  - "flac"
  - "libopus"
  - "libjxl"           # enable only if package builder supports a clean integration
- special note: package builder must decide enable/disable flags carefully to avoid over-declaring unsupported features

mpv
- category: gui-programs
- likely build system: meson
- internal dependencies:
  - "ffmpeg"

handbrake
- category: gui-programs
- likely build system: custom/contrib
- internal dependencies:
  - "ffmpeg"
  - "libsvtav1"
  - "dav1d"
  - "x265"
  - "x264"
  - "libopus"
- special note: package builder may need to opt out of some internal edges if upstream’s build system already vendors or constrains those dependencies differently

qbittorrent
- category: gui-programs
- likely build system: cmake
- internal dependencies: []

obs-studio
- category: gui-programs
- likely build system: cmake
- internal dependencies:
  - "ffmpeg"
  - "x264"
  - "libopus"

darktable
- category: gui-programs
- likely build system: cmake
- internal dependencies:
  - "libjpeg-turbo"
  - "libjxl"
- special note: many other non-registry dependencies remain system-provided

audacity
- category: gui-programs
- likely build system: cmake
- internal dependencies:
  - "ffmpeg"
  - "flac"
  - "libopus"

openblas
- category: misc
- likely build system: make/cmake
- internal dependencies: []

fftw
- category: misc
- likely build system: autotools/cmake
- internal dependencies: []
```

Rules for interpreting this addendum:

* These are the intended internal dependency edges within the supported `homecompiled` registry.
* They do not replace per-package builder authority.
* If upstream reality for a pinned release requires an adjustment, the package module is the final source of truth and must document the reason in code comments.
* External dependencies outside the registry remain system prerequisites.

## Addendum E: Build Pipeline

For each selected package after dependency resolution, the executor should perform this pipeline:

1. Load package builder from registry.
2. Run package-specific preflight checks.
3. Resolve vendored dependency manifests from already built dependency packages.
4. Download archive into cache if not already present.
5. Verify checksum if metadata exists.
6. Extract archive into a clean build workspace.
7. Create package/version-specific staging prefix and log file.
8. Construct dependency environment from dependency manifests.
9. Construct package toolchain environment from CLI defaults and overrides.
10. Run configure phase.
11. Run build phase.
12. Run install phase into staging prefix.
13. Run post-install validation:

    * required binaries exist
    * declared library paths exist if exported
14. Generate manifest in staging prefix.
15. Atomically replace existing install prefix with staging prefix.
16. Record success in the run summary.

On failure:

1. Mark package as failed.
2. Preserve previous install prefix untouched.
3. Mark downstream dependents as skipped due to failed vendored dependency.
4. Continue with the next resolvable package.

This exact pipeline should be reflected in the implementation structure and test suite.
