
# Deploy Apps using ArgoCD via Helm Charts(GitOps) 

## Prerequisites
- ArgoCD
- A git Repo
- Kubernetes Cluster :)

## Summary
In this repository you will find how to implement Git centric deployments to your Kubernetes(as well as Openshift) clusters with ArgoCD. 
- ArgoCD holds the information about the repo's it watches with Application Manifest file, like in [bgd-app](https://raw.githubusercontent.com/redhat-developer-demos/openshift-gitops-examples/main/components/applications/bgd-app.yaml) example.
- We can add Application YAML files to the repo ArgoCD watches. You will create one Git Repo and add it to your ArgoCD manually only once, and then putting Application Manifest files to repo will create chained creation of **Namespace** and **Deployments**. The main idea is this in this explanation.
- That's where we need Helm most. Because of creation of **namespaces** and **deployments** are expected to be the same mostly, I have created two Helm Chart for them.
- ArgoCD Application Manifest File for creating **Namespace**'s and **Deployment**'s are in the templates folder.
	- [deployment-template.yaml] - [namespaces-template.yaml]
- You can find the Helm Chart's used in this repo: https://github.com/burhanuguz/helm-chart-gitops/

It might be confusing at start but once you get the idea, your deployments and namespaces creations will only need git push at the end :)


The idea is, you will have a repository that holds all information about *namespaces*, as well as *applications* like in below. *namespace*, and *app* YAML's will be Application Manifest File that references Helm Chart with custom Chart values.
``` nginx
📦cluster-named-repository
#┣ 📜any-cluster-resources.yaml     # Custom resource file to apply cluster-wide usually, can add as much as you need
 ┣ 📜cluster-namespaces.yaml        # ArgoCD Application Manifest File that references the namespaces folder
 ┗ 📂namespaces                     # Create this folder manually, put .gitignore file
    ┣ 📜namespace-1.yaml            # ArgoCD Application Manifest File that references the namespaces-1 folder
    ┣ 📜namespace-2.yaml            # ArgoCD Application Manifest File that references the namespaces-2 folder
    ┗ 📂namespace-1                 # Create this folder manually, put .gitignore file
      ┣ 📂app-1                     # Create this folder manually, put .gitignore file to avoid false positive  alerts that can come on ArgoCD
#       ┗ 📜app-resource.yaml       # Custom resource file to apply for app-1(e.x configmap), can add as much as you need
      ┣ 📂app-3                     # Create this folder manually, put .gitignore file
#       ┗ 📜app-resource.yaml       # Custom resource file to apply for app-3(e.x configmap), can add as much as you need
      ┣ 📜app-1.yaml                # ArgoCD Application Manifest File that references the app-1 folder
      ┗ 📜app-3.yaml                # ArgoCD Application Manifest File that references the app-3 folder
#     ┗ 📜namespace-resource.yaml   # Custom resource file to apply for namespaces-1, can add as much as you need
    ┗ 📂namespace-2                 # Create this folder manually, put .gitignore file
      ┣ 📂app-2                     # Create this folder manually, put .gitignore file
#       ┗ 📜app-resource.yaml       # Custom resource file to apply for app-2(e.x configmap), can add as much as you need
      ┣ 📂app-4                     # Create this folder manually, put .gitignore file
#       ┗ 📜app-resource.yaml       # Custom resource file to apply for app-4(e.x configmap), can add as much as you need
      ┣ 📜app-2.yaml                # ArgoCD Application Manifest File that references the app-2 folder
      ┗ 📜app-4.yaml                # ArgoCD Application Manifest File that references the app-4 folder
#     ┗ 📜namespace-resource.yaml   # Custom resource file to apply for namespaces-2, can add as much as you need
```

There is no automation explained here to automate git push. You would need to implement your own solution for it with the tools you have.


### [*new-namespace* Chart](https://github.com/burhanuguz/helm-chart-gitops/tree/master/charts/new-namespace)
Simply pushing a YAML to the repo will create **namespace** with **limit range**, **resource quoata**.
In case you have Conjur in your cluster, you can create conjur connection **ConfigMap** as well. You can add fork and add more customisation to the Helm Chart. 

After creating namespaces folder, you can add your namespace with template file here, and change variables inside it. I am using **envsubst** for this example, and push the changes to the repository after the changes are done.

