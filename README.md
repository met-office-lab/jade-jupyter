# Our Pangeo [![Build Status](https://travis-ci.com/informatics-lab/our-pangeo.svg?branch=master)](https://travis-ci.com/informatics-lab/our-pangeo)

We have joined forces with the [Pangeo community](https://pangeo-data.github.io/)! Pangeo is a curated stack of software and tools to empower big data processing in the atmostpheric, oceanographic and climate community. Much of the work we did in our previous [Jade](https://github.com/informatics-lab?utf8=%E2%9C%93&q=jade&type=&language=) project has been integreated into Pangeo.

This repository contains a [helm chart](https://github.com/kubernetes/helm/blob/master/docs/charts.md) which allows you to stand up our custom version of the Pangeo stack. This chart is mainly going to be a wrapper the [Pangeo chart](https://zero-to-jupyterhub.readthedocs.io/en/latest/) along with config to add our custom stuff.

## Usage

First off you need [helm](https://github.com/kubernetes/helm) if you don't have it already.

You'll also need to symlink the config from our [private-config](https://github.com/met-office-lab/private-config) repo.

_If you're not a member of the Informatics Lab and are looking to set this up yourself then check out the `values.yaml` file and the config for the other dependencies._

```shell
ln -s /path/to/private-config/jade-pangeo/prod/secrets.yaml env/prod/secrets.yaml
ln -s /path/to/private-config/jade-pangeo/dev/secrets.yaml env/dev/secrets.yaml
```

Now you can go ahead and run helm.

```shell
# Add upstream pangeo repo and update
helm repo add pangeo https://pangeo-data.github.io/helm-chart/
helm repo update

# Get deps
helm dependency update jadepangeo

# Install
# prod
helm install jadepangeo --name=jupyterhub.informaticslab.co.uk --namespace=jupyter -f env/prod/values.yaml -f env/prod/secrets.yaml
# dev
helm install jadepangeo --name=pangeo-dev.informaticslab.co.uk --namespace=pangeo-dev -f env/dev/values.yaml -f env/dev/secrets.yaml

# Apply changes
# prod
helm upgrade jupyterhub.informaticslab.co.uk jadepangeo -f env/prod/values.yaml -f env/prod/secrets.yaml
# dev
helm upgrade pangeo-dev.informaticslab.co.uk jadepangeo -f env/dev/values.yaml -f env/dev/secrets.yaml

# Delete
# prod
helm delete jupyterhub.informaticslab.co.uk --purge
# dev
helm delete pangeo-dev.informaticslab.co.uk --purge
```

## Troubleshooting

Here are some common problems we experience with our Pangeo and ways to resolve them.


### 503 Errors when starting your notebook server

This happens for a range of reasons. The main ones are:
 - The notebook pod failing to start due to issues with the image. Often experienced after updating the docker image and upgrading to a new version. Roll back to the previous image to resolve.
 - AWS scaling being slow and Jupyter Hub (Kubespawner specifically) timing out. Attempting to start server again usually is successful.
 - User home directory being full. This causes a whole range of problems. Fix for this is to mount the home directory onto a separate pod and cleaning out some files ([see debugging persistent volume claims](https://medium.com/@jacobtomlinson/debugging-kubernetes-pvcs-a150f5efbe95)).


### Jupyter Hub failing to start after upgrade

Occasionally when upgrading the helm chart the hub fails to start and complains about a PVC attachment issue.

This happens because a new hub is created while the old hub is terminating. They both want to have the PVC (which in this case is an AWS EBS volume) but that can only be attached to one host at the same time. If the old and new pods are on different hosts they can get stuck.

This can also happen when AWS occasionally has problems mounting the EBS volume.

This will resolve itself with time, but due to backoff timouts this can be a while. To speed things along you can manually scale the hub down to one pod, then wait for all to temrinate, then scale back up.

```shell
# Scale down
kubectl -n jupyter scale deployment hub --replicas=0

# Scale up
kubectl -n jupyter scale deployment hub --replicas=1
```


### User home directory filling up

Frustratingly when a user's home directory fills up it can present itself in a myriad of ways, none of which are very descriptive of what is going on. Usually it results in repeated 400/500 errors in the browser.

No new kernels can be created as they require temporary files to be placed in the home directory. This means you cannot switch to the shell to tidy the files.

If a user logs out with a full home directory they may not be able to log back in.

If the user has an active kernel either in a notebook or shell they can try to clear out the files them selves. However the easiest way is for an admin with kubectl access to exec a bash session inside the user's pod and clean out the files.

```shell
kubectl -n jupyter exec -it jupyter-jacobtomlinson bash
```


### Kernels dying

When a kernel exceeds the memory limits specified in the `values.yaml` file it will be sent a `SIGKILL` by the Kubernetes kubelet. This causes the kernel to silently exit. When viewing this in the notebook the activity light will switch to 'restarting' then 'idle' but the cell will still appear to be executing and there will be no stderr output.

This is expected functionality but frustrating for users.


## Auto deployment

The auto deployment requires these environment variables to be set.

```shell
SECRETS_REPO # Git url of the private config repo.
SSH_KEY # Base 64 encode version of the private side of the github deploy key
CERTIFICATE_AUTHORITY_DATA 
CLUSTER_URL
CLIENT_CERTIFICATE_DATA
CLIENT_KEY_DATA
PASSWORD
USERNAME
```

`SSH_KEY` is the private key to match the deploy key for the repo. Should be in base64 format.

You can create one like so.
```shell
ssh-keygen -f ./key
SSH_KEY=$(cat key |base64)
```

`$SSH_KEY` is the env var `key.pub` is the public deploy key for github.


If you are already set up with `kubectl` most of the rest of the vars can be found in your `~/.kube/conf`, `k8-config.yaml` is a tempted version of this file.
