# readonly-codeowner
[Codeowners](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners) without write access. Whitelist PR automerge for specific files-users.

Essentially give write access via PRs to users, scoped to a path.

**There is no bot, this is an idea.**

## Idea

file: `.github/CODEOWNERS_ro` (can be later stolen by GitHub, ro definitions compatible with CODEOWNERS, keeping it seperate for now)

Special line: `fallback: @writeuser` (delimiter `:`, normal lines/paths don't have `:` to seperate them)  
This is a who to ping in case we can't go further than the repo root (traversing for reviewers/perms), when a review is needed 

 - `translations/et_EE.js @jtagcat r1 o1 strict`:
   - for path `translations/et_EE.js`
   - org/group/user @jtagcat is codeowner
   - @jtagcat is allowed to self-approve
   - at least 1 review
   - of what 1 must be from a code owner
   - strict: any user with write access to the repo is not a valid reviewer for the mentioned files
   - (note: GitHub prohibits self-approving PRs, submitting a non-draft PR means approval)
   - result: @jtagcat submits a non-draft PR, it is queued to merge by the bot.
   - result: @octocat submits a PR, review from @jtagcat is needed. After a grey-checkmark (no write access) review from @jtagcat, PR is queued to merged by bot.
   - result: @octocat submits a PR with files `README.md` and `translations/et_EE.js`, review from @jtagcat is needed. After a grey-checkmark (no write access) review from @jtagcat, comment is left by bot / test succeeeds. Directory tree is traversed for higher code owners, or fallback org/user/group (@writeuser), whom is then pinged, asking review/merge. Continue manually with write-access users.
 - `translations/ja_JP.js !@octocat !r@octofederation r2 o1`
   - for path `translations/ja_JP.js`
   - user @octocat is codeowner
   - any user with write access to the repo is valid reviewer
   - r: group @octofederation is valid reviewer and 
   - @octocat and @octofederation aren't allowed to self-review
   - at least 2 reviews
   - of what 1 must be from code owner
   - translates to the following YAML:
     ```yaml
      - translations/ja_JP:
        reviews: 2
          codeowner: 1
          strict: False
        codeowners:
          - @octocat
            self-review: False
        reviewers:
          - @octofederation
            self-review: False
     ```
   - result: @octocat submits a PR, it can't be merged. It can be merged since there is no other approving codeowner. Fallback (hail @writeuser)
   - result: @randuser submits a PR, review from @octocat and at least one other reviewer (from @octofederation or users with write access) is needed. Bot queues to merge after approvals.
***
  
 - what happens when repo settings need at least 10 reviews, does the bot bypass it (how?)?
 - display reviews needed as a test (pending/running)
 - countdown 5m as an undo buffer when all reviews are met (can we do checkmark?), then bot merges
