# bundled-pm-change-node-26-9-0

## Probe metadata

| Field            | Value                                             |
|------------------|---------------------------------------------------|
| Pattern          | `version-lang-bundled-pm-change-node-26.9.0`      |
| Category         | `bundled_pm_change`                               |
| Node.js release  | 26.9.0 (Current), 2026-09-16, @aduh95             |
| pm               | npm                                               |
| Schema version   | 1.0                                               |
| Generated        | 2026-09-16                                        |

## Feature exercised

Node.js 26.9.0 introduces SEMVER-MINOR changes to `crypto`,
`ffi`, and `vfs` core modules that alter how the runtime
resolves and loads native addons. This probe pins
`engines.node >= 26.9.0` and includes four direct dependencies
whose postinstall scripts or platform-specific binary bundles
exercise those changed resolution paths:

- `argon2` — uses `node-gyp` and `node-addon-api`; sensitive
  to changes in crypto ABI.
- `sodium-native` — low-level libsodium binding via `node-gyp-build`;
  sensitive to ffi/ABI changes.
- `sharp` — ships platform-specific prebuilt binaries via optional
  `@img/sharp-*` packages; exercises the vfs binary-loading path.
- `@mapbox/node-pre-gyp` — native binary downloader; exercises the
  module-loading path changed in Node 26.9.0.

## Why this probe matters for Mend SCA

Mend's Unified Agent invokes `npm ls` and (when `runPreStep=true`)
`npm install` as child processes that inherit the ambient Node runtime.
If native-addon postinstall fails under the new ABI, `npm ls` may
exit non-zero or produce a truncated tree. This probe verifies that:

1. Mend falls back to lockfile-based detection when `npm ls` fails.
2. All packages recorded in `package-lock.json` appear in the
   reported tree — including packages whose native layer failed
   to load.
3. Optional platform-specific `@img/sharp-*` variants carry
   `optional: true` and the correct platform markers.
4. `sodium-native` and `argon2` are NOT dropped even if their
   gyp builds fail under the new Node crypto surface.

## Expected dependency tree

Direct deps (all `group: main`):

- `@mapbox/node-pre-gyp@1.0.11`
- `argon2@0.31.2`
- `sharp@0.33.4`
- `sodium-native@4.0.9`

Key transitive chains:

```
@mapbox/node-pre-gyp
  detect-libc@2.0.3
  https-proxy-agent@7.0.5
    agent-base@7.1.1
      debug@4.3.6
        ms@2.1.2
  nopt@7.2.1
    abbrev@2.0.0
  npmlog@7.0.1
    are-we-there-yet@4.0.2
      delegates@1.0.0
      readable-stream@3.6.2
        inherits@2.0.4
        string_decoder@1.3.0
          safe-buffer@5.2.1
    console-control-strings@1.1.0
    gauge@5.0.2
      aproba@2.0.0
      color-support@1.1.3
      has-unicode@2.0.1
      signal-exit@4.1.0
      string-width@4.2.3
        emoji-regex@8.0.0
        is-fullwidth-code-point@3.0.0
        strip-ansi@6.0.1
          ansi-regex@5.0.1
      wide-align@1.1.5
    set-blocking@2.0.0
  rimraf@3.0.2
    (nested glob@7.2.3 → fs.realpath, inflight, once, path-is-absolute,
     wrappy, concat-map, balanced-match via rimraf-local minimatch@3.1.2
     and brace-expansion@1.1.11)
  semver@7.6.3
  tar@6.2.1
    chownr@2.0.0
    minipass@5.0.0
    minizlib@2.1.2
      yallist@4.0.0
    mkdirp@3.0.1
      minimatch@9.0.5
        brace-expansion@2.0.1
          balanced-match@1.0.2

argon2
  node-addon-api@7.1.0

sharp
  color@4.2.3
    color-convert@2.0.1
      color-name@1.1.4
    color-string@1.9.1
      is-arrayish@0.3.2
      simple-swizzle@0.2.2
  detect-libc@2.0.3   (shared/deduped with @mapbox entry)
  semver@7.6.3        (shared/deduped)
  @img/sharp-darwin-arm64@0.33.4   optional
  @img/sharp-darwin-x64@0.33.4    optional
  @img/sharp-linux-arm@0.33.4     optional
  @img/sharp-linux-arm64@0.33.4   optional
  @img/sharp-linux-x64@0.33.4     optional
  @img/sharp-win32-x64@0.33.4     optional

sodium-native
  node-gyp-build@4.8.1
```

## Mend failure modes this probe exercises

| Failure mode                            | Expected Mend behavior                     |
|-----------------------------------------|--------------------------------------------|
| `npm ls` exits non-zero (gyp failure)   | Fall back to lockfile; full tree reported  |
| Native addon load silently missing      | Package still in tree (lockfile source)    |
| `optional: true` lost on `@img/*`       | All `@img/sharp-*` must carry `optional`   |
| `sodium-native` dropped on crypto fail  | Must appear as `group: main`               |

## Mend config

Bucket A — no dynamic version detection for npm/Node from manifest.

`.whitesource` pins:
- `node: "26.9.0"` — exact Node.js version under test.
- `npm: "10.9.2"` — npm bundled with Node 26.9.0 LTS toolchain.

`configMode: "AUTO"` — no `whitesource.config` ships with this probe.

The version pin for `node` deliberately matches the release under test
(26.9.0) rather than a generic LTS, so Mend's `install-tool` provisions
the same ABI environment that defines the probe's expected tree.
