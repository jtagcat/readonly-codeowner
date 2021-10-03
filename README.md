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


## Implementation
- code owners icon is not shown for non-write users, sucks

Github Actions can't do this:
 - No Checks ('external tests', before merging, blockers)
 - No good way to have one permissions file

### Github App
#### Permissions
- Repository permissions: checks: rw
  - status indicators and blocking
- Repository permissions: pulls: rw
  - get files
  - get approvals
  - ask for reviews
  - get author
  - merge
- Repository permissions: issues: rw
  - for commenting on pulls
- Repository permissions: metadata: ro
  - get members of orgs/groups
  - get users/groups? with write access to the repo
- Repository permissions: single file: ro
  - `.github/CODEOWNERS_ro` — source for our definitions

#### Workflow
1. Incoming PR
1. Get merge requirements
  1. single file: config file
  1. PR list of files affected
  1. for each file, find definition (may be fallback) to follow 
  1. account for recursive: rule:`dir/` and files:`dir/foo`,`dir/bar`, merge them
1. order rules on most specific match first
  1. first by match outreach (did we get a match, or had to go up in the tree searching for CODEOWNER files)
  1. result sort by directory depth
  1. then by user-set priority (not documented above!)
  (result sort priority is the opposite)
1. Start keeping tabs on how many reviews we need or have, we might have enough already (self-approve)
  - display status in a check
  - start merge countdown
  - merge
3. request groups to review in order OR all at the same time
  1. For each file (group), we have usernames/groups/orgs on who can review, mention them
  2. For sequential review, don't clutter up the comments: delete previous comments or edit them (does it mention?)
4. An approval is submitted
  1. Check if it's from an allowed user:
    1. username match
    1. is part of a group/org
  1. Update keeping tabs
  2. Ping the next group (if sequential)
