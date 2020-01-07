# kubernetes-gitops-fluxcd
Continuous Delivery meets Cloud Native: [Weave Flux](https://github.com/fluxcd/flux) is the Kubernetes GitOps operator (read more about [continuous delivery with GitOps](https://www.weave.works/technologies/gitops/)), that manages deployments for you.

## Pre-requisites 

1. Kubernetes >= v1.11 (minikube, GKE, EKS)
2. Helm v3 cli, to be able to use Helm v2 and Helm v3 in parallel:
    ```bash
    OS=darwin-amd64 && \
    mkdir -p $HOME/.helm3/bin && \
    curl -sSL "https://get.helm.sh/helm-v3.0.2-${OS}.tar.gz" | tar xvz && \
    chmod +x ${OS}/helm && mv ${OS}/helm $HOME/.helm3/bin/helmv3
    ```
3. Edit `.bash_profile` or `.zshrc` file in your home directory, append:
    ```bash
    export PATH=$PATH:$HOME/.helm3/bin
    ```
4. Install Flux CD:
    ```bash
    helmv3 repo add fluxcd https://charts.fluxcd.io
    kubectl create ns fluxcd
    helmv3 upgrade -i flux fluxcd/flux --wait \
    --namespace fluxcd \
    --set git.url=git@github.com:cristinanegrean/kubernetes-gitops-fluxcd.git
    ```
 
5. Install Helm release CRD that contains helm version field:
    ```bash 
    kubectl apply -f https://raw.githubusercontent.com/fluxcd/helm-operator/master/deploy/flux-helm-release-crd.yaml
    ```

6. Install Helm Operator for Helm v3 only:
    ```bash
    helmv3 upgrade -i helm-operator fluxcd/helm-operator --wait \
       --namespace fluxcd \
       --set helm.versions=v3
    ```
   
7. Install Flux CLI:

`fluxctl` is a command-line tool that can talk to Weave Flux - it makes it very easy to manage, automate and even roll back deployments.
Below steps to install:
* macOS: `brew install fluxctl`
* Ubuntu: ```bash
          sudo apt update
          sudo apt install snapd
          sudo snap install fluxctl --classic
          ```
* Arch Linux: ```bash
              git clone https://aur.archlinux.org/fluxctl-bin.git
              cd fluxctl-bin
              makepkg -si
              ```

After installation of Weave Flux agent and `fluxctl`, simply run command `fluxctl identity --k8s-fwd-ns fluxcd` to display public key.  
Copy and add public key as deploy token to Git repo e.g GitHub. After that Weave Flux  will start watching Git repo and deploy services to the Kubernetes cluster. 
To check, tail logs: `kubectl logs flux-569589d5c4-pzxx4  -f -n fluxcd`. It should log something in the trend:
```bash
ts=2020-01-06T15:30:56.296095054Z caller=loop.go:127 component=sync-loop event=refreshed url=ssh://git@github.com/cristinanegrean/kubernetes-gitops-fluxcd.git branch=master HEAD=54e0341ae0de9e7bf2d97da8e12e207803d35b1d
ts=2020-01-06T15:30:56.672658084Z caller=daemon.go:685 component=daemon event="Sync: 54e0341, no workloads changed" logupstream=false
```
Default poll interval is set to [`5m`](https://github.com/fluxcd/helm-operator/blob/master/chart/helm-operator/values.yaml) and can be changed via updating Weave flux agent deployment, value `git.pollInterval`.
It is possible to use `fluxctl` to trigger a force sync of the Kubernetes cluster state to remote Git repo state. Just issue command: `fluxctl sync --k8s-fwd-ns fluxcd`
