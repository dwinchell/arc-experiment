# Sources
https://github.com/some-natalie/kubernoodles/tree/v0.9.6/openshift
https://github.com/some-natalie/kubernoodles/blob/v0.9.6/docs/admin-setup.md

# Instructions

## Walkthrough of Kubernoodles Instructions

These are based on [admin-setup.md](https://github.com/dwinchell/arc-experiment), with differences and additions noted

### Step 1 - Install Helm
Follow kubernoodles instructions.

### Step 2 - Install cert-manager
Install the cert-manager operator in openshift instead of doing these instructions. The reason is that the install method from kubernoodles wanted anyuid for cert-manager.

**Note:** The operator is in Tech Preview. We should assess if that's acceptable for now, but it's better than anyuid. Also, anyuid was not enough. It still had some error about seccomp.

### Step 3 - Install ARC
Follow kubernoodles instructions.

### Step 4 - Set GitHub URL
kubectl set env deploy actions-runner-controller -c manager GITHUB_ENTERPRISE_URL=https://api.github.com/ --namespace actions-runner-system

### Step 5 - Generate and configure PAT
kubectl create secret generic controller-manager -n actions-runner-system --from-literal=github_token=FILL_THIS_IN
**Note:** This will be an application token in the disconnected environment.

### Step 6 - Runner Deployment
oc new-project runners
Download https://raw.githubusercontent.com/some-natalie/kubernoodles/v0.9.6/deployments/podman.yml, renamed to dwinchell-runnerdeployment.yaml in example
Customize  dwinchell-runnerdeployment.yaml .spec.repository to name of repo (dwinchell/arc-experiment or whatever). there is probably a way to do this for multiple repos
oc get runnerdeployment
Don't run `oc get deployment` ... it won't show up because we created a CRD!


### Step 7 - Add SCCs for privliged containers :(

oc adm policy add-scc-to-user privileged -z default -n actions-runner-system
oc adm policy add-scc-to-user privileged -z 

Error here if you don't do this. See toubleshooting section
oc logs -n actions-runner-system actions-runner-controller-58ddc6d859-c6fg4 -c manager

### Step 8 - Fix PVC error

oc get event -n runners
```
11m         Warning   FailedScheduling                pod/podman-rft2m-z592m                          0/6 nodes are available: 6 persistentvolumeclaim "prod-tool-cache-pvc" not found.
```
Clearly it expects a tool cache to exist
Googling suggests this 

You can see this in the runner Pod:
oc get po podman-rft2m-9ks8k -o yaml
...
    volumeMounts:
    - mountPath: /runner
      name: runner
    - mountPath: /opt/statictoolcache
      name: runnertoolcache
...
  volumes:
  - emptyDir: {}
    name: runner
  - name: runnertoolcache
    persistentVolumeClaim:
      claimName: prod-tool-cache-pvc

Comes from RunnerDeployment
oc get runnerdeployment podman -o yaml
...
      volumeMounts:
      - mountPath: /opt/statictoolcache
        name: runnertoolcache
        readOnly: true
      volumes:
      - name: runnertoolcache
        persistentVolumeClaim:
          claimName: prod-tool-cache-pvc

RunnerDeployment volume config docs:
https://github.com/actions/actions-runner-controller/blob/master/docs/using-custom-volumes.md

Config I'm using came from here:
https://github.com/some-natalie/kubernoodles/blob/v0.9.6/deployments/podman.yml
It doesn't explain how to create the pvc

Docs for config I copied:
https://github.com/some-natalie/kubernoodles/tree/v0.9.6/deployments
      volumeMounts:  # this uses the read only cache set up in ../cluster-configs/runner-tool-cache.yml, more on this below
        - mountPath: /opt/hostedtoolcache  # mount path from within the container
          name: runnertoolcache
          readOnly: true
      volumes:
        - name: runnertoolcache
          persistentVolumeClaim:
            claimName: test-tool-cache-pvc  # which persistent volume claim to use
The volumes and volumeMounts blocks give each pod read-only access to a hosted tool cache. This allows users to call pre-made Actions, like actions/setup-python, without needing to download and install Python at every job run if the version of what the user wants is already in cache. More about this here.

That doc links this doc w/ instructions to set it up:
https://github.com/some-natalie/kubernoodles/blob/v0.9.6/cluster-configs/README.md#tool-cache-for-runners-using-persistentvolumeclaim
Which has some Azure Files storage setup instructions (already done because thanks ARO) ... then links this example w/ comments for creating the PVs and PVCs
https://github.com/some-natalie/kubernoodles/blob/v0.9.6/cluster-configs/README.md#tool-cache-for-runners-using-persistentvolumeclaim

oc create -f runner-tool-cache.yaml
(test-runners didn't exist because i haven't set it up. i let it errror out.)


Pod scheduled, didn't manage to attach the pv
```
3m57s       Normal    Scheduled                       pod/podman-rft2m-z592m                          Successfully assigned runners/podman-rft2m-z592m to aro-cluster-8xhn8-x4b7w-worker-eastus1-8hx88
118s        Warning   FailedAttachVolume              pod/podman-rft2m-z592m                          AttachVolume.Attach failed for volume "prod-tool-cache-pv" : Attach timeout for volume prod-unique-volumeid
115s        Warning   FailedMount                     pod/podman-rft2m-z592m                          Unable to attach or mount volumes: unmounted volumes=[runnertoolcache], unattached volumes=[kube-api-access-ff8b9 runner runnertoolcache]: timed out waiting for the condition
```

... going to delete the volume from the RunnerDeployment and come back to it later
... the pv definition probably needs tweaking somehow to work w/ ARO
... Dan is doing some kind of work regarding the tools cache and may have solved this or decided it's OBE
... talk to him about it Monday

### Step 7 - fix rate limiting auth error
Http response code: Forbidden from 'POST https://api.github.com/api/v3/actions/runner-registration'
{"message":"API rate limit exceeded for 137.135.122.226. (But here's the good news: Authenticated requests get a higher rate limit. Check out the documentation for more details.)","documentation_url":"https://docs.github.com/rest/overview/resources-in-the-rest-api#rate-limiting"}

... Runners might have incorrect auth configuration .... i had another error before a PAT ...
... also can decreas replicas to 1


# TODO List for Next Environment

## Caveats
There may be some additional steps because:
1. I was using github.com
2. I was using a PAT instead of an application
3. I disabled the tools cache for now because I was getting an error mounting the volumes. I was using Azure Files/ARO so we'll have to work through that.
4. I hit a rate limiting error before actually connecting. There may be a couple more steps afterward.

# TODO
1. Get cert-manager running, probably w/ the same cert-manager operator we were trying before. When I tried instructions from kubernoodles readme, i got a permissions error. Tried giving it anyuid, still got a seccomp error. Note, kubernoodles expects version v1.10.0. Version of the operator is 1.7.1-1 ... which may or may not be the installed version of cert-manager.
2. Do whatever setup kubernoodles needs of our cert-manager deployment
3. add helm repo for ARC
4. Create GitOps app for ARC helm chart
5. Create new GHES PAT for ARC to use. admin:enterprise scope only is used by default in kubernoodels readme (i.e. make it available enterprise wide). I used a personal PAT for a test repo.
6. Build and host runner images internally
7. Make deployment run rootless
8. Add our tools to our chosen base image
9. Ideally, swap in the ubi base image for fedora by editing the kubernoodels example base images
10. Figure out *IF* and why ARC default SA needs privileged SCC and if there's a way to make it not need that. (Added it, didn't fix, added runner scc, worked, didn't go back and test w/out controllser SCC ... but probably do need it since it's mentioned in the docs)
11. Figureout why runner namespace default AE needs privileged SCC and if there's a way to make it not need that
12. Pin version of base image in RunnerDeployment (no :latest)

