---
title: Managing Helm secret values in ArgoCD
categories:
  - blog
tags:
  - technical
  - kubernetes
  - devops
  - gitops
  - argocd
  - helm
---

Recently I have been reading about GitOps and during the last week, I decided to give it a shot.

I have a self-hosted setup with Kubernetes (k3s) and a bunch of tools running on it. 
My goal was to migrate my apps which are installed via Helm to ArgoCD.

Everything went well until I hit the problem with the secret management. Here's how I solved it.

## The Problem

ArgoCD uses Kubernetes itself as its storage backend - an application is represented as a custom 
resource called an _application_ (`applications.argoproj.io`).

To implement a fully git-driven development, there is a commonly used pattern called
"the app of apps". In this pattern, we create 1 parent app that is a Helm chart directory in our git repo. 
Then we put all of our `application` resource manifests into the `templates/` directory of this chart.

Here's the problem: The `application` resources hold all the helm values that are passed to the app installation.
Some of these values are sensitive such as DB passwords, private keys, and so on. Here's an example:
[the admin password in the PostgreSQL chart](https://github.com/bitnami/charts/tree/master/bitnami/postgresql#postgresql-parameters).

We must not commit them into the code but at the same time implement a fully git-driven workflow.

There is an [open issue](https://github.com/argoproj/argo-cd/issues/1786) for ArgoCD that is unresolved since
more than 2 years.

## In the past

This was not a problem in a traditional `helm install/upgrade` based workflow because it's not fully
git-driven - we could put these sensitive values into our CI/CD tool and pass them via `--set` or `--values`
flags on install/upgrade. 

In my personal setup, I was using 
[Ansible Helm module](https://docs.ansible.com/ansible/latest/collections/kubernetes/core/helm_module.html) 
to install the charts and was using 
[ansible-vault](https://docs.ansible.com/ansible/latest/user_guide/vault.html) 
to encrypt the sensitive variables and check them into the code.

Unfortunately, these solutions are not applicable as-is in the fully git-driven ArgoCD setup.

## The existing solutions (and the problem with them)

ArgoCD have [decided](https://argoproj.github.io/argo-cd/operator-manual/secret-management/) 
to stay agnostic to the secret management methods. 
Looking at the [open issue](https://github.com/argoproj/argo-cd/issues/1786) on the topic,
seeing it is the biggest problem for its adoption for so many people, 
I don't think this is a good decision.

There are 2 common approaches to work around this problem:

#### Solution A: Using [sealed-secrets](https://github.com/bitnami-labs/sealed-secrets)
Sealed Secrets is a great tool to manage Kubernetes secrets securely.
However, it has a shortcoming in our use-case: it doesn't offer a solution to encrypt helm values.
Sealed secrets can be used **only if** the chart has an option to point to an 
external secret (a value like `existingSecret`).

Some charts give this option but not all of them do. 
Especially if you are using many 3rd party charts as I do, 
sealed-secrets won't be enough to solve your problem.

#### Solution B: Using [helm-secrets](https://github.com/jkroepke/helm-secrets)

At first glance, helm-secrets seems like the perfect tool for the problem - it uses PGP to
encrypt and decrypt the helm values in place.

The problem is its integration to ArgoCD - it is nowhere near trivial to integrate it into ArgoCD.
It involves rebuilding the ArgoCD container image to include gpg, sops binaries, helm secrets plugin
and even worse, implementing a wrapper script for the Helm binary so that it wraps the helm
commands, so that `helm upgrade app` becomes `helm secrets upgrade app`.

This solution is described [in this blog post](https://hackernoon.com/how-to-handle-kubernetes-secrets-with-argocd-and-sops-r92d3wt1).

## An alternative solution: Leveraging volumeMounts

Because of these issues with both of these solutions, I started looking into alternatives.

In this approach, we leverage Sealed Secrets together with the `valueFiles` feature of 
helm-based ArgoCD applications. In short, in the ArgoCD installation, we mount a `Secret` 
(sourced from a `SealedSecret`) in the `optional` mode with the secret helm values in it. 
Each entry in the secret becomes available as a file in the ArgoCD application.

The steps are the following:

1. Install/upgrade your ArgoCD with an **optional** secret mount on its repo-server component. 
   Let's call the secret `helm-values` and mount it to the `/helm-values` path in container:
   Your values file:
   ```yaml
   # ...
   repoServer:
     volumeMounts:
       - name: helm-values
         mountPath: /helm-values
     volumes:
       - name: helm-values
         secret:
           secretName: helm-values
           optional: true
   # ...
   ```
   Install it via 
   ```bash
   helm install argocd --values <VALUES_ABOVE>.yaml
   ```
   
**Hint:** We define the volume as `optional: true` because we want ArgoCD to be able to run even when
the secret does not exist yet. By doing this we avoid the chicken and egg problem.
{: .notice--info}

2. We will manage sealed-secrets using ArgoCD as well. We need to make sure 
   that we set up sealed-secrets in a fully deterministic, repeatable manner. In other words, 
   [bring your own certificates](https://github.com/bitnami-labs/sealed-secrets/blob/main/docs/bring-your-own-certificates.md) to it.

3. Locally, create a secret called `helm-values` with one entry named `<app_name>.yaml` for each 
   application you deploy. An example would be:
   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: helm-values
     namespace: argocd
   stringData:
     postgres.yaml: |
       postgresqlPassword: SeCrEtPassword!
     grafana.yaml: |
       adminPassword: sEcReTPW$!_
   ```
   
   Then, seal it with your certificate:
   ```bash
   kubeseal --cert "./${YOURPUBLICKEY}" < helm-values.yaml > helm-values-sealed.yaml
   ```

4. Put this sealed secret `helm-values-sealed.yaml` into your app of apps 
   along with your sealed-secrets and other application manifests.


**Hint:** make sure to use [sync-waves](https://argoproj.github.io/argo-cd/user-guide/sync-waves/) 
to create the sealed-secrets controller before the `SealedSecret` resources.
{: .notice--info}

5. Edit your app manifests to point to their secrets mounted into the container. Here's an example:
   ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: grafana
      namespace: argocd
    spec:
      destination:
        server: https://kubernetes.default.svc
      project: default
      source:
        chart: grafana
        helm:
          values: |
            service:
              type: LoadBalancer
          valueFiles:
            - /helm-values/grafana.yaml
        repoURL: https://grafana.github.io/helm-charts
        targetRevision: 6.16.2
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
   ```

That's it. You can now safely commit all of your resources including the secret values
into your git repository.

When you need to add a new secret you will have to unseal/edit/reseal this secret.
When you edit it, the changes will reflect into the repo-server without requiring a restart.

You can check out the [sample project](https://github.com/utkuozdemir/argocd-helm-secret-values-example) I created to demonstrate the steps above.

## Pros and Cons

This solution has the following **advantages** compared to other alternatives:
- Requires very little change in the charts and chart release configuration
- Doesn't require a restart
- Doesn't require intervention after the initial setup
- No need to build your custom image/wrapper

Still, it **does not solve all problems**, such as:
- All secret values of all apps are kept together in 1 secret
- The secret has to reside in the same namespace as ArgoCD
- Requires modification in the ArgoCD deployment
- It is still a workaround after all, not a full-fledged solution

## Conclusion

Secret management is still a problematic topic in the Kubernetes ecosystem - there are many
approaches but none of them has become the defacto standard yet.

Probably because of that ArgoCD does not offer a clean solution for the secrets passed as helm values.

Until the issue is addressed, we can solve the problem using an 
[indirection](https://en.wikipedia.org/wiki/Fundamental_theorem_of_software_engineering).

Check out the sample project I created **[here](https://github.com/utkuozdemir/argocd-helm-secret-values-example)**.
