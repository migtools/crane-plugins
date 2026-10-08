# AGENTS.md

## What is this repository

This is the **plugin index** for [crane](https://github.com/konveyor/crane) — a Kubernetes migration tool. It contains YAML manifest files that register crane transform plugins and point to their binary download URLs, plus documentation for creating new plugins. There is no plugin implementation source code here.

Crane uses this index to discover and install plugins via `crane plugin-manager`.

## Repository structure

```text
crane-plugins/
├── index.yaml                      # top-level index listing all plugins
├── plugins/
│   ├── OpenShift/index.yaml        # manifest for OpenShiftPlugin
│   └── ImageStream/index.yaml      # manifest for ImageStreamPlugin
└── docs/
    └── creating-crane-transform-plugin.md   # full guide for creating new plugins
```

## Plugin index format

### Top-level index (`index.yaml`)

```yaml
apiServer: crane.konveyor.io/v1alpha1
kind: PluginIndex
plugins:
- name: PluginName
  path: https://raw.githubusercontent.com/org/crane-plugins/main/plugins/PluginName/index.yaml
```

Each entry points to a raw-URL-accessible plugin manifest file.

### Plugin manifest (`plugins/<Name>/index.yaml`)

```yaml
apiServer: crane.konveyor.io/v1alpha1
kind: Plugin
versions:
- name: PluginName
  shortDescription: Short description
  description: Longer description of what the plugin does
  version: v0.1.0
  optionalFields:
  - flagName: my-flag
    help: "Description of the flag"
    example: "key1=val1,key2=val2"
  binaries:
  - os: linux
    arch: amd64
    uri: https://github.com/org/repo/releases/download/v0.1.0/amd64-linux-pluginname-v0.1.0
    sha: <sha256>  # optional but recommended
  - os: darwin
    arch: amd64
    uri: https://github.com/org/repo/releases/download/v0.1.0/amd64-darwin-pluginname-v0.1.0
    sha: <sha256>  # optional but recommended
  - os: darwin
    arch: arm64
    uri: https://github.com/org/repo/releases/download/v0.1.0/arm64-darwin-pluginname-v0.1.0
    sha: <sha256>  # optional but recommended
```

Multiple versions can be listed under `versions:`. Each version has its own binary URLs and optional field definitions.

## Adding a new plugin to the index

1. Create directory `plugins/<YourPluginName>/`
2. Create `plugins/<YourPluginName>/index.yaml` with the manifest format above
3. Add an entry to the root `index.yaml` pointing to the raw URL of the new manifest
4. The binary URIs must be HTTP-accessible (typically GitHub release assets)

## Creating a new crane transform plugin

See [docs/creating-crane-transform-plugin.md](docs/creating-crane-transform-plugin.md) for a complete guide covering:
- Plugin architecture and stdin/stdout JSON protocol
- Core types (`PluginRequest`, `PluginResponse`, `PluginMetadata`)
- CLI harness wiring (`cli.RunAndExit`)
- Two implementation patterns: simple (whiteout/patch) and complex (resource conversion)
- Testing, building, cross-compiling, and releasing

## Related repositories

| Repository | Purpose |
|-----------|---------|
| [crane](https://github.com/konveyor/crane) | Main crane CLI tool |
| [crane-lib](https://github.com/konveyor/crane-lib) | Plugin interface, CLI harness, types |
| [crane-plugin-openshift](https://github.com/migtools/crane-plugin-openshift) | Reference plugin — simple whiteout/patch pattern |
| [crane-plugin-buildconfig-to-shipwright](https://github.com/migtools/crane-plugin-buildconfig-to-shipwright) | Complex plugin — resource conversion with NewResources |