# Troubleshooting
If you get the error below, it's because an SA (controller or runner, probably default in taht namespace), needs privileged SCC. Fix with commands below (and then find a way to not have to do this)

oc adm policy add-scc-to-user privileged -z default -n actions-runner-system
oc adm policy add-scc-to-user privileged -z default -n runners


```
2023-03-25T01:30:57Z	ERROR	Reconciler error	{"controller": "runner-controller", "controllerGroup": "actions.summerwind.dev", "controllerKind": "Runner", "runner": {"name":"podman-rft2m-z592m","namespace":"runners"}, "namespace": "runners", "name": "podman-rft2m-z592m", "reconcileID": "a21ad838-294e-4265-b58f-3f3ddf3e0e19", "error": "pods \"podman-rft2m-z592m\" is forbidden: unable to validate against any security context constraint: [provider \"anyuid\": Forbidden: not usable by user or serviceaccount, spec.containers[0].securityContext.runAsUser: Invalid value: 1000: must be in the ranges: [1000730000, 1000739999], spec.containers[0].securityContext.privileged: Invalid value: true: Privileged containers are not allowed, provider \"nonroot\": Forbidden: not usable by user or serviceaccount, provider \"hostmount-anyuid\": Forbidden: not usable by user or serviceaccount, provider \"machine-api-termination-handler\": Forbidden: not usable by user or serviceaccount, provider \"hostnetwork\": Forbidden: not usable by user or serviceaccount, provider \"hostaccess\": Forbidden: not usable by user or serviceaccount, provider \"node-exporter\": Forbidden: not usable by user or serviceaccount, provider \"privileged\": Forbidden: not usable by user or serviceaccount, provider \"privileged-genevalogging\": Forbidden: not usable by user or serviceaccount]"}
```

^ That error shows up with this command
oc logs -n actions-runner-system actions-runner-controller-58ddc6d859-c6fg4 -c manager
