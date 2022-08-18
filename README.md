# kraken-deploy
#### This repository contains manifests of running kraken as a service in kubernetes/openshift. Refer to this readme on how to get started with kraken. More information about kraken can be found at https://github.com/chaos-kubox.

## Getting Started with Kraken
#### Kraken can be run in multiple ways on a standalone container using podman, from openshift. In this example we will be running kraken from openshift and targeting a different openshift cluster to inject chaos into. kraken scenario will run as a pod and target a different openshift cluster as shown in the image below


![kraken-image](https://user-images.githubusercontent.com/72143431/181103468-14e4a870-cd58-4e1f-8ac8-6ebfd595e035.jpg)

## Running Kraken
You need 2 clusters, a host cluster to run kraken on and a tagrget cluster to inject chaos into.

Make sure you are cluster-admin on both of the clusters

Clone ![this](https://github.com/harshil-codes/kraken-deploy.git) git repository.

```
cd kraken-scenarios
# Login to the target cluster and get the kubeconfig, make sure you are cluster-admin on the target cluster
oc login $target_cluster
oc login --token=$(oc whoami -t) --server=$(oc whoami --show-server) --kubeconfig config
# Login to the host cluster create a new project
oc login $host_cluster
oc create ns $(oc whoami)-kraken
oc project $(oc whoami)-kraken
### Create the kube-config that is required by kraken
oc create configmap kube-config --from-file=config
### Create a new servieaccount and give it privileges to run as a privileged container
oc apply -f krkn-sa.yaml 
oc adm policy add-scc-to-user privileged -z useroot 
```

Now we identify and edit the scenario we want to run and apply the job, in this eg we are running pod disruption scenario, to kill an etcd pod.

```
$ cat krkn-job-pod-scenario.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: kraken-pod
spec:
  selector: {}
  template:
    metadata:
      name: kraken
    spec:
      serviceAccountName: useroot
      volumes:
      - name: config
        configMap:
          name: kube-config
      containers:
        - name: kraken-runtime
          serviceAccount: useroot
          image: quay.io/chaos-kubox/krkn-hub:pod-scenarios
          env:
            - name: NAMESPACE
              value: openshift-etcd  ---> Change the namespace to your desired namespace on the target cluster
            - name: POD_LABEL
              value: "app=etcd"      ---> Identify the right label that matches your pods
            - name: DISRUPTION_COUNT
              value: "1"              ---> Select the number of pods you want to delete
            - name: EXPECTED_POD_COUNT
              value: "3"
            - name: CERBERUS_ENABLED
              value: "False"
          volumeMounts:
            - name: config
              mountPath: /root/.kube
          securityContext:
            privileged: true
      restartPolicy: Never
```
Apply the scenario in your namespace on the host cluster
```
oc apply -f krkn-job-pod-scenario.yaml
```

This will create a pod, monitor that logs and see how kraken disrupts the pod on the target cluster.
