# Kustomize + Tekton

How to hack your [`Tekton`](https://tekton.dev) tasks with [`kustomize`](https://kustomize.io) super powers to reuse parts and customize new tasks

Let's say we want two different tasks `golang-test` and `golangci-lint` need to add a `reviewdog-report` step as seen below


![tasks](images/tasks-diagram.drawio.png)


The most  obvious way would be copying and pasting the steps, but for more complex scenarios, where there are `n` tasks, this becomes error prone. Using a template engine like [`helm`](https://helm.sh) could help, but learning another templating engine, plus having to change the contents of said tasks also becomes a burden. Instead, [`kustomize`](https://kustomize.io) has a set of tools to make this job easier, while enjoying reutilizing tasks from the [`tektoncd/catalog`](https://github.com/tektoncd/catalog).

## Folder structure


```
├── overlays
└── tasks
```

 - *tasks* will host your tasks files
 - *overlays* will host your patches, like adding a shared step

## Tasks

Prepare your tasks, for the example we will use [`golang-test`](https://github.com/tektoncd/catalog/tree/main/task/golang-test/0.2) and [`golangci-lint`](https://github.com/tektoncd/catalog/tree/main/task/golangci-lint/0.2), adjust accordingly.

## Snippet

With the tasks ready, it is time to prepare a snippet, which should consist of all the necessary additions, like `params`, `workspaces`, `results`, `steps`, etc:

```yaml
spec:
  params:
  # since we are splitting the steps, the previous step needs to save the output to a file
  - name: report-file
    default: reportfile
    description: Report file with errors
  # format of the report file
  - name: format
    default: golint
    description: Format of error input from the task
  # reviewdog supports a number of reporter types
  - name: reporter
    default: local
    description: Reporter type for reviewdog https://github.com/reviewdog/reviewdog#reporters
  # reviewdog needs a diff for precise pull request comments
  - name: diff
    default: git diff FETCH_HEAD
    description: Diff command https://github.com/reviewdog/reviewdog#reporters

  workspaces:
  - name: token
    description: |
      Workspace which contains a token file for Github Pull Request comments. Must have a token file with the Github API access token

  steps:
  - name: reviewdog-report
    image: golangci/golangci-lint:v1.31-alpine
    # both have the same workspace name
    workingDir: $(workspaces.source.path)
    script: |
      #!/bin/sh

      set -ue
      wget -O - -q https://raw.githubusercontent.com/reviewdog/reviewdog/master/install.sh | sh -s -- -b $(go env GOPATH)/bin
      export REVIEWDOG_GITHUB_API_TOKEN=$(cat $(workspaces.token.path)/token)
      cat $(params.reportfile) | reviewdog -f=$(params.format) -diff="$(params.diff)"


```


## Patch

With above snippet, it is time to create our patch. I've tried using the snippet directly using `patch` but was not successful, so I decided to create a [`JSON6902 patch`](https://kubectl.docs.kubernetes.io/references/kustomize/glossary/#patchjson6902):

```yaml
# parameters
- op: add
  path: /spec/params/-
  value:
    name: report-file
    default: reportfile
    description: Report file with errors
- op: add
  path: /spec/params/-
  value:
    name: format
    default: golint
    description: Format of error input from the task
- op: add
  path: /spec/params/-
  value:
    name: reporter
    default: local
    description: Reporter type for reviewdog https://github.com/reviewdog/reviewdog#reporters
- op: add
  path: /spec/params/-
  value:
    name: diff
    default: git diff FETCH_HEAD
    description: Diff command https://github.com/reviewdog/reviewdog#reporters

# workspaces
- op: add
  path: /spec/workspaces/-
  value:
    name: token
    description: |
      Workspace which contains a token file for Github Pull Request comments. Must have a token file with the Github API access token

# steps
- op: add
  path: /spec/steps/-
  value:
    name: reviewdog-report
    image: golangci/golangci-lint:v1.31-alpine
    # both have the same workspace name
    workingDir: $(workspaces.source.path)
    script: |
      #!/bin/sh

      set -ue
      wget -O - -q https://raw.githubusercontent.com/reviewdog/reviewdog/master/install.sh | sh -s -- -b $(go env GOPATH)/bin
      export REVIEWDOG_GITHUB_API_TOKEN=$(cat $(workspaces.token.path)/token)
      cat $(params.reportfile) | reviewdog -f=$(params.format) -diff="$(params.diff)"

```

save as `reviewdog-step-patch.yaml` and create a `kustomization.yaml` with the following content:


```yaml
bases:
- ../tasks

patches:
- path: ./reviewdog-step-patch.yaml
  target:
    kind: Task
```


## More patching

Make sure all `kustomization.yaml` files are setup correctly, and try running `kustomize build overlays`. You should see the `params`, `workspaces` and `params` added.

*But wait!*, we still need to connect the dots. Without modifying the imported tasks this would still not work.

For the `golangci-lint` it is necessary to save the result to a file as given in the parameter `$(params.report-file)`, and change default format for the `$(params.format)` parameter to `golangci-lint`:

### golangci-lint

```yaml
- op: replace
  path: /spec/params/11/default
  value: golangci-lint

- op: replace
  path: /spec/steps/0/script
  value: |
    golangci-lint run $(params.flags) > $(params.report-file)
```

Save the file as `overlays/golangci-lint-patch.yaml` and add it to the `overlays/kustomization.yaml`

```yaml
[...]
- path: ./golangci-lint-patch.yaml
  target:
    kind: Task
    name: golangci-lint
```

### golang-test

Here is the change for `golang-test`, and actually making it do a `golint`:

```yaml
- op: replace
  path: /spec/steps/0/script
  value: |
    if [ ! -e $GOPATH/src/$(params.package)/go.mod ];then
         SRC_PATH="$GOPATH/src/$(params.package)"
         mkdir -p $SRC_PATH
         cp -R "$(workspaces.source.path)/$(params.context)"/* $SRC_PATH
         cd $SRC_PATH
      fi
      golint $(params.packages) > $(params.report-file)
```

Add the entry to `overlays/kustomization.yaml`:

```yaml
- path: ./golint-patch.yaml
  target:
    kind: Task
    name: golang-test
```

## Suffix

If you want to preserve the original tasks and only add new tasks, just change the `overlays/kustomization.yaml` adding a suffix to the files:


```yaml
nameSuffix: -review
```

## Testing

Apply your tasks into your `kubernetes cluster` using `kubectl apply -k overlays` or `kustomize build overlays | kubectl apply -f -`



The resulting file tree should be something like:

```
├── overlays
│   ├── golangci-lint-patch.yaml
│   ├── golint-patch.yaml
│   ├── kustomization.yaml
│   └── reviewdog-step-patch.yaml
└── tasks
    ├── golang-test.yaml
    ├── golangci-lint.yaml
    └── kustomization.yaml
```

Enjoy your new `golang-test-review` and `golangci-lint-review`