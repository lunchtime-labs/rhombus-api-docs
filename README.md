# Rhombus Documentation

This repository holds the documentation for the [Rhombus][rhombus] project.

The production deploy can be found at:
http://docs.rhombus.network/.

The staging deploy can be found at:
https://lunchtime-labs.github.io/rhombus-api-docs

This repository is a fork of the API Generator [Slate][slate]. It should be
rebased off of the head of the upstream repository.

## Getting Started

For instructions on getting up and running, please refer to the README
documentation for [Slate][slate].

## Updating

Add the upstream as a git remote:

```
git remote add upstream git@github.com:lord/slate.git
```

Then rebase off the head of the upstream master.

```
git pull upstream master --rebase
```

## Deploying

### Staging

Add the remote for staging:

```
git remote add staging git@github.com:lunchtime-labs/rhombus-api-docs.git
```

Deploy to staging:

```
./deploy.sh -r staging
```

### Staging

Add the remote for production:

```
git remote add production git@github.com:RhombusNetwork/rhombus-docs.git
```

Deploy to production:

```
./deploy.sh -r production
```

[rhombus]: https://rhombus.network
[slate]: https://github.com/lord/slate
