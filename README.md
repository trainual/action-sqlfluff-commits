# action-sqlfluff

<!-- TODO: replace reviewdog/yu-iskw/action-sqlfluff with your repo name -->
[![Test](https://github.com/yu-iskw/action-sqlfluff/workflows/Test/badge.svg)](https://github.com/yu-iskw/action-sqlfluff/actions?query=workflow%3ATest)
[![reviewdog](https://github.com/yu-iskw/action-sqlfluff/workflows/reviewdog/badge.svg)](https://github.com/yu-iskw/action-sqlfluff/actions?query=workflow%3Areviewdog)
[![depup](https://github.com/yu-iskw/action-sqlfluff/workflows/depup/badge.svg)](https://github.com/yu-iskw/action-sqlfluff/actions?query=workflow%3Adepup)
[![release](https://github.com/yu-iskw/action-sqlfluff/workflows/release/badge.svg)](https://github.com/yu-iskw/action-sqlfluff/actions?query=workflow%3Arelease)
[![GitHub release (latest SemVer)](https://img.shields.io/github/v/release/yu-iskw/action-sqlfluff?logo=github&sort=semver)](https://github.com/yu-iskw/action-sqlfluff/releases)
[![action-bumpr supported](https://img.shields.io/badge/bumpr-supported-ff69b4?logo=github&link=https://github.com/haya14busa/action-bumpr)](https://github.com/haya14busa/action-bumpr)


This is a Github Action to lint and fix SQL on PRs with [SQLFluff](https://github.com/sqlfluff/sqlfluff).

Supports 3 operations:

* **"lint"** - Runs `sqlfluff lint`
* **"fix"** - Runs `sqlfluff fix` then makes GitHub Suggestions on the PR.
* **"commit"** - Runs "fix" then commits those changes.

This uses [ReviewDog](https://github.com/reviewdog/reviewdog) to Post comments and suggestions on the PR.

## Lint Mode

The lint mode leaves comments on github pull requests. Comments are pointed out by SQLFluff.
![github-pr-review demo (lint)](./docs/images/github-pr-review-demo-lint.png)

## Fix Mode

Suggest mode makes GitHub Suggestions based on `sqlfluff fix`.
![github-pr-review demo (fix)](./docs/images/github-pr-review-demo-fix.png)

## Commit Mode

Commit mode commits and pushes the changes from `sqlfluff fix` to your PR.

## Example: Fix Mode

```yaml
name: sqlfluff with reviewdog
on:
  pull_request:
jobs:
  test-check:
    name: runner / sqlfluff (github-check)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: yu-iskw/action-sqlfluff@v3
        id: lint-sql
        with:
          github_token: ${{ secrets.github_token }}
          reporter: github-pr-review
          sqlfluff_version: "1.4.5"
          sqlfluff_command: "fix" # Or "lint" or "commit"
          config: "${{ github.workspace }}/.sqlfluff"
          paths: '${{ github.workspace }}/models'
      - name: 'Show outputs (Optional)'
        shell: bash
        run: |
          echo '${{ steps.lint-sql.outputs.sqlfluff-results }}' | jq -r '.'
          echo '${{ steps.lint-sql.outputs.sqlfluff-results-rdjson }}' | jq -r '.'
```

## Example: Commit mode

This example differs slightly in that it:

1. Writes the GCP-SA-Key Secret to disk, then uses that to authenticate with BigQuery. In dbt mode, SQLFluff reads some metadata from the DB.
1. Does the Git Checkout in such a way that doesn't leave it in a detached-HEAD state. The `with:` block on `checkout@v3`.

```yaml
name: 📜 SQLFluff
on:
  pull_request:

jobs:
  format_fix:
    runs-on: ubuntu-latest
    env:
      DBT_PROFILES_DIR: ${{ github.workspace }}/.github/ci_cd
      PR_BRANCH_NAME: ${{ github.event.pull_request.head.ref }}
    steps:
      - name: 🛒 Checkout repo
        uses: actions/checkout@v3
        with:
          ref: ${{ github.event.pull_request.head.ref }}
      - name: 🔑 Write GCP Service-Acct Key
        env:
          BIGQUERY_KEYFILE: ${{ secrets.BIGQUERY_KEYFILE }}
        run: |
          echo "$BIGQUERY_KEYFILE" > ${{ github.workspace }}/.github/ci_cd/dbt-service-account.json
      - name: 📜 SQLFLuff
        uses: trainual/action-sqlfluff-commits@main
        id: lint-sql
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          reporter: github-pr-review
          sqlfluff_version: 1.4.5
          sqlfluff_command: commit
          templater: dbt
          config: ${{ github.workspace }}/.sqlfluff
          paths: ${{ github.workspace }}/models
          extra_requirements_txt: ${{ github.workspace }}/.github/ci_cd/requirements.txt
      - name: 🐣 Show outputs
        shell: bash
        run: |
          echo '${{ steps.lint-sql.outputs.sqlfluff-results }}' | jq -r '.'
          echo '${{ steps.lint-sql.outputs.sqlfluff-results-rdjson }}' | jq -r '.'
```

## NOTE
The tested sqlfluff versions in the repositories are:
- 1.0.0
- 1.1.0
- 1.2.0
- 1.3.0
- 1.4.5

## CAUTION
Because dbt-core==1.4 changes the implementation of custom exceptions like `CompilationException`, sqlfluff 1.4.5 or less doesn't work with dbt-core 1.4 or later.
So, we have to use dbt-core 1.3 or less until the subsequent change is released.
- [Handle renamed dbt exceptions by greg\-finley · Pull Request \#4317 · sqlfluff/sqlfluff](https://github.com/sqlfluff/sqlfluff/pull/4317)

## Input

```yaml
inputs:
  github_token:
    description: 'GITHUB_TOKEN'
    required: true
    default: '${{ github.token }}'
  working-directory:
    description: 'working directory'
    required: false
    default: '${{ github.workspace }}'
  ### Flags for reviewdog ###
  level:
    description: 'Report level for reviewdog [info,warning,error]'
    required: false
    default: 'error'
  reporter:
    description: 'Reporter of reviewdog command [github-check,github-pr-review].'
    required: false
    default: 'github-check'
  filter_mode:
    description: |
      Filtering mode for the reviewdog command [added,diff_context,file,nofilter].
      Default is file.
    required: false
    default: 'file'
  fail_on_error:
    description: |
      Exit code for reviewdog when errors are found [true,false]
      Default is `false`.
    required: false
    default: 'false'
  reviewdog_version:
    description: 'reviewdog version'
    required: false
    default: '0.13.0'
  ### Flags for SQLFluff ###
  sqlfluff_version:
    description: |
      sqlfluff version. Use the latest version if not set.
    required: false
    default: '1.4.5'
  sqlfluff_command:
    description: 'The operation to perform. One of lint, fix, and commit'
    required: false
    default: 'lint'
  paths:
    description: |
      PATH is the path to a sql file or directory to lint.
      This can be either a file ('path/to/file.sql'), a path ('directory/of/sql/files'), a single ('-') character to indicate reading from *stdin* or a dot/blank ('.'/' ') which will be interpreted like passing the current working directory as a path argument.
    required: true
  encoding:
    description: 'Specifiy encoding to use when reading and writing files. Defaults to autodetect.'
    required: false
    default: ''
  config:
    description: |
      Include additional config file.
      By default the config is generated from the standard configuration files described in the documentation.
      This argument allows you to specify an additional configuration file that overrides the standard configuration files.
      N.B. cfg format is required.
    required: false
    default: ''
  exclude-rules:
    description: |
      Exclude specific rules.
      For example specifying –exclude-rules L001 will remove rule L001 (Unnecessary trailing whitespace) from the set of considered rules.
      This could either be the allowlist, or the general set if there is no specific allowlist.
      Multiple rules can be specified with commas e.g. –exclude-rules L001,L002 will exclude violations of rule L001 and rule L002.
    required: false
    default: ''
  rules:
    description: |
      Narrow the search to only specific rules.
      For example specifying –rules L001 will only search for rule L001 (Unnecessary trailing whitespace).
      Multiple rules can be specified with commas e.g. –rules L001,L002 will specify only looking for violations of rule L001 and rule L002.
    required: false
    default: ''
  templater:
    description: 'The templater to use'
    required: false
    default: ''
  disable-noqa:
    description: 'Set this flag to ignore inline noqa comments.'
    required: false
    default: ''
  dialect:
    description: 'The dialect of SQL to lint'
    required: false
    default: ''
  #  annotation-level:
  #    description: |
  #      When format is set to github-annotation, default annotation level.
  #      Options
  #      notice | warning | failure
  #    required: false
  #    default: ''
  #  nofail:
  #    description: |
  #      If set, the exit code will always be zero, regardless of violations found.
  #      This is potentially useful during rollout.
  #    required: false
  #    default: ''
  #  disregard-sqlfluffignores:
  #    description: 'Perform the operation regardless of .sqlfluffignore configurations'
  #    required: false
  #    default: ''
  processes:
    description: 'The number of parallel processes to run.'
    required: false
    default: "2"
  # Mainly used to install dbt adapters
  # NOTE:
  # sqlfluff tries to dynamically import a dbt adapter based on a configuration.
  # There is no great way to dynamically install required dbt adapters to fit to users of action.
  # It might be possible to support only dbt adapters craeted by dbt labo.
  # But, as that doesn't support 3rd party dbt adapters, we have no choise but for users to pass their custom extra requirements.txt.
  extra_requirements_txt:
    description: |
      A path to your custom `requirements.txt` to install extra modules for your dbt adapters.
      Please make sure not to contain `sqlfluff` and its dependent packages, because the action can be broken by the conflicts.
    required: false
    default: ''
```

## Outputs
The outputs are available only when the `sqlfluff_command` input is `lint`.
```yaml
outputs:
  sqlfluff-results:
    description: 'The JSON object string of sqlfluff results'
    value: ${{ steps.sqlfluff-with-reviewdog-in-composite.outputs.sqlfluff-results }}
  sqlfluff-exit-code:
    description: 'The exit code of sqlfluff'
    value: ${{ steps.sqlfluff-with-reviewdog-in-composite.outputs.sqlfluff-exit-code }}
  sqlfluff-results-rdjson:
    description: 'The JSON object string of sqlfluff results'
    value: ${{ steps.sqlfluff-with-reviewdog-in-composite.outputs.sqlfluff-results-rdjson }}
  reviewdog-return-code:
    description: 'The exit code of reviewdog'
    value: ${{ steps.sqlfluff-with-reviewdog-in-composite.outputs.reviewdog-return-code }}
```

## Development

### Release

#### [haya14busa/action-bumpr](https://github.com/haya14busa/action-bumpr)
You can bump version on merging Pull Requests with specific labels (bump:major,bump:minor,bump:patch).
Pushing tag manually by yourself also work.

#### [haya14busa/action-update-semver](https://github.com/haya14busa/action-update-semver)

This action updates major/minor release tags on a tag push. e.g. Update v1 and v1.2 tag when released v1.2.3.
ref: https://help.github.com/en/articles/about-actions#versioning-your-action

### Lint - reviewdog integration

This reviewdog action template itself is integrated with reviewdog to run lints
which is useful for Docker container based actions.

![reviewdog integration](https://user-images.githubusercontent.com/3797062/72735107-7fbb9600-3bde-11ea-8087-12af76e7ee6f.png)

Supported linters:

- [reviewdog/action-shellcheck](https://github.com/reviewdog/action-shellcheck)
- [reviewdog/action-hadolint](https://github.com/reviewdog/action-hadolint)
- [reviewdog/action-misspell](https://github.com/reviewdog/action-misspell)

### Dependencies Update Automation
This repository uses [reviewdog/action-depup](https://github.com/reviewdog/action-depup) to update
reviewdog version.

![reviewdog depup demo](https://user-images.githubusercontent.com/3797062/73154254-170e7500-411a-11ea-8211-912e9de7c936.png)
