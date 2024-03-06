<div align="center">
  
# Repo File Sync Action

</div>

## 👋 Introduction

With this you can sync files, like workflow `.yml` files, configuration files or whole directories between repositories or branches. It works by running a GitHub Action in your main repository everytime you push something to that repo. The action will use a `sync.yml` config file to figure out which files it should sync where. If it finds a file which is out of sync it will open a pull request in the target repository with the changes.

## 🚀 Features

- Keep GitHub Actions workflow files in sync across all your repositories
- Sync any file or a whole directory to as many repositories as you want
- Easy configuration for any use case
- Create a pull request in the target repo so you have the last say on what gets merged
- Automatically label pull requests to integrate with other actions like [automerge-action](https://github.com/pascalgn/automerge-action)
- Assign users to the pull request
- Render [Jinja](https://jinja.palletsprojects.com/)-style templates as use variables thanks to [Nunjucks](https://mozilla.github.io/nunjucks/)

## 📚 Usage


### Workflow

Create a `.yml` file in your `.github/workflows` folder (you can find more info about the structure in the [GitHub Docs](https://docs.github.com/en/free-pro-team@latest/actions/reference/workflow-syntax-for-github-actions)):

**.github/workflows/sync.yml**

```yml
name: Sync Files
on:
  push:
    branches:
      - master
  workflow_dispatch:
jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@master
      - name: Run GitHub File Sync
        uses: BetaHuhn/repo-file-sync-action@v1
        with:
          GH_PAT: ${{ secrets.GH_PAT }}
```

#### Token

In order for the Action to access your repositories you have to specify a [Personal Access token](https://docs.github.com/en/free-pro-team@latest/github/authenticating-to-github/creating-a-personal-access-token) as the value for `GH_PAT` (`GITHUB_TOKEN` will **not** work). The PAT needs the full repo scope ([#31](https://github.com/BetaHuhn/repo-file-sync-action/discussions/31#discussioncomment-674804)).

It is recommended to set the token as a
[Repository Secret](https://docs.github.com/en/free-pro-team@latest/actions/reference/encrypted-secrets#creating-encrypted-secrets-for-a-repository).

Alternatively, you can provide the token of a GitHub App Installation via the `GH_INSTALLATION_TOKEN` input. You can obtain such token for example via [this](https://github.com/marketplace/actions/github-app-token) action. Tokens from apps have the advantage that they provide more granular access control.

The app needs to be configured for each repo you want to sync to, and have the `Contents` read & write and `Metadata` read-only permission. If you want to use PRs (default setting) you additionally need `Pull requests` read & write access, and to sync workflow files you need `Workflows` read & write access.

If using an installation token you are required to provide the `GIT_EMAIL` and `GIT_USERNAME` input.

### Sync configuration

The last step is to create a `.yml` file in the `.github` folder of your repository and specify what file(s) to sync to which repositories:

**.github/sync.yml**

```yml
user/repository:
  - .github/workflows/test.yml
  - .github/workflows/lint.yml

user/repository2:
  - source: workflows/stale.yml
    dest: .github/workflows/stale.yml
```

More info on how to specify what files to sync where [below](#%EF%B8%8F-sync-configuration).


With the `v1` tag you will always get the latest non-breaking version which will include potential bug fixes in the future. If you use a specific version, make sure to regularly check if a new version is available, or enable Dependabot.

## ⚙️ Action Inputs

| Key | Value | Required | Default |
| ------------- | ------------- | ------------- | ------------- |
| `GH_PAT` | Your [Personal Access token](https://docs.github.com/en/free-pro-team@latest/github/authenticating-to-github/creating-a-personal-access-token) | **`GH_PAT` or `GH_INSTALLATION_TOKEN` required** | N/A |
| `GH_INSTALLATION_TOKEN` | Token from a GitHub App installation | **`GH_PAT` or `GH_INSTALLATION_TOKEN` required** | N/A |
| `CONFIG_PATH` | Path to the sync configuration file | **No** | .github/sync.yml |
| `PR_LABELS` | Labels which will be added to the pull request. Set to false to turn off | **No** | sync |
| `ASSIGNEES` | Users to assign to the pull request | **No** | N/A |
| `REVIEWERS` | Users to request a review of the pull request from | **No** | N/A |
| `TEAM_REVIEWERS` | Teams to request a review of the pull request from | **No** | N/A |
| `COMMIT_PREFIX` | Prefix for commit message and pull request title | **No** | 🔄 |
| `COMMIT_BODY` | Commit message body. Will be appended to commit message, separated by two line returns. | **No** | '' |
| `PR_BODY` | Additional content to add in the PR description. | **No** | '' |
| `ORIGINAL_MESSAGE` | Use original commit message instead. Only works if the file(s) were changed and the action was triggered by pushing a single commit. | **No** | false |
| `COMMIT_AS_PR_TITLE` | Use first line of the commit message as PR title. Only works if `ORIGINAL_MESSAGE` is `true` and working. | **No** | false |
| `COMMIT_EACH_FILE` | Commit each file seperately | **No** | true |
| `GIT_EMAIL` | The e-mail address used to commit the synced files | **Only when using installation token** | the email of the PAT used |
| `GIT_USERNAME` | The username used to commit the synced files | **Only when using installation token** | the username of the PAT used |
| `OVERWRITE_EXISTING_PR` | Overwrite any existing Sync PR with the new changes | **No** | true |
| `BRANCH_PREFIX` | Specify a different prefix for the new branch in the target repo | **No** | repo-sync/SOURCE_REPO_NAME |
| `TMP_DIR` | The working directory where all git operations will be done | **No** | tmp-${ Date.now().toString() } |
| `DRY_RUN` | Run everything except that nothing will be pushed | **No** | false |
| `SKIP_CLEANUP` | Skips removing the temporary directory. Useful for debugging | **No** | false |
| `SKIP_PR` | Skips creating a Pull Request and pushes directly to the default branch | **No** | false |
| `FORK` | A Github account username. Changes will be pushed to a fork of target repos on this account. | **No** | false |

### Outputs

The action sets the `pull_request_urls` output to the URLs of any created Pull Requests. It will be an array of URLs to each PR, e.g. `'["https://github.com/username/repository/pull/number", "..."]'`.

## 🛠️ Sync Configuration

The top-level key should be used to specify the target repository in the format `username`/`repository-name`@`branch`, after that you can list all the files you want to sync to that individual repository:

```yml
user/repo:
  - path/to/file.txt
user/repo2@develop:
  - path/to/file2.txt
```

There are multiple ways to specify which files to sync to each individual repository.

### List individual file(s)

The easiest way to sync files is the list them on a new line for each repository:

```yml
user/repo:
  - .github/workflows/build.yml
  - LICENSE
  - .gitignore
```

### Different destination path/filename(s)

Using the `dest` option you can specify a destination path in the target repo and/or change the filename for each source file:

```yml
user/repo:
  - source: workflows/build.yml
    dest: .github/workflows/build.yml
  - source: LICENSE.md
    dest: LICENSE
```

### Sync entire directories

You can also specify entire directories to sync:

```yml
user/repo:
  - source: workflows/
    dest: .github/workflows/
```

### Exclude certain files when syncing directories

Using the `exclude` key you can specify files you want to exclude when syncing entire directories (#26).

```yml
user/repo:
  - source: workflows/
    dest: .github/workflows/
    exclude: |
      node.yml
      lint.yml
```

> **Note:** the exclude file path is relative to the source path

### Don't replace existing file(s)

By default if a file already exists in the target repository, it will be replaced. You can change this behaviour by setting the `replace` option to `false`:

```yml
user/repo:
  - source: .github/workflows/lint.yml
    replace: false
```

### Using templates

You can render templates before syncing by using the [Jinja](https://jinja.palletsprojects.com/)-style template syntax. It will be compiled using [Nunjucks](https://mozilla.github.io/nunjucks/) and the output written to the specific file(s) or folder(s).

Nunjucks supports variables and blocks among other things. To enable, set the `template` field to a context dictionary, or in case of no variables, `true`:

```yml
user/repo:
  - source: src/README.md
    template:
      user:
        name: 'Maxi'
        handle: '@BetaHuhn'
```

In the source file you can then use these variables like this:

```yml
# README.md

Created by {{ user.name }} ({{ user.handle }})
```

Result:

```yml
# README.md

Created by Maxi (@BetaHuhn)
```

You can also use `extends` with a relative path to inherit other templates. Take a look at Nunjucks [template syntax](https://mozilla.github.io/nunjucks/templating.html) for more info.

```yml
user/repo:
  - source: .github/workflows/child.yml
    template: true
```

```yml
# child.yml
{% extends './parent.yml' %}

{% block some_block %}
This is some content
{% endblock %}
```

### Delete orphaned files

With the `deleteOrphaned` option you can choose to delete files in the target repository if they are deleted in the source repository. The option defaults to `false` and only works when [syncing entire directories](#sync-entire-directories):

```yml
user/repo:
  - source: workflows/
    dest: .github/workflows/
    deleteOrphaned: true
```

It only takes effect on that specific directory.

### Sync the same files to multiple repositories

Instead of repeating yourself listing the same files for multiple repositories, you can create a group:

```yml
group:
  repos: |
    user/repo
    user/repo1
  files: 
    - source: workflows/build.yml
      dest: .github/workflows/build.yml
    - source: LICENSE.md
      dest: LICENSE
```

You can create multiple groups like this:

```yml
group:
  # first group
  - files:
      - source: workflows/build.yml
        dest: .github/workflows/build.yml
      - source: LICENSE.md
        dest: LICENSE
    repos: |
      user/repo1
      user/repo2

  # second group
  - files: 
      - source: configs/dependabot.yml
        dest: .github/dependabot.yml
    repos: |
      user/repo3
      user/repo4
```

### Syncing branches

You can also sync different branches from the same or different repositories (#51). For example, a repository named `foo/bar` with branch `main`, and `sync.yml` contents:

```yml
group:
  repos: |
    foo/bar@de
    foo/bar@es
    foo/bar@fr
  files:
    - source: .github/workflows/
      dest: .github/workflows/
```

Here all files in `.github/workflows/` will be synced from the `main` branch to the branches `de`/`es`/`fr`.

## 📖 Examples

Here are a few examples to help you get started!

### Basic Example

**.github/sync.yml**

```yml
user/repository:
  - LICENSE
  - .gitignore
```

### Sync all workflow files

This example will keep all your `.github/workflows` files in sync across multiple repositories:

**.github/sync.yml**

```yml
group:
  repos: |
    user/repo1
    user/repo2
  files:
    - source: .github/workflows/
      dest: .github/workflows/
```
