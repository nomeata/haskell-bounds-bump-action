A haskell dependency bumper action
==================================

A Github Action to create PRs that update your cabal dependencies.

Usage
-----

1. Copy this file to `.github/workflows/bump.yml`:

   ```
   name: Create dependency bump PR
   on:
     # allows manual triggering from https://github.com/../../actions/workflows/bump.yml
     workflow_dispatch:
     # runs weekly on Thursday at 8:00
     schedule:
       - cron: '0 8 * * 4'

   permissions:
     contents: write
     pull-requests: write

   jobs:
     bump:
       runs-on: ubuntu-latest
       steps:
       - uses: nomeata/haskell-bounds-bump-action@main
   ```

2. Push this to your `main` or `master` branch.

3. In “Settings” → “Actions” → “General” → “Workflow permissions” tick
   “Allow GitHub Actions to create and approve pull requests”

To run it right away, go to “Actions” → “Create dependency bump PR” →
“Run Workflow”.

What does this do?
------------------

1. Checks out your repository
2. Uses `cabal bounds` to recognize bumpable dependencies
3. Build and tests (`cabal build && cabal test`), forcing the use of that
   dependency with `--allow-newer` and `--constraint`
4. Update the cabal file accordingly
5. Create a PR (or update an existing PR)

Why not dependabot/renovate? Why run the tests before creating the PR?
----------------------------------------------------------------------

For the typical Haskell CI setup, a PR that just bumps the dependency is not
enough:

Imagine your package `foo` depends on `bar < 1.2` and `baz`. Now
`bar-1.2` is released, and someone (or something) creates a PR against your
repository changing the version bound to `bar < 1.3`. Your CI runs the usual
set of tests, and turns 🟢. You merge he PR. All well?

No! If `baz` happens to depend on `bar < 1.2` and no new version is available
yet, your CI still silently used the old version of `bar`!

This is why this action test the compatibility wih precisely that new version.

(There may be ways to have your existing CI perform such logic, see
[this `haskell-ci` issue](https://github.com/haskell-CI/haskell-ci/issues/667).
Then the PR creation would be much simpler, and could even be delegated to
tools like dependabot or renovate.)

More questions
--------------

### CI is not run on the created pull request

This is a limitation of Github Actions: [PRs created by actions cannot trigger further
actions](https://docs.github.com/en/actions/using-workflows/triggering-a-workflow#triggering-a-workflow-from-a-workflow).

The low-tech work around is to manually close and open the PR to trigger your CI.

### My package cannot be tested that easily

The present action aims to cover the 80% of simple cases, if you have some
special requirements to run the test suite, I suggest you simply copy the steps
from this repo's `action.yml` and adjust as needed.

### I do not like the reformatting of the cabal file

I understand, but I fear there is not that much we can do about this. You could
manually edit the PR before merging, and still get some of the benefits.

### If there is a problem with whether CI tests the upper bounds, isn't there one with other versions as well?

Absolutely! See my [`cabal-plan-bounds`](https://github.com/nomeata/cabal-plan-bounds) project and the discussion on
[Haskell Discourse](https://discourse.haskell.org/t/don-t-edit-dependency-bounds-manually-with-this-ci-setup/5539).

### Why are there no releases of this action?

We just created this, and expect changes. Use `uses: nomeata/haskell-bounds-bump-action@main` if you are bold and want to help test the latest, or use a line like
`uses: nomeata/haskell-bounds-bump-action@44f853718e3cae367bd0d43372a126cd62796d80` to pin a specific revision.

### Can I help

Please do! I am certainly no expert on Github Actions.


## Contact

This action was created by Joachim Breitner <mail@joachim-breitner.de>, with input from Andreas Abel, during [MuniHac 2023](https://munihac.de/2023.html).
