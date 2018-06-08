[Source](https://docs.gitlab.com/ee/user/project/import/github.html "Source")

# Import your project from GitHub to GitLab

Using the importer, you can import your GitHub repositories to GitLab.com or to
your self-hosted GitLab instance.

## Overview

NOTE: **Note:**
While these instructions will always work for users on GitLab.com, if you are an
administrator of a self-hosted GitLab instance, you will need to enable the
[GitHub integration][gh-import] in order for users to follow the preferred
import method described on this page. If this is not enabled, users can alternatively import their
GitHub repositories using a [personal access token](#using-a-github-token) from GitHub,
but this method will not be able to associate all user activity (such as issues and pull requests)
with matching GitLab users. As an administrator of a self-hosted GitLab instance, you can also use
the [GitHub rake task](https://gitlab.com/gitlab-org/gitlab-ee/blob/master/doc/administration/raketasks/github_import.md) to import projects from
GitHub without the constraints of a Sidekiq worker.

The following aspects of a project are imported:
  * Repository description (GitLab.com & 7.7+)
  * Git repository data (GitLab.com & 7.7+)
  * Issues (GitLab.com & 7.7+)
  * Pull requests (GitLab.com & 8.4+)
  * Wiki pages (GitLab.com & 8.4+)
  * Milestones (GitLab.com & 8.7+)
  * Labels (GitLab.com & 8.7+)
  * Release note descriptions (GitLab.com & 8.12+)
  * Pull request review comments (GitLab.com & 10.2+)
  * Regular issue and pull request comments

References to pull requests and issues are preserved (GitLab.com & 8.7+), and
each imported repository defaults to `private` but [can be made public](https://gitlab.com/gitlab-org/gitlab-ee/blob/master/doc/user/project/settings/index.md#sharing-and-permissions), as needed.

## How it works

When issues and pull requests are being imported, the importer attempts to find their GitHub authors and
assignees in the database of the GitLab instance (note that pull requests are called "merge requests" in GitLab).

For this association to succeed, prior to the import, each GitHub author and assignee in the repository must
have either previously logged in to a GitLab account using the GitHub icon **or** have a GitHub account with
a [public email address](https://help.github.com/articles/setting-your-commit-email-address-on-github/) that
matches their GitLab account's email address.

If a user referenced in the project is not found in GitLab's database, the project creator (typically the user
that initiated the import process) is set as the author/assignee, but a note on the issue mentioning the original
GitHub author is added.

The importer creates any new namespaces (groups) if they do not exist, or, if the namespace is taken, the
repository is imported under the namespace of the user who initiated the import process. The namespace/repository
name can also be edited, with the proper permissions.

The importer will also import branches on forks of projects related to open pull requests. These branches will be
imported with a naming scheme similar to `GH-SHA-username/pull-request-number/fork-name/branch`. This may lead to
a discrepancy in branches compared to those of the GitHub repository.

For additional technical details, you can refer to the
[GitHub Importer](https://gitlab.com/gitlab-org/gitlab-ee/blob/master/doc/development/github_importer.md "Working with the GitHub importer")
developer documentation.

## Import your GitHub repository into GitLab

### Using the GitHub integration

Before you begin, ensure that any GitHub users who you want to map to GitLab users have either:

1. A GitLab account that has logged in using the GitHub icon
\- or -
2. A GitLab account with an email address that matches the [public email address](https://help.github.com/articles/setting-your-commit-email-address-on-github/) of the GitHub user

User-matching attempts occur in that order, and if a user is not identified either way, the activity is associated with
the user account that is performing the import.

NOTE: **Note:**
If you are using a self-hosted GitLab instance, this process requires that you have configured the
[GitHub integration][gh-import].

1. From the top navigation bar, click **+** and select **New project**.
2. Select the **Import project** tab and then select **GitHub**.
3. Select the first button to **List your GitHub repositories**. You are redirected to a page on github.com to authorize the GitLab application.
4. Click **Authorize gitlabhq**. You are redirected back to GitLab's Import page and all of your GitHub repositories are listed.
5. Continue on to [selecting which repositories to import](#selecting-which-repositories-to-import).

### Using a GitHub token

NOTE: **Note:**
For a proper author/assignee mapping for issues and pull requests, the [GitHub integration method (above)](#using-the-github-integration)
should be used instead of the personal access token. If you are using GitLab.com or a self-hosted GitLab instance with the GitHub
integration enabled, that should be the preferred method to import your repositories. Read more in the [How it works](#how-it-works) section.

If you are not using the GitHub integration, you can still perform an authorization with GitHub to grant GitLab access your repositories:

1. Go to https://github.com/settings/tokens/new
2. Enter a token description.
3. Select the repo scope.
4. Click **Generate token**.
5. Copy the token hash.
6. Go back to GitLab and provide the token to the GitHub importer.
7. Hit the **List Your GitHub Repositories** button and wait while GitLab reads your repositories' information.
   Once done, you'll be taken to the importer page to select the repositories to import.

### Selecting which repositories to import

After you have authorized access to your GitHub repositories, you are redirected to the GitHub importer page and
your GitHub repositories are listed.

1. By default, the proposed repository namespaces match the names as they exist in GitHub, but based on your permissions,
   you can choose to edit these names before you proceed to import any of them.
2. Select the **Import** button next to any number of repositories, or select **Import all repositories**.
3. The **Status** column shows the import status of each repository. You can choose to leave the page open and it will
   update in realtime or you can return to it later.
4. Once a repository has been imported, click its GitLab path to open its GitLab URL.

## Mirroring and pipeline status sharing

Depending your GitLab tier, [project mirroring](https://gitlab.com/gitlab-org/gitlab-ee/blob/master/doc/workflow/repository_mirroring.md) can be set up to keep
your imported project in sync with its GitHub copy.

Additionally, you can configure GitLab to send pipeline status updates back GitHub with the
[GitHub Project Integration](https://gitlab.com/gitlab-org/gitlab-ee/blob/master/doc/user/project/integrations/github.md). **[PREMIUM]**

If you import your project using [CI/CD for external repo](https://gitlab.com/gitlab-org/gitlab-ee/blob/master/doc/ci/ci_cd_for_external_repos/index.md), then both
of the above are automatically configured. **[PREMIUM]**

## Improving the speed of imports on self-hosted instances

NOTE: **Note:**
Admin access to the GitLab server is required.

For large projects it may take a while to import all data. To reduce the time necessary, you can increase the number of
Sidekiq workers that process the following queues:

* `github_importer`
* `github_importer_advance_stage`

For an optimal experience, it's recommended having at least 4 Sidekiq processes (each running a number of threads equal
to the number of CPU cores) that *only* process these queues. It's also recommended that these processes run on separate
servers. For 4 servers with 8 cores this means you can import up to 32 objects (e.g., issues) in parallel.

Reducing the time spent in cloning a repository can be done by increasing network throughput, CPU capacity, and disk
performance (e.g., by using high performance SSDs) of the disks that store the Git repositories (for your GitLab instance).
Increasing the number of Sidekiq workers will *not* reduce the time spent cloning repositories.

[gh-import]: https://gitlab.com/gitlab-org/gitlab-ee/blob/master/doc/integration/github.md "GitHub integration"
