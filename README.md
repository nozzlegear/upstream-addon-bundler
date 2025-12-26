# Upstream Addon Bundler Template

This template repository contains the workflows necessary for automatically syncing and bundling WoW addons from upstream repositories. While [the CurseBreaker CLI](https://github.com/AcidWeb/CurseBreaker) I've been using to manage addons since 2019 *does* support several addon repositories, including Wago and WowInterface, it does *not* support Curseforge's closed API. Luckily, CurseBreaker can install addons via Github – as long as they're [bundled in the correct format](https://github.com/BigWigsMods/packager), which some addon authors don't do.

The workflows in this template repository are my solution to that problem. Instead of bothering every addon author by asking them to publish to addon repositories they don't use, or to add GitHub workflows when they may not be familiar with how they work, I can simply fork their repository, and let these workflows release a new bundle whenever the author updates their addon.

## Setup Instructions

### 1. Create Repository from Template

1. Click "Use this template" on GitHub
2. Create a **private** repository (recommended for personal addon builds, as a courtesy to the addon author)
3. Clone your new repository locally

### 2. Run Initial Setup

After creating your repository, run the setup workflow to configure the upstream source:

#### Using GitHub CLI:
```bash
gh workflow run setup.yaml -f upstream_repo=username/repo-name
```

#### Using GitHub UI:
1. Go to **Actions** tab
2. Select **Initial Setup** workflow
3. Click **Run workflow**
4. Enter the upstream repository (format: `username/repo-name`)
5. Click **Run workflow**

The setup workflow will:
- Fetch and merge all files from the upstream repository (replacing template files like README.md, LICENSE, etc.)
- Create a `.upstream-repo` file containing the upstream repository URL
- Restore the workflow files to enable automation
- Trigger the initial sync
- Create the first release

### 3. Verify Setup

Check that:
- A `.upstream-repo` file exists in your repository root
- The repository contains the addon files from upstream
- The initial sync has completed
- A release has been created

### 4. Install Addon via [CurseBreaker](https://github.com/AcidWeb/CurseBreaker)

#### Set the CurseBreaker Github token:
CurseBreaker needs a GitHub fine-grained personal access token with read-only content access for your private repository. Go to your [Personal Access Tokens page in GitHub's settings](https://github.com/settings/personal-access-tokens) to create a token. Once you have it, pass it to CurseBreaker:

```bash
cursebreaker set gh_api [API_KEY]
```

#### Install the addon
After you've set the GitHub API key, you can install the addon by running CurseBreaker's install command pointing it to your new GitHub repository:

```bash
# Use the "gh:" prefix + username/repository
cursebreaker install gh:nozzlegear/repository_name
# Or use the full GitHub url
cursebreaker install https://github.com/nozzlegear/repository_name
```

#### Install error
```bash
┌────────┬───────────────┬─────────┐
│ Status │ Name / Author │ Version │
├────────┼───────────────┼─────────┤
└────────┴───────────────┴─────────┘
RuntimeError: Failed to parse addon data: https://github.com/nozzlegear/foo
```
If you get an error that looks something like the above, and CurseBreaker says it failed to parse addon data from your repository, there are three things you should check:

1. Make sure you correctly set the GitHub token on CurseBreaker (`cursebreaker set gh_api [API_KEY]`)
2. Make sure youve given that GitHub token permission to read the contents of your repository.
3. Check that the release workflow has run and published a release. There should be a zip file attached to the release which contains the addon files.

## Workflows

### Initial Setup (`setup.yaml`)
One-time setup workflow that configures the repository.

**Trigger**: Manual (`workflow_dispatch`)

**Input**:
- `upstream_repo`: The upstream repository (e.g., `username/repo-name`)

**Process**:
1. Fetches the upstream repository
2. Replaces all template files with upstream content
3. Restores workflow files from template
4. Creates `.upstream-repo` config file with the upstream URL
5. Commits and pushes all changes
6. Triggers the initial sync

### Sync with Upstream (`sync-upstream.yaml`)
Automatically syncs with the upstream repository.

**Triggers**:
- Daily at midnight UTC (7pm CDT / 6pm CST)
- Manual dispatch

**Process**:
1. Reads upstream URL from `.upstream-repo` file
2. Fetches upstream changes
3. Checks for new commits
4. Rebases local main on upstream/main
5. Force pushes with `--force-with-lease`
6. Creates timestamped tag (`auto-sync-YYYYMMDD-HHMMSS`)
7. Triggers release workflow if changes detected

### Pack and Release (`release.yaml`)
Packages and releases the addon.

**Triggers**:
- When tags are pushed
- Manual dispatch
- After successful sync with changes

**Process**:
1. Creates timestamped tag (if manually triggered)
2. Uses BigWigsMods packager to create GitHub release

## Workflow Chain

```
Daily Schedule → Sync → Detect Changes → Rebase → Tag → Release
```

## Manual Operations

### Trigger Sync Manually
```bash
gh workflow run sync-upstream.yaml
```

### Trigger Release Manually
```bash
gh workflow run release.yaml
```

## Requirements

- GitHub repository with Actions enabled
- Upstream repository must be a valid WoW addon (compatible with BigWigsMods packager)

## Notes

- The repository should be set to **private**, as a courtesy to the addon author, if you're creating personal builds
- Sync uses `--force-with-lease` to safely force-push rebased changes
- Tags are automatically created with timestamps to trigger releases
- The BigWigsMods packager requires specific addon structure (TOC file, etc.)
- The `.upstream-repo` file stores the upstream repository URL and is read by the sync workflow
- Do not delete the `.upstream-repo` file or the sync workflow will fail
