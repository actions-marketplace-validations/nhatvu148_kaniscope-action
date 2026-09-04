# Releasing

`action.yml` pins the container by a **floating image tag**:

```yaml
image: 'docker://ghcr.io/nhatvu148/kaniscope-action:v1'
```

So what a consumer on `@v1` actually runs comes from the **GHCR image**, not from
the git tag. That single fact is what makes the order below matter.

## The order, and why

```sh
# 1. Annotated tag on main. Its body becomes the release notes verbatim —
#    release.yml reads `git tag -l --format='%(contents)'`.
git tag -a v0.1.11 -m "v0.1.11

Whatever the release notes should say, in markdown."

# 2. Push it. This triggers BOTH publish.yml (builds and pushes the image,
#    including the floating :v1 image tag) and release.yml (creates the release).
git push origin v0.1.11

# 3. WAIT for "Publish image" to go green. Do not skip this.

# 4. Only now move the floating git tag.
git tag -f v1 main
git push --force origin v1
```

## The failure this prevents

Moving `v1` first — or moving it *alone* — leaves consumers pulling the previous
image. Everything looks correct: the git tag points at the new commit, the diff
is right, CI is green, and `@v1` still runs the old code. Nothing fails, so
nothing tells you.

That is not hypothetical. Between 2026-08-18 and 2026-09-04 the `:v1` image was
eight `pr-review-core` minor versions behind `main`, because releases stopped
being cut while `main` moved on. Consumers on `@v1` got the August build the
whole time.

## Checks worth doing

`consumer check (published @v1)` is the one that matters — it consumes the action
the way a real user does, through the published tag, so it exercises the image
rather than the working tree. Green there means a consumer gets what you think
they get.

To confirm the image really moved:

```sh
gh api /users/nhatvu148/packages/container/kaniscope-action/versions \
  --jq '[.[] | select(.metadata.container.tags[]? == "v1")] | .[0]
        | "\(.updated_at)  \(.metadata.container.tags)"'
```

The timestamp should be from the release you just cut, and the tag list should
carry both `v1` and the new `vX.Y.Z`.

## Don't

- **Don't `workflow_dispatch` publish.yml from an untagged branch to "refresh"
  `:v1`.** The `enable` guard in publish.yml exists to stop exactly that — it
  would move the floating image tag onto an untagged build, and then no git tag
  describes what consumers are running.
- **Don't move `v1` to a commit that has not been published.** That is the
  failure above, in one step.
