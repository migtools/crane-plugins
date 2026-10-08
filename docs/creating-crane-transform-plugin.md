# Creating a Crane Transform Plugin

A complete guide for building crane transform plugins. This document is designed to be consumed by both humans and AI coding assistants (Claude Code, OpenAI Codex, Cursor, etc.).

## Overview

[Crane](https://github.com/konveyor/crane) is a Kubernetes migration tool with three phases:

1. **Export** — dumps all resources from a namespace into YAML files
2. **Transform** — runs plugins over each resource to modify, filter, or generate resources
3. **Apply** — writes the final output ready for `kubectl apply`

Transform plugins are **standalone Go binaries** that crane discovers in a `--plugin-dir` directory and invokes once per resource via stdin/stdout JSON.

## Plugin protocol

Crane communicates with plugins exclusively through **stdin/stdout JSON**. All logging goes to **stderr** (stdout is reserved for protocol).

### Metadata request

Crane sends an **empty JSON object** `{}` on stdin. The plugin must respond with `PluginMetadata` JSON on stdout. This lets crane discover what the plugin is called and what flags it accepts.

### Transform request

Crane sends a `PluginRequest` on stdin — a single Kubernetes resource (as unstructured JSON) plus an `extras` map containing flag values. The plugin responds with a `PluginResponse`.

## Core types

All types live in `github.com/konveyor/crane-lib/transform`:

```go
// The interface your plugin implements
type Plugin interface {
    Run(PluginRequest) (PluginResponse, error)
    Metadata() PluginMetadata
}

// What crane sends (one Kubernetes resource per call)
type PluginRequest struct {
    unstructured.Unstructured `json:",inline"`  // the K8s resource
    Extras map[string]string  `json:"extras,omitempty"` // flag values from --optional-flags
}

// What your plugin returns
type PluginResponse struct {
    Version      string                        `json:"version,omitempty"`
    IsWhiteOut   bool                          `json:"isWhiteOut,omitempty"`   // true = delete this resource
    Patches      jsonpatch.Patch               `json:"patches,omitempty"`     // JSON patches to apply (RFC 6902)
    NewResources []unstructured.Unstructured   `json:"newResources,omitempty"` // generate new resources
}

// Returned on metadata request (empty {} input)
type PluginMetadata struct {
    Name            string           `json:"name"`
    Version         string           `json:"version"`
    RequestVersion  []Version        `json:"requestVersion"`
    ResponseVersion []Version        `json:"responseVersion"`
    OptionalFields  []OptionalFields `json:"optionalFields,omitempty"`
}

// Declares a CLI flag the user can pass via --optional-flags
type OptionalFields struct {
    FlagName string `json:"flagName"`
    Help     string `json:"help"`
    Example  string `json:"example"`
}
```

### Response semantics

A plugin can do one of four things per resource:

| Action | Response fields |
|--------|----------------|
| Pass through (ignore) | Return empty `PluginResponse{}` |
| Modify in-place | Set `Patches` with JSON patch operations |
| Delete (whiteout) | Set `IsWhiteOut: true` |
| Replace with new resource(s) | Set `IsWhiteOut: true` + `NewResources` |

**Important:** `NewResources` requires the latest version of crane-lib from GitHub main branch. Use `go get github.com/konveyor/crane-lib@main` to get it.

## CLI harness

The `transform/cli` package handles all stdin/stdout protocol. Your `main()` is minimal:

```go
package main

import (
    "github.com/yourorg/crane-plugin-myplugin/myplugin"
    "github.com/konveyor/crane-lib/transform/cli"
    "github.com/sirupsen/logrus"
)

func main() {
    plugin := &myplugin.MyTransformPlugin{Log: logrus.New()}
    meta := plugin.Metadata()
    cli.RunAndExit(cli.NewCustomPlugin(meta.Name, myplugin.PluginVersion, meta.OptionalFields, plugin.Run))
}
```

`cli.RunAndExit` handles:
- Reading JSON from stdin
- Detecting metadata vs transform requests (empty `{}` → metadata)
- Calling your `Run()` function
- Writing JSON response to stdout
- Writing errors to stderr and exiting with code 1

## Helper functions

```go
// Parse "key1=val1,key2=val2" into map[string]string
transform.ParseOptionalFieldMapVal(extras["my-map-flag"])

// Parse "a,b,c" into []string
transform.ParseOptionalFieldSliceVal(extras["my-list-flag"])
```

## Project structure

```text
crane-plugin-YOURNAME/
├── main.go                    # entry point — wires plugin to cli.RunAndExit
├── go.mod
├── yourpkg/
│   ├── plugin.go              # Plugin struct, Metadata(), Run(), ParseOptionalFields()
│   ├── yourpkg_test.go        # tests
│   └── (additional files)     # conversion logic, helpers
└── README.md
```

The binary name **must** start with `crane-plugin-` for crane to discover it in the plugin directory.

## Pattern A: Simple plugin (whiteout/patch)

Reference implementation: [crane-plugin-openshift](https://github.com/migtools/crane-plugin-openshift)

This pattern inspects resource kind/group and either whiteouts, patches, or passes through. Suitable for filtering, stripping fields, renaming references, etc.

### plugin.go

```go
package myplugin

import (
    jsonpatch "github.com/evanphx/json-patch"
    "github.com/konveyor/crane-lib/transform"
    "github.com/sirupsen/logrus"
)

const PluginVersion = "v0.1.0"

type MyTransformPlugin struct {
    Log logrus.FieldLogger
}

func (p *MyTransformPlugin) Metadata() transform.PluginMetadata {
    return transform.PluginMetadata{
        Name:    "MyPlugin",
        Version: PluginVersion,
        OptionalFields: []transform.OptionalFields{
            {
                FlagName: "registry-replacement",
                Help:     "Map of image registry paths to replace: old1=new1,old2=new2",
                Example:  "docker.io/old=quay.io/new",
            },
        },
        RequestVersion:  []transform.Version{transform.V1},
        ResponseVersion: []transform.Version{transform.V1},
    }
}

func (p *MyTransformPlugin) Run(request transform.PluginRequest) (transform.PluginResponse, error) {
    u := request.Unstructured
    var patch jsonpatch.Patch
    whiteOut := false

    // Parse optional fields
    var registryMap map[string]string
    if val := request.Extras["registry-replacement"]; len(val) > 0 {
        registryMap = transform.ParseOptionalFieldMapVal(val)
    }

    // Filter by Group/Version/Kind to handle only specific resources
    gvk := u.GroupVersionKind()
    
    switch {
    case gvk.Group == "example.io" && gvk.Version == "v1" && gvk.Kind == "ResourceToDelete":
        whiteOut = true
    case gvk.Group == "example.io" && gvk.Version == "v1" && gvk.Kind == "ResourceToModify":
        // Check if the annotation exists before creating remove patch (RFC 6902 requires target to exist)
        annotations := u.GetAnnotations()
        if _, exists := annotations["unwanted-key"]; !exists {
            // Annotation doesn't exist, nothing to remove - pass through
            return transform.PluginResponse{}, nil
        }
        // Build JSON patches (RFC 6902)
        patchJSON := []byte(`[{"op":"remove","path":"/metadata/annotations/unwanted-key"}]`)
        var err error
        patch, err = jsonpatch.DecodePatch(patchJSON)
        if err != nil {
            return transform.PluginResponse{}, err
        }
    default:
        // pass through
    }

    return transform.PluginResponse{
        Version:    string(transform.V1),
        IsWhiteOut: whiteOut,
        Patches:    patch,
    }, nil
}

func (p *MyTransformPlugin) log() logrus.FieldLogger {
    if p.Log != nil {
        return p.Log
    }
    return logrus.New()
}
```

### How the OpenShift plugin works (real-world example)

The OpenShift plugin (`crane-plugin-openshift`) handles many resource kinds in a single `Run()` function:

- **Whiteouts**: Build, ImageStream, ImageStreamTag, ImageTag, default ServiceAccounts (builder/deployer), default RoleBindings, default CA bundle ConfigMap
- **Patches**: BuildConfig (pull secret replacement, registry replacement), DeploymentConfig (PVC renames, pod template transforms), Route (remove auto-generated hostname), ServiceAccount (strip default secrets), RoleBinding (remove namespace from subjects), Pod (strip SCC security context)
- **Pass-through**: everything else

Optional flags control behavior: `strip-default-rbac`, `strip-default-cabundle`, `strip-default-pull-secrets`, `pull-secret-replacement`, `registry-replacement`, `pvc-rename-map`.

## Pattern B: Complex plugin (resource conversion)

Reference implementation: [crane-plugin-buildconfig-to-shipwright](https://github.com/migtools/crane-plugin-buildconfig-to-shipwright)

This pattern converts one resource type into a completely different one using `IsWhiteOut` + `NewResources`. Suitable for migrating between API versions, converting vendor-specific resources to standard ones, etc.

### Key differences from Pattern A

1. **Unmarshal to typed structs** — instead of working with `unstructured.Unstructured` and JSON patches, unmarshal into typed Go structs from the source API
2. **Build target typed structs** — construct the new resource using typed Go structs from the target API
3. **Convert back to unstructured** — use `runtime.DefaultUnstructuredConverter.ToUnstructured()` to produce `unstructured.Unstructured` for `NewResources`
4. **Multiple new resources** — a single input resource can produce multiple outputs (e.g., BuildConfig → Build + ServiceAccount)

### plugin.go (conversion pattern)

```go
package converter

import (
    "encoding/json"
    "fmt"

    "github.com/konveyor/crane-lib/transform"
    "github.com/sirupsen/logrus"
    "k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
    "k8s.io/apimachinery/pkg/runtime"

    sourcev1 "some.api/source/v1"      // source typed API
    targetv1 "some.api/target/v1beta1" // target typed API
)

const PluginVersion = "v0.1.0"

type ConverterPlugin struct {
    Log logrus.FieldLogger
}

func (p *ConverterPlugin) Metadata() transform.PluginMetadata {
    return transform.PluginMetadata{
        Name:    "ConverterPlugin",
        Version: PluginVersion,
        OptionalFields: []transform.OptionalFields{
            {
                FlagName: "mapping",
                Help:     "Mapping of source references to target references",
                Example:  "old-ref=new-ref",
            },
        },
        RequestVersion:  []transform.Version{transform.V1},
        ResponseVersion: []transform.Version{transform.V1},
    }
}

func (p *ConverterPlugin) Run(request transform.PluginRequest) (transform.PluginResponse, error) {
    u := request.Unstructured

    // Filter: only handle specific resource types (complete GVK match)
    gvk := u.GroupVersionKind()
    if gvk.Group != "source.api.io" || gvk.Version != "v1" || gvk.Kind != "SourceKind" {
        return transform.PluginResponse{}, nil
    }

    // Parse optional fields
    opts, err := ParseOptionalFields(request.Extras)
    if err != nil {
        return transform.PluginResponse{}, err
    }

    // Unmarshal to typed struct
    var source sourcev1.SourceResource
    raw, _ := json.Marshal(u.Object)
    if err := json.Unmarshal(raw, &source); err != nil {
        return transform.PluginResponse{}, fmt.Errorf("unmarshaling source: %w", err)
    }

    // Convert
    target, err := convert(&source, opts)
    if err != nil {
        return transform.PluginResponse{}, err
    }

    // Convert to unstructured
    targetUnstructured, err := toUnstructured(target)
    if err != nil {
        return transform.PluginResponse{}, err
    }

    // Ensure GVK is set (in case convert doesn't populate TypeMeta)
    targetUnstructured.SetAPIVersion("target.api/v1beta1")
    targetUnstructured.SetKind("TargetKind")

    return transform.PluginResponse{
        Version:      string(transform.V1),
        IsWhiteOut:   true,
        NewResources: []unstructured.Unstructured{*targetUnstructured},
    }, nil
}

func toUnstructured(obj interface{}) (*unstructured.Unstructured, error) {
    data, err := runtime.DefaultUnstructuredConverter.ToUnstructured(obj)
    if err != nil {
        return nil, err
    }
    return &unstructured.Unstructured{Object: data}, nil
}
```

### How the BuildConfig-to-Shipwright plugin works (real-world example)

This plugin converts OpenShift `BuildConfig` (`build.openshift.io/v1`) to Shipwright `Build` (`shipwright.io/v1beta1`):

- **Strategy mapping**: Docker → `buildah` ClusterBuildStrategy, Source (S2I) → `source-to-image`
- **Source processing**: Git URL/revision/cloneSecret, contextDir, binary/image sources produce warnings
- **Output processing**: image reference, pushSecret — with offline ImageStream resolution via flags
- **Registry config**: search/insecure/block registries mapped to Shipwright paramValues
- **ServiceAccount generation**: when pull secrets are referenced, generates a separate ServiceAccount as an additional NewResource
- **Offline ImageStream resolution**: instead of calling the live cluster API (like the original `crane convert`), uses `--imagestream-mapping` and `--registry-mapping` flags with fallback to internal OpenShift registry URL

Optional flags: `registry-mapping`, `imagestream-mapping`, `default-build-strategy`, `search-registries`, `insecure-registries`, `block-registries`.

## go.mod

### Simple plugin (Pattern A)

```go
module github.com/yourorg/crane-plugin-myplugin

go 1.24

require (
    github.com/evanphx/json-patch v5.9.11+incompatible
    github.com/konveyor/crane-lib v0.1.5
    github.com/sirupsen/logrus v1.9.4
    k8s.io/apimachinery v0.28.0
)
```

### Conversion plugin (Pattern B) — needs NewResources

```go
module github.com/yourorg/crane-plugin-converter

go 1.24

require (
    github.com/konveyor/crane-lib v0.1.6-0.20260818123419-d279d85c1dd1  // Get latest via: go get github.com/konveyor/crane-lib@main
    github.com/sirupsen/logrus v1.9.4
    k8s.io/apimachinery v0.28.0
    // Add typed API deps as needed:
    // github.com/openshift/api v0.0.0-...
    // github.com/shipwright-io/build v0.17.0
    // k8s.io/api v0.28.0
)
```

## Testing

### Test that irrelevant resources are passed through

```go
func TestRunSkipsIrrelevantResources(t *testing.T) {
    plugin := &MyTransformPlugin{Log: logrus.New()}
    request := transform.PluginRequest{
        Unstructured: unstructured.Unstructured{
            Object: map[string]interface{}{
                "apiVersion": "v1",
                "kind":       "ConfigMap",
                "metadata":   map[string]interface{}{"name": "test", "namespace": "default"},
            },
        },
    }
    resp, err := plugin.Run(request)
    assert.NoError(t, err)
    assert.False(t, resp.IsWhiteOut)
    assert.Nil(t, resp.Patches)
    assert.Nil(t, resp.NewResources)
}
```

### Test with extras (optional flags)

```go
func TestWithFlags(t *testing.T) {
    plugin := &MyTransformPlugin{Log: logrus.New()}
    request := transform.PluginRequest{
        Unstructured: buildTestResource(),
        Extras: map[string]string{
            "registry-replacement": "old.io=new.io,docker.io/foo=quay.io/bar",
        },
    }
    resp, err := plugin.Run(request)
    require.NoError(t, err)
    // assert expected patches or whiteout
}
```

### Test resource conversion (Pattern B)

```go
func TestConversion(t *testing.T) {
    plugin := &ConverterPlugin{Log: logrus.New()}
    input := unstructured.Unstructured{
        Object: map[string]interface{}{
            "apiVersion": "source.api.io/v1",
            "kind":       "SourceKind",
            "metadata":   map[string]interface{}{"name": "test", "namespace": "ns"},
            "spec": map[string]interface{}{
                // realistic spec fields
            },
        },
    }
    resp, err := plugin.Run(transform.PluginRequest{Unstructured: input})
    require.NoError(t, err)
    assert.True(t, resp.IsWhiteOut)
    require.Len(t, resp.NewResources, 1)

    newRes := resp.NewResources[0]
    assert.Equal(t, "TargetKind", newRes.GetKind())
    assert.Equal(t, "target.api.io/v1beta1", newRes.GetAPIVersion())

    spec, _, _ := unstructured.NestedMap(newRes.Object, "spec")
    assert.Equal(t, "expected-value", spec["fieldName"])
}
```

### Test optional field parsing

```go
func TestParseOptionalFields(t *testing.T) {
    extras := map[string]string{
        "registry-replacement": "old.io=new.io,docker.io=quay.io",
        "my-list-flag":         "a,b,c",
    }
    mapVal := transform.ParseOptionalFieldMapVal(extras["registry-replacement"])
    assert.Equal(t, "new.io", mapVal["old.io"])
    assert.Equal(t, "quay.io", mapVal["docker.io"])

    sliceVal := transform.ParseOptionalFieldSliceVal(extras["my-list-flag"])
    assert.Equal(t, []string{"a", "b", "c"}, sliceVal)
}
```

## Building, testing, and releasing

### Build

```bash
go build -o crane-plugin-YOURNAME .
```

### Run tests

```bash
go test ./...
```

### Manual protocol test

```bash
# Metadata request — should return plugin metadata JSON
echo '{}' | ./crane-plugin-YOURNAME

# Transform request — pass a Kubernetes resource
echo '{"apiVersion":"v1","kind":"ConfigMap","metadata":{"name":"test","namespace":"default"}}' | ./crane-plugin-YOURNAME

# Transform request with extras (flags)
echo '{"apiVersion":"v1","kind":"ConfigMap","metadata":{"name":"test"},"extras":{"my-flag":"value"}}' | ./crane-plugin-YOURNAME
```

### Cross-compile for release

```bash
GOOS=linux   GOARCH=amd64 go build -o amd64-linux-pluginname-v0.1.0 .
GOOS=darwin  GOARCH=amd64 go build -o amd64-darwin-pluginname-v0.1.0 .
GOOS=darwin  GOARCH=arm64 go build -o arm64-darwin-pluginname-v0.1.0 .
```

Upload binaries as GitHub release assets, then register in the [crane-plugins](https://github.com/migtools/crane-plugins) index (see AGENTS.md for manifest format).

## Common pitfalls

1. **Never write to stdout except JSON responses** — all logging must go to stderr. Use `logrus.New()` which defaults to stderr.
2. **Binary name must start with `crane-plugin-`** — crane discovers plugins by this prefix in the plugin directory.
3. **Handle unknown resource types gracefully** — return an empty `PluginResponse{}` for resources your plugin doesn't handle. Don't error on unknown kinds.
4. **Don't assume field existence** — Kubernetes resources may have optional fields. Always check for nil before accessing nested structs. For example, `BuildConfig.Spec.Source` may be nil.
5. **`Extras` map may be nil or have missing keys** — always check `len(extras[key]) > 0` before parsing.
6. **JSON patches use RFC 6902** — paths use `/` separators, array indices are numeric. Use `github.com/evanphx/json-patch`.
7. **One resource per invocation** — crane calls your plugin once per resource in the export. Your plugin sees one resource at a time, not the whole namespace.
8. **`NewResources` requires unreleased crane-lib** — if you need to generate new resources (not just patch), add a `replace` directive in go.mod.
9. **Set GVK on new resources** — when using `runtime.DefaultUnstructuredConverter.ToUnstructured()`, the GVK may not be set automatically. Set `apiVersion` and `kind` explicitly on the typed struct or the resulting unstructured object.

## Related repositories

| Repository | Purpose |
|-----------|---------|
| [crane](https://github.com/konveyor/crane) | Main crane CLI tool |
| [crane-lib](https://github.com/konveyor/crane-lib) | Plugin interface, CLI harness, types |
| [crane-plugins](https://github.com/migtools/crane-plugins) | Plugin index (this repo — YAML manifests for discovery) |
| [crane-plugin-openshift](https://github.com/migtools/crane-plugin-openshift) | Reference: simple whiteout/patch pattern |
| [crane-plugin-buildconfig-to-shipwright](https://github.com/migtools/crane-plugin-buildconfig-to-shipwright) | Reference: complex resource conversion with NewResources |
