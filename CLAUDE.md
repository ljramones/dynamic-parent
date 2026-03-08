# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

`dynamis-parent` is the shared parent POM (`org.dynamisengine:dynamis-parent`) for the Dynamis Engine ecosystem. It contains no source code — only Maven build configuration inherited by all Dynamis libraries (Vectrix, MeshForge, DynamisAI, etc.).

## Build Commands

```bash
mvn clean install              # Build and install to local Maven repo
mvn -Prelease clean verify     # Pre-deploy verification (GPG signing + source/javadoc jars)
mvn -Prelease deploy           # Publish to Maven Central via Sonatype Central Portal
```

Convenience scripts: `./build.sh` (install), `./deploy.sh` (verify + deploy).

## Key Details

- **JDK 25** (`maven.compiler.release=25`)
- **GPG signing** is always active (pinentry loopback mode)
- **Release profile** (`-Prelease`) activates `central-publishing-maven-plugin` for Maven Central publishing; requires Sonatype credentials in `~/.m2/settings.xml`
- This POM must be installed locally (`mvn install`) before building any downstream Dynamis component that declares it as `<parent>`

## What Changes Here Affect

Any change to plugin versions, compiler settings, or build configuration propagates to all child projects across the ecosystem. Test changes locally against at least one downstream project (e.g., Vectrix) before publishing.
