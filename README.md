# Renovate discussion number TBD

Replacing Tekton bundle (docker image) via `replacementNameTemplate` results
in incorrect `newDigest`, `replacementName` works as expected.

## Details

[`.tekton/pipeline.yaml`](.tekton/pipeline.yaml) has two Tekton bundles (docker images,
except they only contain YAML data) that are supposed to get migrated from
`quay.io/redhat-appstudio-tekton-catalog` to `quay.io/konflux-ci/tekton-catalog`:

* [`quay.io/redhat-appstudio-tekton-catalog/task-init`][rhinit] -> [`quay.io/konflux-ci/tekton-catalog/task-init`][kfluxinit]
* [`quay.io/redhat-appstudio-tekton-catalog/task-git-clone`][rhclone] -> [`quay.io/konflux-ci/tekton-catalog/task-git-clone`][kfluxclone]

[`renovate.json`](renovate.json) has a general `replacementNameTemplate` rule that
would match both of them:

```json
    {
      "matchPackageNames": [
        "quay.io/redhat-appstudio-tekton-catalog/*"
      ],
      "replacementNameTemplate": "{{{replace 'redhat-appstudio-tekton-catalog' 'konflux-ci/tekton-catalog' packageName}}}"
    }
```

And one specifically for `task-git-clone` that takes precedence and uses a static
`replacementName` instead:

```json
    {
      "matchPackageNames": [
        "quay.io/redhat-appstudio-tekton-catalog/task-git-clone"
      ],
      "replacementName": "quay.io/konflux-ci/tekton-catalog/task-git-clone"
    }
```

## Current behavior

`task-init` gets an incorrect digest after replacement - Renovate uses the digest
of the old bundle from `redhat-appstudio-tekton-catalog`, doesn't resolve the digest
of the new bundle.

```json
{
  "depName": "quay.io/redhat-appstudio-tekton-catalog/task-init",
  "packageName": "quay.io/redhat-appstudio-tekton-catalog/task-init",
  "currentValue": "0.2",
  "currentDigest": "sha256:ebfb603b73c2e1500fa1d2c757a585bc2da5043afe0798abdf61466e26fd2b0c",
  "replaceString": "quay.io/redhat-appstudio-tekton-catalog/task-init:0.2@sha256:ebfb603b73c2e1500fa1d2c757a585bc2da5043afe0798abdf61466e26fd2b0c",
  "autoReplaceStringTemplate": "{{depName}}{{#if newValue}}:{{newValue}}{{/if}}{{#if newDigest}}@{{newDigest}}{{/if}}",
  "datasource": "docker",
  "depType": "tekton-bundle",
  "updates": [
    {
      "updateType": "replacement",
      "newName": "quay.io/konflux-ci/tekton-catalog/task-init",
      "newValue": "0.2",
      "newDigest": "sha256:ebfb603b73c2e1500fa1d2c757a585bc2da5043afe0798abdf61466e26fd2b0c",
      "branchName": "renovate/quay.io-redhat-appstudio-tekton-catalog-task-init-replacement"
    }
  ],
  "versioning": "docker",
  "warnings": [],
  "registryUrl": "https://quay.io",
  "lookupName": "redhat-appstudio-tekton-catalog/task-init",
  "currentVersion": "0.2",
  "fixedVersion": "0.2"
}
```

```bash
$ skopeo inspect --raw docker://quay.io/konflux-ci/tekton-catalog/task-init@sha256:ebfb603b73c2e1500fa1d2c757a585bc2da5043afe0798abdf61466e26fd2b0c

FATA[0001] ...: manifest unknown
```

`task-git-clone`, which is replaced via the static `replacementName`, gets a correct
digest.

```json
{
  "depName": "quay.io/redhat-appstudio-tekton-catalog/task-git-clone",
  "packageName": "quay.io/redhat-appstudio-tekton-catalog/task-git-clone",
  "currentValue": "0.1",
  "currentDigest": "sha256:b99d377c3e28fad51009849f6ba3a1bc47d1dc4c46f470ea12ed7b1b444599d7",
  "replaceString": "quay.io/redhat-appstudio-tekton-catalog/task-git-clone:0.1@sha256:b99d377c3e28fad51009849f6ba3a1bc47d1dc4c46f470ea12ed7b1b444599d7",
  "autoReplaceStringTemplate": "{{depName}}{{#if newValue}}:{{newValue}}{{/if}}{{#if newDigest}}@{{newDigest}}{{/if}}",
  "datasource": "docker",
  "depType": "tekton-bundle",
  "updates": [
    {
      "updateType": "replacement",
      "newName": "quay.io/konflux-ci/tekton-catalog/task-git-clone",
      "newValue": "0.1",
      "newDigest": "sha256:7939000e2f92fc8b5d2c4ee4ba9000433c5aa7700d2915a1d4763853d5fd1fd4",
      "branchName": "renovate/quay.io-redhat-appstudio-tekton-catalog-task-git-clone-replacement"
    }
  ],
  "versioning": "docker",
  "warnings": [],
  "registryUrl": "https://quay.io",
  "lookupName": "redhat-appstudio-tekton-catalog/task-git-clone",
  "currentVersion": "0.1",
  "fixedVersion": "0.1"
}
```

```bash
$ skopeo inspect --raw docker://quay.io/konflux-ci/tekton-catalog/task-git-clone@sha256:7939000e2f92fc8b5d2c4ee4ba9000433c5aa7700d2915a1d4763853d5fd1fd4

# works
```

The renovate job reports an error and opens a PR only for the git-clone task. I suspect
the reason is what I describe above, though I cannot tell from the job logs (attached:
[`renovate.log`](renovate.log)). The relevant part seems to be this one, but it doesn't
mention the specific problem.

```json
{
  "branchName": "renovate/quay.io-redhat-appstudio-tekton-catalog-task-init-replacement",
  "prNo": null,
  "prTitle": "Replace quay.io/redhat-appstudio-tekton-catalog/task-init Docker tag with quay.io/konflux-ci/tekton-catalog/task-init 0.2",
  "result": "error",
  "upgrades": [
    {
      "datasource": "docker",
      "depName": "quay.io/redhat-appstudio-tekton-catalog/task-init",
      "displayPending": "",
      "fixedVersion": "0.2",
      "currentVersion": "0.2",
      "currentValue": "0.2",
      "currentDigest": "sha256:ebfb603b73c2e1500fa1d2c757a585bc2da5043afe0798abdf61466e26fd2b0c",
      "newValue": "0.2",
      "newDigest": "sha256:ebfb603b73c2e1500fa1d2c757a585bc2da5043afe0798abdf61466e26fd2b0c",
      "packageFile": ".tekton/pipeline.yaml",
      "updateType": "replacement",
      "packageName": "quay.io/redhat-appstudio-tekton-catalog/task-init"
    }
  ]
}
```

## Expected behavior

Expected `replacementName` and `replacementNameTemplate` to both resolve the new
digest correctly.

## Link to the Renovate issue or Discussion

TBD

[rhinit]: https://quay.io/redhat-appstudio-tekton-catalog/task-init
[kfluxinit]: https://quay.io/konflux-ci/tekton-catalog/task-init
[rhclone]: https://quay.io/redhat-appstudio-tekton-catalog/task-git-clone
[kfluxclone]: https://quay.io/konflux-ci/tekton-catalog/task-git-clone
