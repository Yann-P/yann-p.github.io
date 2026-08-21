---
title: "Github actions credentials security without breaking your workflows"
date: 2026-05-11
---

In the light of the recent [Tanstack supply-chain attack](https://tanstack.com/blog/npm-supply-chain-compromise-postmortem), pipeline security has become the priority of several projects.

Mistakes in github actions are tricky to see and can have devastating consequences, through PR-controlled variable injection [^2].

Luckily there are powerful tools to catch them:
- [Zizmor](https://github.com/zizmorcore/zizmor)
- [Actionlint](https://github.com/rhysd/actionlint)

One common suggestion from Zizmor is to set  `persist-credentials: false` when using actions/checkout [^1]

## The problem: `persist-credentials: false` + https git commands = boom

If you combine these two ingredients you will be greeted with the error

```
fatal: could not read Username for 'https://github.com/': No such device or address
```

## Workaround 1: Let `gh` provide credentials

```diff
 - name: Commit and push
+   env:
+     GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
   run: |
+    gh auth setup-git
     git commit -m "your message"
     git push
```

`gh auth setup-git` configures git to fetch credentials from the `gh` command, which in turn reads the token from `GH_TOKEN`.

This avoids storing it in `.git/config` (which is what `persist-credentials: false` protects us from).

{{< callout type="warning" >}}
If you self-host your runner, or use a job with a custom `container:`, you need to install and set up the `gh` cli yourself. https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-github-cli

{{< /callout >}}


## Workaround 2: Inject the token into the remote URL (and clean it after)

```diff
 - name: Commit and push
   env:
     GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
   run: |
     git commit -m "your message"
+    git remote set-url origin "https://x-access-token:${GH_TOKEN}@github.com/${{ github.repository }}"
     git push

+    # Embedding the token in the remote URL writes it to `.git/config`, 
+    # so we reset the URL after pushing.
+    git remote set-url origin "https://github.com/${{ github.repository }}"
```

{{< callout type="warning" >}}
If git push fails, the credentials stay in `.git/config`. This is an unlikely attack vector but workaround 1 is safer.
{{< /callout >}}



[^1]: https://docs.zizmor.sh/audits/#artipacked
[^2]: https://securitylab.github.com/resources/github-actions-preventing-pwn-requests/
[^3]: https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-github-cli
