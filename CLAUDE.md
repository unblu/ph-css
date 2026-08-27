# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

ph-css is a Java-based CSS 3 parser and builder library (v8.2.2-SNAPSHOT). It parses CSS into a Java object model, supports traversal/modification via the visitor pattern, and can serialize back to CSS. The companion module `ph-csscompress-maven-plugin` provides build-time CSS compression.

## Fork Context

This repository is unblu's fork of [phax/ph-css](https://github.com/phax/ph-css). It has two long-lived branches:

| Branch | Content |
|--------|---------|
| `master` | 1:1 mirror of the upstream `phax/ph-css` `master`. Nothing is committed here directly. |
| `develop` | Default working branch, holding everything this fork adds on top of upstream. Fork releases are tagged here. |

Work branches (`feature/...`, `<issue>-<slug>`) are created from `develop` and merged back into `develop` by pull request. Upstream changes land on `master` and are merged from there into `develop`.

### Branch-specific POM coordinates

`develop` carries fork-specific coordinates so the produced artifacts never collide with the upstream ones:

- `groupId`: `unblu.patched.com.helger` (upstream uses `com.helger`)
- `version`: `8.2.2-SNAPSHOT`, released as `<upstream-version>-unblu-<n>` (for example `8.1.2-unblu-3`)

`master` keeps the upstream coordinates. That is why the promotion workflow restores the three `pom.xml` files from `master` when it merges `develop` into `master`.

### POM coordinates and the project version

The fork renames the coordinates to `unblu.patched.com.helger` in all three POMs *and* in the
`dependencyManagement` block of the root POM. Keeping the managed entries renamed matters: they are what
lets `ph-csscompress-maven-plugin` depend on `ph-css` without a `<version>` of its own, exactly as upstream
does. The project version therefore appears exactly three times - in `pom.xml`, and in the `<parent>` block
of each of the two modules:

```bash
grep -rn -- "-SNAPSHOT" --include=pom.xml .
```

If a managed entry ever falls back to `com.helger` - for instance through a `master` -> `develop` merge
that brings upstream's POMs back - the plugin module stops resolving and has to spell its `ph-css` version
out again. Such a hardcoded version does not follow the next upstream version bump, because the merge keeps
our side of the dependency block, and the build then fails on the last module with:

```
[ERROR] Failed to execute goal on project ph-csscompress-maven-plugin: Could not resolve dependencies
[ERROR] dependency: unblu.patched.com.helger:ph-css:jar:8.1.2-SNAPSHOT (compile)
```

So after merging `master` into `develop`, check both the three versions and the group ids of the two
managed entries.

### Automation (`.github/workflows/`)

| Workflow | Trigger | Effect |
|----------|---------|--------|
| `sync-upstream.yml` | nightly + manual | Force-pushes `upstream/master` onto `sync/upstream-master` and opens a PR against `master`. |
| `promote-develop.yml` | a PR is merged into `develop`, or a `develop` -> `master` PR is opened | Rebuilds `promotion/develop-to-master` from `master`, merges `develop` into it, restores the `master` POMs, opens the PR against `master` and enables squash auto-merge. A manually opened `develop` -> `master` PR is closed in favour of it. |
| `validate-master-pr.yml` | PR targeting `master` | Only `promotion/develop-to-master` and `sync/upstream-master` may target `master`; anything else - including a direct `develop` PR - fails. |
| `release-to-slack.yml` | GitHub release published | Announces the release on a chat webhook. |
| `maven.yml` | push / PR | Upstream's build matrix (Java 17, 21, 25). |

In short: upstream changes flow in via `master` -> `develop`, and changes made here flow out via `develop` -> `promotion/develop-to-master` -> `master`, from where they can be proposed upstream.

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
