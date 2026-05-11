# scoop-bucket

[Scoop](https://scoop.sh) bucket for [Lenny Labs](https://github.com/lennylabs) projects. One manifest per project; each project's release workflow keeps its own manifest current.

## Use

```powershell
scoop bucket add lennylabs https://github.com/lennylabs/scoop-bucket
scoop install <package>
```

`scoop update` picks up new versions as each project's release workflow updates the corresponding manifest. Even between manifest updates, the `checkver` / `autoupdate` blocks in each manifest let Scoop notice new GitHub Releases directly.

## Available manifests

| Manifest | Project | Description |
|:--|:--|:--|
| `podium` | [lennylabs/podium](https://github.com/lennylabs/podium) | Catalog and registry for reusable AI agent artifacts |

Add a row to this table when you add a new manifest.

## Layout

```
bucket/
├── podium.json
└── <future>.json     ← drop a new manifest here for each new project
```

Each manifest is independent. Adding `bucket/foo.json` doesn't touch any other manifest in the bucket.

## How manifests stay current

Each project's release workflow updates **only its own manifest** in this repo. The pattern, by convention:

1. After a tag push, the project's release workflow downloads its just-released Windows binary from its own GitHub Release.
2. Computes the SHA256.
3. Clones this repo using a fine-grained PAT (`TAP_BUCKET_TOKEN` or similar) that has write access scoped to this single repo.
4. Patches `bucket/<project>.json` (jq against the version + url + hash fields).
5. Commits and pushes.

The reference implementation lives in [lennylabs/podium's `.github/workflows/release.yml`](https://github.com/lennylabs/podium/blob/main/.github/workflows/release.yml) — copy and adapt the `publish-tap-bucket` job for any new project that wants to ship through this bucket.

## Adding a new project

1. Drop `bucket/<project>.json` here as a one-off commit (initially with placeholder SHA256).
2. Add a row to the table above.
3. In the new project's repo, add the `TAP_BUCKET_TOKEN` secret (write access to this repo).
4. Adapt the `publish-tap-bucket` job from podium's release workflow.

## Updating a manifest by hand

```powershell
$VERSION = "X.Y.Z"
$URL = "https://github.com/lennylabs/podium/releases/download/v$VERSION/podium-windows-amd64.exe"
Invoke-WebRequest $URL -OutFile podium.exe
Get-FileHash podium.exe -Algorithm SHA256
# Edit bucket/podium.json: update version + architecture.64bit.url + .hash
```

Or use `jq` from a Linux/macOS machine:

```sh
PROJECT=podium
VERSION=X.Y.Z
HASH=$(curl -sSL "https://github.com/lennylabs/${PROJECT}/releases/download/v${VERSION}/${PROJECT}-windows-amd64.exe" | sha256sum | awk '{print $1}')
jq --arg v "${VERSION}" --arg h "${HASH}" --arg p "${PROJECT}" '
  .version = $v
  | .architecture."64bit".url = "https://github.com/lennylabs/\($p)/releases/download/v\($v)/\($p)-windows-amd64.exe#/\($p).exe"
  | .architecture."64bit".hash = "sha256:" + $h
' bucket/${PROJECT}.json > tmp.json && mv tmp.json bucket/${PROJECT}.json
```

## License

[MIT](LICENSE).
