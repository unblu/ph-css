# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

ph-css is a Java-based CSS 3 parser and builder library (v8.2.2-SNAPSHOT). It parses CSS into a Java object model, supports traversal/modification via the visitor pattern, and can serialize back to CSS. The companion module `ph-csscompress-maven-plugin` provides build-time CSS compression.

## Fork Context

This repository is unblu's fork of [phax/ph-css](https://github.com/phax/ph-css). Two branches are
permanent, the others are created and force-updated by the automation:

| Branch | Content |
|--------|---------|
| `master` | Mirror of the upstream `phax/ph-css` `master`, plus the fork-only files (see below). Nothing is committed here directly. |
| `develop` | Default working branch, holding everything this fork adds on top of upstream. Fork releases are tagged here. |
| `sync/upstream-master` | Automation: upstream's `master`, waiting to be merged into `master`. |
| `promotion/develop-to-master` | Automation: `master` with `develop` merged into it, waiting to be merged into `master`. |
| `promotion/master-to-upstream` | Automation: lives in `phax/ph-css`; `upstream/master` with our `master` merged into it, proposed there as a pull request. |

Work branches (`feature/...`, `<issue>-<slug>`) are created from `develop` and merged back into `develop`
by pull request. Everything else is automated: each hop between `develop`, `master` and upstream is done by
a workflow, not by hand.

### POM coordinates and the project version

`develop` carries fork-specific coordinates so that the produced artifacts never collide with the upstream
ones:

- `groupId`: `unblu.patched.com.helger` instead of `com.helger`, in the three POMs *and* in the
  `dependencyManagement` block of the root POM
- `version`: `8.2.2-SNAPSHOT`, released as `<upstream-version>-unblu-<n>` (for example `8.1.2-unblu-3`)

`master` keeps the upstream coordinates, so the three `pom.xml` files are never promoted in either
direction: `promote-develop.yml` restores them from `master`, `sync-master-to-develop.yml` restores them
from `develop`.

Keeping the managed entries renamed matters: they are what lets `ph-csscompress-maven-plugin` depend on
`ph-css` without a `<version>` of its own, exactly as upstream does. The project version therefore appears
exactly three times - in `pom.xml`, and in the `<parent>` block of each of the two modules:

```bash
grep -rn -- "-SNAPSHOT" --include=pom.xml .
```

If a managed entry ever falls back to `com.helger` - for instance through a merge that brings upstream's
POMs back - the plugin module stops resolving and has to spell its `ph-css` version out again. Such a
hardcoded version does not follow the next upstream version bump, because the merge keeps our side of the
dependency block, and the build then fails on the last module with:

```
[ERROR] Failed to execute goal on project ph-csscompress-maven-plugin: Could not resolve dependencies
[ERROR] dependency: unblu.patched.com.helger:ph-css:jar:8.1.2-SNAPSHOT (compile)
```

So after any merge into `develop`, check both the three versions and the group ids of the two managed
entries.

### Automation (`.github/workflows/`)

| Workflow | Trigger | Effect |
|----------|---------|--------|
| `sync-upstream.yml` | nightly (03:00 UTC) + manual | Force-pushes `upstream/master` onto `sync/upstream-master` and opens a PR against `master`. |
| `validate-master-pr.yml` | PR targeting `master` | Only `promotion/develop-to-master` and `sync/upstream-master` may target `master`; anything else - including a direct `develop` PR - fails. |
| `promote-develop.yml` | a PR is merged into `develop`, or a `develop` -> `master` PR is opened | Rebuilds `promotion/develop-to-master` from `master`, merges `develop` into it, restores the `master` POMs, force-pushes it, opens the PR against `master` and enables auto-merge. A manually opened `develop` -> `master` PR is closed in favour of it. |
| `sync-master-to-develop.yml` | the `promotion/develop-to-master` PR is merged into `master` | Merges `master` back into `develop`, restoring the `develop` POMs, and pushes `develop`. This keeps the two branches identical apart from the POMs. |
| `promote-to-upstream.yml` | a PR from this fork's `master` to `phax/ph-css` `master` | Rebuilds `promotion/master-to-upstream` from `upstream/master`, merges our `master` into it, restores `.github/workflows/` and `CLAUDE.md` from upstream, force-pushes the branch into `phax/ph-css` and opens (or updates) the upstream PR with the source PR's title, body and a back-link. The manually opened PR is closed. |
| `release-to-slack.yml` | GitHub release published | Announces the release on a chat webhook. |
| `maven.yml` | push / PR | Upstream's build matrix (Java 17, 21, 25). |

Two things to know before touching these workflows:

- The promotion and sync-back workflows push with a GitHub App installation token
  (`ph-css-promotion[bot]`), not with `github.token`. A push made with the default token does not trigger
  the next workflow, which would break the chain. `promote-to-upstream.yml` writes to `phax/ph-css` with
  the `UPSTREAM_TOKEN` secret.
