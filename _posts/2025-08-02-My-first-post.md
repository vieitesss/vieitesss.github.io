---
title: ArgoCD & SOPS
date: 2025-08-02 21:10:55 +0200
categories: [DevOps]
tags: [ArgoCD]
---


Do you want to public your secrets in a secure way? Are you trying to make Argo read this secrets and decrypt them in order to deploy your application?

After four days fighting to implement this feature, I finally did it. I have never worked with ArgoCD, and I have learnt a lot by doing this. It was a pain for me to search and understand why this works, so I want to make it easy for you.

## The problem

TL;DR
We need to store our Kubernetes Secrets in a secure way inside our state repository, this is, encrypted, and tell Argo how to get them, decrypt them and deploy everything.

---

I am finishig my Bachelor's Thesis. What I am trying to do is to create a *full CI/CD workflow with Dagger* for a sample application. The primary objective is to demonstrate and analize the advantages of using it, compared to traditional implementation methodologies.

- *What is Dagger?* - You can check [this post](https://www.linkedin.com/feed/update/urn:li:activity:7343216392298528771/) if you want to know about Dagger.

As a crucial part of an application's life, *Continuous Deployment* plays a vital role. Its goal is to ensure that the user always have access to the latest stable version of the application. This allows the development team to gather feedback quickly and resolve any error as fast as possible.

One popular tool for this is ArgoCD

- *What is ArgoCD?* - It is a software solution for managing deployments that was created specifically for Kubernetes.

- *How does Argo work?* - Argo follows the GitOps principles, which include *storing the desired state of the deployment in a Git repository* and using automation to keep the system in sync with that state. This desired state is defined by some YAML files, containing *all* the information needed to deploy the application. That information may include, in the mayority of cases, *passwords and important data* that should not be public on the Internet.

So, we need to *encrypt* this kind of data, also know as *Secrets*, in a repository, and tell Argo how to get and *decrypt* them. Furthermore, it has to deploy every other kind of Kubernetes object, such as *Deployment*, *Ingress*, *ConfigMap*, etc.

## The solution

### age and SOPS

The first thing we need to do is to encrypt our secrets. To achieve this, we are going to use a modern encryption tool known as age (pronounced as the italian word "*aghe*").

Follow [the installation process](https://github.com/FiloSottile/age?tab=readme-ov-file#installation) for your environment.

Once we have installed age, we can generate a public and a private key with the command below:

```bash
age-keygen -o age.agekey
```

The previous command will return two things:
- The public key to the standard output.
- A file named `age.agekey`, with the private key and a comment with the public key, with the creation date in another comment.

We are not encrypting directly with age, we are using SOPS (Secret OPerationS), that integrates with it. SOPS is self-described as an "editor of encrypted files". In essence, this means that *SOPS can encrypt specific values within a file instead of encrypting it entirely*. Its primary use case, therefore, is to secure sensitive data, such as passwords, within files like YAML or JSON. This approach *keeps the file readable in terms of structure, while preventing the leakage of confidential information*.

We can configure sops to use our generated key with a `.sops.yaml` file like this:

```yaml
creation_rules:
    # encrypt every .yaml file found
  - path_regex: ".*\\.ya?ml$"
    # do not encrypt this values
    unencrypted_regex: "^(apiVersion|metadata|kind|type)$"
    # your public key
    age: age15peyc7pedj8...
```

With the previous file located at our current working directory, we can encrypt a YAML file with the following command:

```bash
sops --encrypt --in-place secret.yaml
```

Now we have our secret encrypted, ready to be pushed to our state repository.

### ksops

To let Argo decrypt our secrets, we have to tell him how to do it. Hence, we need to use a kustomize plugin called ksops.

- *What is kustomize?* - It is a software that lets you customize the YAML definition of a resource without touching the original file.

- *How does kustomize work?* - You define a `kustomization.yaml` file in a working directory relative to the definitions you want to kustomize. This file describes the changes you want to make.

For this example we will create the following `kustomization.yaml` file:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
# resources:
#   - non-secrets.yaml
generators:
  - secret_generator.yaml
```

- *What is a generator?* - It describes what *new* resources should be created. Instead of creating a resource directly, this file will generate one or more Kubernetes resources when Kustomize runs.

Our `secret_generator.yaml` will look like this:

```yaml
apiVersion: viaduct.ai/v1
kind: ksops
metadata:
  name: secret-generator
  annotations:
    config.kubernetes.io/function: |
      exec:
        path: ksops
files:
  - secrets.yaml # The name of the encrypted files
```

The previous file tells Kustomize to use the ksops tool.
- Identifies the file as a `ksops` Kustomize plugin.
- `config.kubernetes.io/function`: This special annotation tells Kustomize that to process this file, it must execute an external program. In this case, the `ksops` binary. *It must be in the `$PATH`* in order to be defined like this.
- The `files` section defines the input for the `ksops` command, telling it to find and decrypt them.

Both files must be *placed in the same or in a relative directory from the secrets definition or other resources*.

### KinD and private key

Now we need a cluster with Argo installed. To achieve this we will use KinD (Kubernetes in Docker). We can create a cluster with the following command:

```bash
kind create cluster --wait 5m --name example
```

The next step is to create a namespace where Argo will be deployed.

```bash
kubectl create namespace argocd
```

The following step consists in providing the private key to the cluster. This can be done with the next command:

```bash
cat "./age.agekey" |
    kubectl create secret generic sops-age \
    -n argocd --from-file=keys.txt=/dev/stdin
```

This command creates a generic secret with name `sops-age`, that will be added to the `argocd` namespace. This secret will have an entry data with the key `keys.txt` and a value equal to the content of the `age.agekey` file.

### Argo

Once we have added the secret to the cluster, we are ready to install Argo in it. We are going to use the Argo Helm chart, so we first have to install the repository.

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

But, before installing the chart, we need to define some values in order to tell Argo *where* to get our deployment files from. This is our `values.yaml` definition:

```yaml
configs:
  cm:
    kustomize.buildOptions: "--enable-alpha-plugins --enable-exec"

repoServer:
  env:
    - name: XDG_CONFIG_HOME
      value: /.config
    - name: SOPS_AGE_KEY_FILE
      value: /.config/sops/age/keys.txt

  volumes:
    - name: custom-tools
      emptyDir: {}
    - name: sops-age
      secret:
        secretName: sops-age

  initContainers:
    - name: install-ksops
      image: viaductoss/ksops:v4
      command: ["/bin/sh", "-c"]
      args:
        - echo "Installing KSOPS and Kustomize...";
          mv ksops /custom-tools/;
          mv kustomize /custom-tools/;
          echo "Done.";
      volumeMounts:
        - mountPath: /custom-tools
          name: custom-tools

  volumeMounts:
    - mountPath: /usr/local/bin/kustomize
      name: custom-tools
      subPath: kustomize
    - mountPath: /usr/local/bin/ksops
      name: custom-tools
      subPath: ksops
    - name: sops-age
      mountPath: /.config/sops/age
```

Let's go through each of the main blocks of configuration.

- `configs`: Here we are modifying the ConfigMap map, adding some needed arguments to the `kustomize` command. This will allow `kustomize` to execute the `ksops` binary.
- `repoServer`: We define two environment variables.
    - `XDG_CONFIG_HOME` to set the base directory relative to which user-specific configuration files should be written.
    - `SOPS_AGE_KEY_FILE` to tell where the private key will be located.
- `volumes`: We define two volumes.
    - `custom-tools` will store both `ksops` and `kustomize` binaries.
    - `sops-age` is our previously added secret with the age private key data.
- `initContainers`: This container will be run before the installation of Argo. Its purpose is to install the `kustomize` and `ksops` binaries from an image that have them already installed (`viaductoss/ksops:v4`). It takes the binaries and copies them inside our `custom-tools` volume, that is mounted in the path of the initContainer to make it accessible inside of it.
- `volumeMounts`: Finally, we mount everything we need to have access to in the Argo release. We take both installed binaries from our `custom-tools` volume and make them available in the `$PATH`. And we save the private key inside its default configurations directory.

Now it is the moment to release the Argo chart.

```bash
helm install argocd argo/argo-cd \
    -n argocd \
    -f values.yaml \
    --wait \
    --version 6.11.1
```

But this is not everything, now we need to tell Argo where to get our resource definitions from. I have created a repository called `state` with a `deploy` branch where I store the files related and need to deploy: `secrets.yaml`, `kustomization.yaml` and `secrets_generator.yaml` (and `non-secrets.yaml` if needed).

Here we have a sample file `argo_dev.yaml` to tell Argo where to find the files related to the deployment into the "dev" environment.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/<USER>/state.git'
    path: dev # <- relative path from the root (the "./dev" dir)
    targetRevision: deploy # <- the branch
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

We only need to apply the file with the following command.

```bash
kubectl apply -f "/argo_dev.yaml"
```

## Deployed

After all the previous steps, we have finally deployed Argo with the ability to decrypt encrypted secrets from our repository! This lets us push any secret to a public repository without the fear of leaking confidential information.

If you have followed the steps, you should get your Secret resources inside Argo.

# Conclusion

This is not an straightforward way to achieve this, but it is how I made it work. Overall, once it is set up, it is very satisfying and relieving to know that your secrets are safely stored.

Thanks for reading so far!

Share it and leave a like if you found it useful.
