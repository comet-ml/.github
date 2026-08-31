# .github

This repository holds organization-wide GitHub configuration for [comet-ml](https://github.com/comet-ml). It is not a code project.

| Path | What it does |
| --- | --- |
| `profile/README.md` | Renders as the banner on the [comet-ml organization page](https://github.com/comet-ml) |
| `SECURITY.md` | Default security policy for every comet-ml repo without its own |
| `CODE_OF_CONDUCT.md` | Default code of conduct |
| `SUPPORT.md` | Default "where do I get help" page |
| `.github/ISSUE_TEMPLATE/` | Default issue forms |
| `.github/PULL_REQUEST_TEMPLATE.md` | Default pull request template |

Every file here is a **fallback**: a repository that defines its own copy of one of these files always wins. Changing a file in this repo changes the default for all comet-ml repos that have not overridden it, so review changes with that blast radius in mind.

See [GitHub's docs on default community health files](https://docs.github.com/communities/setting-up-your-project-for-healthy-contributions/creating-a-default-community-health-file) for the full list of supported paths.
