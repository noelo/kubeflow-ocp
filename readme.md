# Kubeflow for OpenShift

This guides helps you set up Kubeflow in an existing environment.
We assume that that teh following operators are installed:

* Red Hat ServiceMesh
* cert-manager
* Red Hat Serverless (not tested yet).
* namespace-configuration-operator
* resource-locker-operator

We assume that there is a service mesh control plane fully dedicated to the kubeflow workloads.

We set up sso between kubeflow and OpenShift.
When an OCP User is created the corresponding kubeflow profile is also created.

Prepare notebook image compatible with OCP security

```shell
oc apply -f ./openshift/build-notebook-image.yaml -n openshift
oc start-build tf-notebook-image -n openshift

this will create this image:
image-registry.openshift-image-registry.svc:5000/openshift/tf-notebook-image
```

Patch the service mesh for sso:

```shell
export sm_cp_namespace=istio-system #change based on your settings
export sm_cp_name=basic #change
export base_domain=$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')
envsubst < ./openshift/sm_cp_patch.yaml | oc apply -f - -n ${sm_cp_namespace}
```

Prepare the kubeflow namespace

```shell
oc new-project kubeflow
oc label namespace kubeflow  control-plane=kubeflow katib-metricscollector-injection=enabled istio-injection=enabled
envsubst < ./openshift/kubeflow-sm-member.yaml | oc apply -f - -n kubeflow
oc apply -f ./openshift/allow-apiserver-webhooks.yaml -n kubeflow
oc adm policy add-scc-to-user anyuid -z application-controller-service-account -n kubeflow
oc adm policy add-scc-to-user anyuid -z default -n kubeflow
oc adm policy add-scc-to-user anyuid -z seldon-manager -n kubeflow
oc adm policy add-scc-to-user anyuid -z kubeflow-pipelines-cache-deployer-sa -n kubeflow
oc adm policy add-scc-to-user anyuid -z admission-webhook-service-account -n kubeflow
```

deploy kubeflow

```shell
export manifests_dir=/home/rspazzol/git/kubeflow-manifests #change to something else if need be
git -C $(dirname "$manifests_dir") clone https://github.com/raffaelespazzoli/manifests -b ocp
export KF_DIR=$(pwd)/kfctl-run

## loop from here

rm -rf ${KF_DIR}
mkdir -p ${KF_DIR}
envsubst < ./kfctl_kube_multi-user.v1.2.0.yaml  > ./kfctl-run/kfctl_kube_multi-user.v1.2.0.yaml
cd ${KF_DIR}
kfctl build -V --file=./kfctl_kube_multi-user.v1.2.0.yaml
kfctl apply -V --file=./kfctl_kube_multi-user.v1.2.0.yaml
cd ..

## to here
```

Configure automatic profile creation, and profile namespace configuration:

```shell
export sm_cp_namespace=istio-system #change based on your settings
export sm_cp_name=basic #change
oc apply -f ./openshift/kubeflow-profile-creation.yaml
envsubst < ./openshift/kubeflow-profile-namespace-config.yaml | oc apply -f -
```

remove kubeflow

```shell
kfctl delete -V --file=./kfctl_kube_multi-user.v1.2.0.yaml
```
