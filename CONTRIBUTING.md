# Contributing

## Developer Guide

### Local Development

To spin up a local development environment with locally built artifacts, run:

```
make build local-dev
```

You can override default local development configuration by overriding environment
variables [here](https://github.com/upbound/universal-crossplane/blob/main/cluster/local/config/config.env).

For example, the following will enable connecting your local development environment to Upbound Cloud:

```
export LOCALDEV_CONNECT_CP_ORG=<YOUR_UBC_ORG>
export LOCALDEV_CONNECT_API_TOKEN=<YOUR_ACCESS_TOKEN>

make build local-dev
```

### Cleanup

To clean up local dev environment, first delete self hosted control plane (if connected) from Upbound Cloud Console
and then run:

```
make local.down
```

### Validation

To run validation tests locally, run:

```
make e2e.run
```

To run validation test including Upbound Cloud connectivity, run:

```
export LOCALDEV_CONNECT_CP_ORG=<YOUR_UBC_ORG>
export LOCALDEV_CONNECT_API_TOKEN=<YOUR_ACCESS_TOKEN>

make e2e.run
```

## Release Process

### Crossplane fork sync:

To update Crossplane version in UXP follow the steps below:

#### Prepare repos and forks

All the steps below will assume you have forked the following repos with the following names:

- upstream Crossplane: `crossplane/crossplane` -> `$MY_GITHUB_USER/crossplane`
- Upbound Crossplane's fork `upbound/crossplane` -> `$MY_GITHUB_USER/upbound-crossplane`

Once you have created them, you'll need to setup your local environment, if you
already did it in the past just skip to the next part, otherwise run the
following commands, taking care to set your GitHub user instead of
`<MY_GITHUB_USER>`:

```shell
export MY_GITHUB_USER=<MY_GITHUB_USER>

mkdir sync-upbound-crossplane
cd sync-upbound-crossplane
git clone https://github.com/$MY_GITHUB_USER/crossplane
cd crossplane
git remote add upstream https://github.com/crossplane/crossplane
git remote add upbound-upstream https://github.com/upbound/crossplane
git remote add upbound-origin https://github.com/$MY_GITHUB_USER/upbound-crossplane
git fetch --all
git submodule update --init
```

Now based on the task at hand, pick and go through one of the task below:

##### Sync **a new** release branch:

If you are releasing a new minor of UXP, e.g. `vX.Y.0-up.1`, and you want to
create a new release branch `release-X.Y` on `upbound/crossplane` based on the
upstream `crossplane/crossplane` release branch.

```shell
RELEASE_BRANCH=release-X.Y

git fetch --all
git checkout -b $RELEASE_BRANCH upstream/$RELEASE_BRANCH

# Cherry-pick upbound/crossplane specific changes.
# makefile, workflow and readme updates
git cherry-pick -x 5d53beb3cb423b13960a92d1f8f9284c9a146ccc # https://github.com/upbound/crossplane/commit/5d53beb3cb423b13960a92d1f8f9284c9a146ccc
# docs publishing and codeowners changes
git cherry-pick -x 85027abd2449fa69cbd825faa3fc68f4c64bb36d # https://github.com/upbound/crossplane/commit/85027abd2449fa69cbd825faa3fc68f4c64bb36d
# proidc support
git cherry-pick -x e9d31c76752e86b8cbf99b9651b1a38e8c9d7b7f # https://github.com/upbound/crossplane/commit/e9d31c76752e86b8cbf99b9651b1a38e8c9d7b7f

git push upbound-upstream $RELEASE_BRANCH
```

Once the branch has been created, the branch protection rules will require you to go through PRs to update it, so see the following section to do so.

##### Sync **an existing** release branch:

If you are releasing a new patch of UXP, e.g. `vX.Y.Z-up.1`, and you want to
open a PR to update `release-X.Y` on `upbound/crossplane` with the latest
changes up to `vX.Y.Z` of upstream `crossplane/crossplane`.

```shell
RELEASE_BRANCH=release-X.Y
UPSTREAM_RELEASE_TAG=vX.Y.Z

git checkout -b sync-upstream-$RELEASE_BRANCH
git reset --hard upbound-upstream/$RELEASE_BRANCH
git fetch --tags upstream
git merge $UPSTREAM_RELEASE_TAG

# Resolve conflicts, if any, and then push to your own fork
git push --set-upstream upbound-origin sync-upstream-$RELEASE_BRANCH
```

You can then create a PR from your fork's `sync-upstream-X.Y` branch to
`upbound/crossplane`'s `release-X.Y` branch and get it reviewed and merged.

##### Sync latest master:

This step is not required at the moment, but if you want to sync
`upbound/crossplane`'s master branch to the latest `crossplane/crossplane`
branch, run the following commands and open a PR from your fork's `sync-upstream-master` branch to
`upbound/crossplane`'s `master` branch and get it reviewed and merged.

```shell
git fetch --all
git checkout -b sync-upstream-master
git reset --hard upbound-upstream/master
git merge upstream/master
# Resolve conflicts, if any
git push --set-upstream upbound-origin sync-upstream-master
```

## Deprecated or on-demand release channels

New UXP releases can be also published to the following release channels, but
for the time being we are not proactively doing so. Below sections should only
be followed if explicitly requested:

### Operator Lifecycle Manager

OLM uses its own packaging format for its marketplaces.

1. **download bundle**: Download the bundle from the following link:
   ```
   VERSION=<release version> # including v prefix, i.e. v1.3.3-up.1
   curl -sL https://releases.upbound.io/universal-crossplane/stable/$VERSION/olm/$VERSION.tar.gz  | tar xz
   ```
1. **open pull requests**: Extract the content into a new folder named with version **without v prefix**:
   and open the PRs to the following directories:
    * https://github.com/redhat-openshift-ecosystem/community-operators-prod/tree/main/operators/universal-crossplane
    * https://github.com/k8s-operatorhub/community-operators/tree/main/operators/universal-crossplane
1. **sanity check**: Once the PRs are merged and CI in main branch completes the
   propagation, you should see the new version in [OperatorHub product page](https://operatorhub.io/operator/universal-crossplane).

See [this page](cluster/olm/README.md) for details.

### AWS Marketplace

AWS Marketplace installations are done using usual Helm commands but the images
need to exist in given ECR repositories. Currently, due to technical limitations,
it is not possible to define IAM policies on the allocated marketplace ECR repository
so you need to push the images there manually.

1. **list images**: In order to see what images are used in a specific release,
   you can examine the Helm chart `values.yaml` file:
   ```bash
   VERSION=<released-version> # excluding v prefix, i.e. 1.3.3-up.1
   curl -sL https://charts.upbound.io/stable/universal-crossplane-$VERSION.tgz | tar xz
   ```
1. **login to ECR**: You need to have an AWS user in production marketplace account
   and use the following command to login your docker client:
   ```bash
   # Note that the account ID is the one used by all products in the marketplace, not our
   # account ID.
   aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin "709825985650.dkr.ecr.us-east-1.amazonaws.com"
   ```
1. **tag & push**: Tag and push every image like the following:
   ```bash
   # You can check the full repository URLs from Repositories list of the product
   # in AWS Marketplace Management Portal
   docker tag upbound/crossplane:<tag-in-values.yaml> "709825985650.dkr.ecr.us-east-1.amazonaws.com/upbound/crossplane:<tag-in-values.yaml>"
   ```
   Note that all images, including the ones under crossplane DockerHub organization,
   need to be re-tagged pushed to ECR.
1. **add a new version**: Login to [AWS Marketplace](https://aws.amazon.com/marketplace/)
   and add a new version using `Request changes` menu in the UI. It will
   pre-populate most of the metadata from the last release, make sure all
   versions are updated.
1. **release version**: Using the same `Request changes`, release the version you
   created.
1. **sanity check**: After change requests are completed, visit [the product page](https://aws.amazon.com/marketplace/pp/prodview-uhc2iwi5xysoc)
   to make sure the new version is there.

### Rancher Marketplace

UXP is available as a partner chart in the Rancher Marketplace. Rancher partner charts are served from the
Rancher's [partner-charts repository](https://github.com/rancher/partner-charts). To get UXP chart updated,
we need to open a PR against the [main-source branch](https://github.com/rancher/partner-charts/tree/main-source) there.

To prepare the PR, we need to follow the workflow steps listed [here](https://github.com/rancher/partner-charts/tree/main-source#workflow).
Due to the reasons outlined in [this issue](https://github.com/upbound/universal-crossplane/issues/119), we need an
additional change in `Chart.yaml` where we convert UXP version from `x.y.z-up.t` to `x.y.z00t` in the [make changes step](https://github.com/rancher/partner-charts/tree/main-source#4-make-changes).
See [this](https://github.com/rancher/partner-charts/pull/89#discussion_r640533267) as an example.
