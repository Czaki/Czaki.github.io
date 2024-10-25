---
date: 2024-10-25
categories:
  - GitHub
  - Testing
  - Article
tags:
    - GitHub
    - Testing
    - Article
draft: true
---

# Pinning Test Dependencies

This note is about pinning CI dependencies for better CI stability. It is based on work done in the [napari](https://github.com/napari/napari) project

<!-- more -->

## Motivation

### Guaranteeing test stability

I'm one of core developer of [napari](https://github.com/napari/napari) project. This Project has 35 direct dependencies and 116 total dependencies for minimum operational capability. For testing, it uses 188 dependencies in total, and for documentation assembly, it uses up to 224 dependencies. With such massive dependency chains, it's not uncommon for us to encounter broken tests due to updates in the dependencies.

As open-source projects, we rely on the maintainers of these dependencies to ensure that their libraries are backward compatible. But we also understand that maintaining backward compatibility can be challenging, and sometimes backward incompatible changes are introduced by mistake.


Given the substantial volume of these dependencies, there are instances where the maintainers of other libraries might unintentionally disrupt certain API in their packages during updates. These occasional or unintentional disruptions can cause unforeseen complications. On the other hand, there are times when such breakages are deliberate. Providing a backward compatibility layer can be tricky in these situations. Balancing the maintenance of deprecated APIs with both reasonable time constraints and performance requirements often proves unfeasible in the long run. These factors illuminated the need for us to commence pinning our test dependencies.

Prior to implementing the pinning of test dependencies, we frequently encountered issues with broken tests, roughly about once a month. Many of our contributors are novices in programming and contributing to open-source projects. They often struggle with understanding why tests fail and whether it's an outcome of their modifications. Thus, alongside fixing the test, we have to embark on additional maintenance work to clearly explain the circumstances leading to the failure.

### Enhancing reproducibility in science-based projects

Napari, being in its alpha phase, is substantially a science-based project. As such, we occasionally have to make API modifications. Consequently, newer library updates may not be compatible with older code. However, in scientific research, there's often a need to re-run older codes for comparison purposes.

Luckily, there are project tools like [`pypi-timemachine`](https://pypi.org/project/pypi-timemachine/) that design proxy servers to retrieve PyPi packages up till a specified date. Despite this, providing a definitive list of constraints enables the environment to be easily replicated without resorting to additional tools and avoiding discrepancies that could impact results. Also, this list of constraints simplifies the process of instructing others on how to recreate the environment.


<!-- more -->


## Distinguishing between pip's Requirements and Constraints

In pip, requirements are a list of packages that need to be installed and could contain version constraints and [extras](https://pip.pypa.io/en/stable/reference/requirement-specifiers/#requirement-specifiers) definitions. On the other hand, constraints are version boundaries for packages that should be considered during the resolution process. 

Requirements can be passed by using the package name like `pip install numpy`, or via a file like `pip install -r requirements.txt`. Constraints, however, can be passed using the file `pip install napari -c constraints.txt` or via environment variables `PIP_CONSTRAINT=constraints.txt pip install napari`. It's important to note that putting packages in constraints.txt does not install them. It only limits the versions that could be installed.

## What we need
 
Here is a list of our needs:
 
1. We need to pin all dependencies for tests and docs.
2. We need to pin per Python version, as some dependencies do not provide single-version support for all Python versions that we support.
3. Constraints need to be automatically updated (creating Pull Requests).
4. As we use upper constraints for some dependencies, we need to be able to upgrade dependencies in any PR, that comes from a forked repository. 
5. We need to minimize the infrastructure cost for this process.
 
## Our solution
 
Here, I will describe our solution to the problem. Today's date is 2023.11.7. Workflow may change over time. The most recent version of the workflow is available [here](https://github.com/napari/napari/blob/main/.github/workflows/upgrade_test_constraints.yml).

### Tigers

```yaml
on:
  workflow_dispatch: # Allow running on-demand
  schedule:
    # Runs every Monday at 8:00 UTC (4:00 Eastern)
    - cron: '0 8 * * 1'

  issue_comment:
    types: [ created ]

  pull_request:
    paths:
      - '.github/workflows/upgrade_test_constraints.yml'
```

We have 4 triggers:

1. `workflow_dispatch` - for the manual trigger of update or create PR with constraints upgrade. 
2. `schedule` - for automatic weekly update of constraints.
3. `issue_comment` - for the manual trigger of constraints upgrade from comment. This generates new constraints files that needs to be manually applied, because of the lack of permissions to push to the different repositories.
4. `pull_request` - for development purposes.


### Setup action
  
```yaml
jobs:
upgrade:
  permissions:
    pull-requests: write
    issues: write
  name: Upgrade & Open Pull Request
  if: (github.event.issue.pull_request != '' && contains(github.event.comment.body, '@napari-bot update constraints')) || github.event_name == 'workflow_dispatch' || github.event_name == 'schedule' || github.event_name == 'pull_request'
  runs-on: ubuntu-latest
```

To be able to open PR from the workflow, we need to add `pull-request: write` and `issues: write` permissions to the workflow.
To run a workflow only on comments containing `@napari-bot update constraints` we use `contains(github.event.comment.body, '@napari-bot update constraints')` condition. 
The rest of the conditions are to allow starting workflow from triggers described above.

### Visual signaling that workflow is running

```yaml
      - name: Add eyes reaction
        # show that workflow has started
        if: github.event_name == 'issue_comment'
        run: |
          COMMENT_ID=${{ github.event.comment.id }}
          curl \
            -X POST \
            -H "Accept: application/vnd.github+json" \
            -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
            "https://api.github.com/repos/${{ github.repository }}/issues/comments/$COMMENT_ID/reactions" \
            -d '{"content": "eyes"}'
```

```yaml
      - name: Add rocket reaction
        # inform that new constraints are available in artifacts
        if: github.event_name == 'issue_comment'
        run: |
          COMMENT_ID=${{ github.event.comment.id }}
          curl \
            -X POST \
            -H "Accept: application/vnd.github+json" \
            -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
            "https://api.github.com/repos/${{ github.repository }}/issues/comments/$COMMENT_ID/reactions" \
            -d '{"content": "rocket"}'
```

To signal that the workflow is running, we add `eyes` reaction to the comment that triggered the workflow. 
When the workflow is finished, we add `rocket` reaction to the comment that triggered the workflow. You may remove it or replace with other mechanisms.

### Get repo details

```yaml

      - name: Get PR details
        # extract PR number and branch name from issue_comment event
        if: github.event_name == 'issue_comment'
        run: |
          PR_number=${{ github.event.issue.number }}
          PR_data=$(curl \
              -H "Accept: application/vnd.github.v3+json" \
              "https://api.github.com/repos/${{ github.repository }}/pulls/$PR_number" \
          )

          FULL_NAME=$(echo $PR_data  | jq -r .head.repo.full_name)
          echo "FULL_NAME=$FULL_NAME" >> $GITHUB_ENV

          BRANCH=$(echo $PR_data  | jq -r .head.ref)
          echo "BRANCH=$BRANCH" >> $GITHUB_ENV

        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Get repo info
        # when schedule or workflow_dispatch triggers workflow, then we need to get info about which branch to use
        if: github.event_name != 'issue_comment' && github.event_name != 'pull_request'
        run: |
          echo "FULL_NAME=${{ github.repository }}" >> $GITHUB_ENV
          echo "BRANCH=${{ github.ref_name }}" >> $GITHUB_ENV
```

We use these two conditional steps to determine the repository name and branch name. 
Runtime determines this allows to use of the same workflow for different repositories and branches. 
This is useful when you want to test the workflow before applying it to the main repository,
but may allow you to easier use it in your repository.


### Checkout repository

```yaml
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Clone docs repo
        uses: actions/checkout@v4
        with:
          path: docs  # place in a named directory
          repository: napari/docs

      - name: Clone target repo (remote)
        uses: actions/checkout@v4
        if: github.event_name == 'issue_comment'
        with:
          path: napari_repo  # place in a named directory
          repository: ${{ env.FULL_NAME }}
          ref: ${{ env.BRANCH }}
          token: ${{ secrets.GHA_TOKEN_BOT_REPO }}

    
      - name: Clone target repo (pull request)
        # we need separate step as passing empty token to actions/checkout@v4 will not work
        uses: actions/checkout@v4
        if: github.event_name == 'pull_request'
        with:
          path: napari_repo  # place in a named directory

      - name: Clone target repo (main)
        uses: actions/checkout@v4
        if: github.event_name != 'issue_comment' && github.event_name != 'pull_request'
        with:
          path: napari_repo  # place in a named directory
          repository: ${{ env.FULL_NAME }}
          ref: ${{ env.BRANCH }}
          token: ${{ secrets.GHA_TOKEN_NAPARI_BOT_MAIN_REPO }}
```

We use have docs in a separate repository. We also want to update the constraints for the docs build. If you have a single repository, you may skip the second step.

We use two copies of the target repository. The 
