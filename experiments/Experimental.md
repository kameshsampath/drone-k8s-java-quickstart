# Experiments

Following are the experiments that could be done with Kube runner

## Deploy Drone Docker Runner(Devel)

```shell
helm upgrade --install --devel \
  drone-runner-docker drone/drone-runner-docker \
  --namespace=drone \
  --values $QUICKSTART_HOME/helm_vars/drone-runner-docker/values.yaml \
  --post-renderer  $QUICKSTART_HOME/k8s/drone-runner-docker/kustomize \
  --wait
```