```bash
# First get the values YAML for the helm chart, and fill the values with your choice
wget https://raw.githubusercontent.com/burhanuguz/helm-chart-gitops/master/charts/new-namespace/values.yaml
# Edit values.yaml
vi values.yaml # Adjust Limit Range and Resource Quota and edit Conjur values if you have Conjur
# Prepare variables to substitute in template
export CLUSTERNAME="test-cluster"
export GITREPO="https://<your-repo>/${CLUSTERNAME}.git"
export CHARTURL="https://burhanuguz.github.io/helm-chart-gitops/"
export CHARTNAME="new-namespace"
export CHARTVERSION="0.1.1"
export NAMESPACE="test-1"
export ARGOAPPNAME="${NAMESPACE}"
export fullnameOverride="${NAMESPACE}"
export ARGOCDNAMESPACE="openshift-gitops"
export valuesYAML=$(echo '|'; awk 'BEGIN{s=sprintf("%-8s", "");}{print s $0}' ../values.yaml)

# Clone repo and enter the folder of git repo
git clone "${GITREPO}" && cd "${CLUSTERNAME}"

# Create namespace yaml
mkdir namespaces/${NAMESPACE}
touch namespaces/${NAMESPACE}/.gitignore
envsubst < templates/applicationManifestTemplate.yaml > namespaces/${NAMESPACE}.yaml

git add --all; git commit -m "${NAMESPACE}"; git push origin master
```
### [*deploypackage* Chart](https://github.com/burhanuguz/helm-chart-gitops/tree/master/charts/deploypackage)
Pushing a YAML to the repo will create **deployment**, **service**. And if enabled **ingress**, **route**(for Openshift) and **serviceaccount**. It is very customizable deployment that can include sidecar, init containers and lots of definitions you can do with values YAML.

In case you have Conjur in your cluster, you can edit values.yaml to enable sidecar or init container for Conjur. You can add fork and add more customisation to the Helm Chart. 

After creating application folder in the namespace, you can add application YAML file with template here, and change variables inside it. I am using **envsubst** for this example, and push the changes to the repository after the changes are done.

Almost everything is the same as editing template for namespace here. Only ARGOAPPNAME changes

```bash
# First get the values YAML for the helm chart, and fill the values with your choice
wget https://raw.githubusercontent.com/burhanuguz/helm-chart-gitops/master/charts/deploypackage/values.yaml
# Edit values.yaml
vi values.yaml # Adjust Deployment, Route, Ingress, Service and related entries. Also edit Conjur values if you have Conjur

# Prepare variables to substitute in template
export CLUSTERNAME="test-cluster"
export GITREPO="https://<your-repo>/${CLUSTERNAME}.git"
export CHARTURL="https://burhanuguz.github.io/helm-chart-gitops/"
export CHARTNAME="deploypackage"
export CHARTVERSION="0.1.2"
export NAMESPACE="test-1"
export APPLICATIONNAME="app-1" # Deployment name
export ARGOAPPNAME="${NAMESPACE}-${APPLICATIONNAME}"
export fullnameOverride="${APPLICATIONNAME}"
export ARGOCDNAMESPACE="openshift-gitops"
export valuesYAML=$(echo '|'; awk 'BEGIN{s=sprintf("%-8s", "");}{print s $0}' ../values.yaml)

# Clone repo and enter the folder of git repo
git clone "${GITREPO}" && cd "${CLUSTERNAME}"

# Create deployment yaml 
mkdir namespaces/${NAMESPACE}/${APPLICATIONNAME}
touch namespaces/${NAMESPACE}/${APPLICATIONNAME}/.gitignore
envsubst < templates/applicationManifestTemplate.yaml > namespaces/${APPLICATIONNAME}.yaml

git add --all; git commit -m "${APPLICATIONNAME}"; git push origin master
```

## Testing
You can deploy ArgoCD on Openshift with [Openshift Playground](https://developers.redhat.com/courses/explore-openshift/openshift-49-playground) via Red Hat GitOps Operator.

After deploying ArgoCD, you will need to define some RBAC rules to the **openshift-gitops-argocd-application-controller** for Openshift GitOps. For simplicity, I gave cluster-admin privileges in the test-cluster that is online on Openshift Playground.
```bash
## First login to the cluster
oc login -u admin -p admin
## Then give privileges to openshift-gitops-argocd-application-controller service account
oc adm policy add-cluster-role-to-user cluster-admin -n openshift-gitops -z openshift-gitops-argocd-application-controller
```

Now you can create an Application Object for ArgoCD to watch our repo, you will see that **namespaces** and all the resources will be created instantly.
```bash
cat << EOF | oc apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: test-cluster
  namespace: openshift-gitops
spec:
  destination:
    server: 'https://kubernetes.default.svc'
  project: default
  source:
    path: .
    repoURL: 'https://github.com/burhanuguz/test-cluster.git'
    targetRevision: HEAD
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - PruneLast=true
      - ApplyOutOfSyncOnly=true
EOF
```
