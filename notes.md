## Preface:
Some of these notes are written before I actually did what I wrote about, and some are written after-the-fact. As a consequence of this, it's a bit of a mish-mash of past, present and future tense.

Also, by the very nature of having my own kubernetes cluster at home already, there are some things I may have skipped over. For example already having done a lot of the legwork in setting up a cluster with StorageClass, Ingress, Flux, etc etc...

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

*the section was returned to later to add the following*

I have decided to also experiment with Flux image scanning as part of the CICD process.

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

I did #5 before #4 because that was an easy, next step. Using a combination of a service (defined in svc.yaml) and Nginx ingress + Letsencrypt cert-manager, I was able to quickly expose the application with HTTPS included.

The application is currently live on https://uniwise.vald.io/ - I am willing to take the URL down immediately upon request if the presence of the word "uniwise" in the URL is inappropriate.

## 4) Resilience

It seems the application is already set up to use Redis as the backend, so it should be trivial to set up redis on the side and let the application connect to it through the provided REDIS_URL environment variable. That way, if the pod goes down or gets rolled, data is not lost. Also, it seems a redis instance is *required* because if not provided, the application throws an error in the console log about it when the application is interacted with...

I don't have a redis instance set up already, so this will be a fun exercise to set this up using helm+flux. I'll use BitNami for the helm chart, since I have experience with their helm charts and they do good work.

---

That was pretty neat. The bitname helm chart made setup a breeze. I had to disable authentication since the application doesn't seem to have any authentication capabilites built into it. Not quite best practice, but in the real world, this would solved in collaboration with the application developer.