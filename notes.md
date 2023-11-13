## 1) Dockerization

The phrase in the project pitch "Bonus points if only include what is needed to run the application." had me create a very simple dockerfile (it can perhaps be even simpler).

```bash
docker build -t uniwise .
```

```bash
docker run \
--env-file ./.env \
--mount type=bind,source="$(pwd)"/secrets/users.json,readonly,target=/app/secrets/users.json \
-p 4444:4444 devops-assignment
```

This creates a working and runnable image. .env and secrets/users.json needs to be loaded in manually due to .dockerignore.

## 2) CICD

I set up two CICD pipelines, one that runs the tests on every PR to the main branch from the dev branch. Github is smart enough to even add the result of the pipeline directly to the PR which can then be inspected before approving/denying merge.

The other pipeline is more "manual". It waits for a new release to be created, then creates a GHCR package in the repo, tagging it with "latest" and the release tag.

## 3) Kubernetes

I had some trouble PRIOR to beginning this, as I've been wanting to upgrade my kubernetes cluster for awhile now and saw this as an excuse to stop procastinating and actually do it, which ended up adding a lot of time to the task. It was a fun side-adventure however. As part of the upgrade, I installed Flux CD to the cluster, which I will then use to deploy the GHCR package onto the cluster.

The cluster is a small 3-node cluster I run at home as my own little sandbox environment.

I store the GitRepository manifest and kustomization in another private repo of mine, so I will just embed them in this notes.md:

```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: uniwise
  namespace: flux-system
spec:
  interval: 1m0s
  ref:
    branch: dev
  url: https://github.com/purvaldur/uniwise-devops-assignment
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: uniwise
  namespace: flux-system
spec:
  interval: 5m0s
  path: ./flux
  prune: true
  retryInterval: 2m0s
  sourceRef:
    kind: GitRepository
    name: uniwise
  targetNamespace: uniwise
  timeout: 3m0s
  wait: true
```

Before doing anything else, I created the ns.yaml to create the namespace.

The application was then deployed to the cluster using a deployment manifest (deploy.yaml) along with a secret for the users.json that was mounted as a file within the deployment (secret.yaml).

## 5) Expose out of cluster

I did #5 before #4 because that was an easy, next step. Using a combination of a service (defined in svc.yaml) and nginx ingress together with letsencrypt certmanager, I was able to quickly expose the application with HTTPS included.

The application is currently live on https://uniwise.vald.io/ - I am willing to take the URL down immediately upon request if the presence of the word "uniwise" in the URL is inappropriate.

## 4) Resilience