- `.github/workflows/` and this `CLAUDE.md` are fork-only. `promote-to-upstream.yml` restores them from
  upstream, so they never show up in an upstream pull request.

A full round trip:

1. `sync-upstream.yml` brings new upstream commits onto `master`.
2. Work happens on `develop`, through work-branch pull requests.
3. `promote-develop.yml` carries `develop` onto `master` via `promotion/develop-to-master`.
4. `sync-master-to-develop.yml` merges `master` back into `develop`.
5. Opening a pull request from our `master` to `phax/ph-css` triggers `promote-to-upstream.yml`, which
   turns it into a clean upstream pull request.

### Releasing

Releases are cut from `develop`, never from `master`:

1. Merge the work branch into `develop`.
2. Commit the version bump to `<upstream-version>-unblu-<n>`.
3. Tag that commit and publish the GitHub release.
4. Build with Java 17+ (`mvn clean install`) and publish the resulting `ph-css-parent-pom` and `ph-css` artifacts (POM, JAR, sources JAR) to the consuming project's internal Maven repository. Nothing from this fork is published to Maven Central.

## Build Commands

```bash
# Full build (requires Java 17+, Maven 3.x)
mvn clean install

# Run all tests
mvn test

# Run a single test class
mvn -pl ph-css test -Dtest=CSSReaderFuncTest

# Run a single test method
mvn -pl ph-css test -Dtest=CSSReaderFuncTest#testReadBadButSucceeding

# Build only the main library
mvn -pl ph-css clean install

# Build only the Maven plugin
mvn -pl ph-csscompress-maven-plugin clean install

# Check license headers
mvn license:check
```

## Module Structure

- **`ph-css/`** - Core library: CSS parsing, object model, and serialization
- **`ph-csscompress-maven-plugin/`** - Maven Mojo for CSS compression at build time

## Architecture

### Parser (JavaCC-generated)

Grammar files in `ph-css/src/main/jjtree/`:
- `ParserCSS30.jjt` - Main CSS 3.0 grammar
- `ParserCSSCharsetDetector.jjt` - Charset detection

JavaCC generates parser sources into `target/generated-sources/jjtree` and `target/generated-sources/javacc`. Do not edit generated parser files directly; modify the `.jjt` grammars instead.

### Core Package Layout (`com.helger.css`)

| Package | Purpose |
|---------|---------|
| `decl` | CSS object model: `CascadingStyleSheet`, style/media/font-face/keyframes rules, declarations |
| `decl.visit` | Visitor pattern interfaces and implementations for CSS traversal |
| `decl.shorthand` | Shorthand CSS property expansion |
| `reader` | `CSSReader` (full stylesheets), `CSSReaderDeclarationList` (inline styles) |
| `reader.errorhandler` | Parse error handlers (logging, collecting, throwing) |
| `writer` | `CSSWriter` for serializing the object model back to CSS text |
| `parser` | JavaCC-generated parser classes (do not edit manually) |
| `property` | CSS property definitions and metadata (`ECSSProperty`, `CCSSProperties`) |
| `propertyvalue` | Typed CSS values (colors, functions, URIs, etc.) |
| `media` | Media query model and `ECSSMedium` enum |
| `tools` | Utilities like `MediaQueryTools` |
| `utils` | Helpers for colors, numbers, URLs, rectangles |
| `annotation` | Custom annotations (`@DeprecatedInCSS`) |
| `handler` | Exception handlers for the parser |

### Read/Write Flow

1. **Reading**: `CSSReader.readFromString/File/Stream()` -> JavaCC parser -> `CascadingStyleSheet` object model
2. **Manipulation**: Visitor pattern via `ICSSVisitor` / `DefaultCSSVisitor` on the object model
3. **Writing**: `CSSWriter.getCSSAsString(CascadingStyleSheet)` -> CSS text output, controlled by `CSSWriterSettings` (minified/formatted, version)

### Key Dependencies

- `ph-commons` (12.3.5) - Collection types (`ICommonsList`, `CommonsArrayList`), I/O utilities, type conversion
- `ph-javacc-maven-plugin` (5.0.1) - Parser generation from `.jjt` grammars
- JUnit 4 for tests

## Test Resources

CSS test files live in `ph-css/src/test/resources/testfiles/css30/`:
- `good/` - Valid CSS files (used to verify successful parsing)
- `bad/` - Invalid CSS files (expected to fail parsing)
- `bad_but_succeeding/` - Invalid CSS that the parser recovers from
- `bad_but_browsercompliant/` - Invalid CSS that browsers accept

## Coding Conventions

See the global rules in `~/.claude/rules/naming.md` for Hungarian notation, formatting, and naming. Key project-specific points:
- OSGi bundle packaging (`<packaging>bundle</packaging>`) via `maven-bundle-plugin`
- Apache 2.0 license header required on all Java files (enforced by `license-maven-plugin`)
- Forbidden APIs plugin blocks unsafe/deprecated JDK usage
