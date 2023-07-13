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
         with:
           test: false
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
3. Update the cabal file accordingly
4. _Optionally_ tests these changes using `cabal build && cabal test`, forcing the use of that
   dependency through `--constraint` and `--allow-newer`
5. Create a PR (or update an existing PR)

When to run the tests, when not?
--------------------------------

For the typical Haskell CI setup, a PR that just bumps the dependency is not
enough:

Imagine your package `foo` depends on `bar < 1.2` and `baz`. Now
`bar-1.2` is released, and someone (or something) creates a PR against your
repository changing the version bound to `bar < 1.3`. Your CI runs the usual
set of tests, and turns 🟢. You merge the PR. All well?

No! If `baz` happens to depend on `bar < 1.2` and no new version is available
yet, your CI still silently used the old version of `bar`!

There are two ways of fixing this:

1. (Recommended, but harder)

   Set up your CI pipeline to always perform an “forced upper bounds check” on
   a pull request; something along these lines may work, of course you have to adjust
   the condition

       - name: Fetch cabal-force-upper-bounds
         if: matrix.plan == 'upper-bounds'
         run: |
           curl -L https://github.com/nomeata/cabal-force-upper-bound/releases/latest/download/cabal-force-upper-bound.linux.gz | gunzip  > /usr/local/bin/cabal-force-upper-bound
           chmod +x /usr/local/bin/cabal-force-upper-bound

       - name: Special handling for upper-bounds
         if: matrix.plan == 'upper-bounds'
         run: |
           echo -n "extra_flags=" >> $GITHUB_ENV
           cabal-force-upper-bound --allow-newer *.cabal >> $GITHUB_ENV

       - run: cabal build --ghc-options -Werror --project-file "ci-configs/${{ matrix.plan }}.config" ${{ env.extra_flags }}

   Maybe in the future, [`haskell-ci` will support this out of the
   box](https://github.com/haskell-CI/haskell-ci/issues/667).

   This setup is preferrable because it will test any PR, not just those
   created by this actions, but also those created by you, by contributors or
   by other dependabot/renovate like bots.

   A possible downside is that if the new dependency cannot be supported, you
   have this PR open and waiting. (I’d consider that a feature, as it is a
   reminder of an open issue.)

2. (Simpler)

   Just set `test: true` in this action's `with` clause. Then this action will
   only create PRs after the package builds and its test pass, with the new
   version forced to be used.

   This will only work in simple packages where `cabal build && cabal test`
   works without any special considerations.

   A corner case downside is that after such a bump is merged, the upper bounds
   are not necessarily tested any more by your normal CI, and you may
   accidentially break it again.

More questions
--------------

### CI is not run on the created pull request

This is a limitation of Github Actions: [PRs created by actions cannot trigger further
actions](https://docs.github.com/en/actions/using-workflows/triggering-a-workflow#triggering-a-workflow-from-a-workflow).

The low-tech work around is to manually close and open the PR to trigger your
CI. The PR description reminds you to do that.

### My package cannot be tested that easily

The present action aims to cover the 80% of simple cases. If you have some
special requirements to run the test suite, I suggest you simply copy the steps
from this repo's `action.yml` and adjust as needed, or use the recommended
workflow where your normal CI pipeline tests the upper bounds.

### I get errors about caret syntax

If you get this error
```
unexpected major bounded version syntax (caret, ^>=) used. To use this syntax
the package need to specify at least 'cabal-version: 2.0'. Alternatively, if
broader compatibility is important then use: >=0.10.4.1.2 && <0.11
expecting "." or "-"
Error: cabal: Failed parsing "/home/jojo/build/haskell/netrc/netrc.cabal".
```
make sure to use `cabal-version: 2.0` or newer in the first line of your cabal file.

### I do not like the reformatting of the cabal file

Please open an issue, maybe we can do something. But do not expect much, there
are too many possible styles around.

If you choose to use
[`cabal-plan-bounds`](https://github.com/nomeata/cabal-plan-bounds) to manage dependency version bounds for you, the syntax would be the same.

### Why bump all dependencies together?

In the happy path, this is the least noisy. Of course this can cause problem,
when one dependency can't be supported for a while, rendering this action useless.

I am not sure if there is a good alternative. Bumping packages independently
maybe? But maybe they need to be bumped together. Or try all combinations? But
that has combinatoric explosion.

### If there is a problem with whether CI tests the upper bounds, isn't there one with other versions as well?

Absolutely! See my [`cabal-plan-bounds`](https://github.com/nomeata/cabal-plan-bounds) project and the discussion on
[Haskell Discourse](https://discourse.haskell.org/t/don-t-edit-dependency-bounds-manually-with-this-ci-setup/5539).

### Why are there no releases of this action?

We just created this, and expect changes. Use `uses:
nomeata/haskell-bounds-bump-action@main` if you are bold and want to help test
the latest, or use a line like `uses:
nomeata/haskell-bounds-bump-action@44f853718e3cae367bd0d43372a126cd62796d80` to
pin a specific revision.

### Can I help

Please do! I am certainly no expert on Github Actions.

## Contact

This action was created by Joachim Breitner <mail@joachim-breitner.de>, with
input from Andreas Abel, during [MuniHac 2023](https://munihac.de/2023.html).
