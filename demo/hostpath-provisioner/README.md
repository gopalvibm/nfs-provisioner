#Writing an Out-of-tree Dynamic Provisioner Using nfs-provisioner

One of the goals of the [nfs-provisioner](https://github.com/kubernetes-incubator/nfs-provisioner) project is to serve as an example implementation of an out-of-tree dynamic provisioner to help others write their own. nfs-provisioner creates NFS-backed PVs in its own opinionated way, and we'll show that it's easy to write your own provisioner that does it your way.

Note you do **not** have to fork nfs-provisioner. As we will see here, you can treat the [`controller` directory](https://github.com/kubernetes-incubator/nfs-provisioner/tree/master/controller) of the nfs-provisioner repo as a library/dependency to import. This gives you the ability to control which version of it to use and you won't have to worry about rebasing your fork. You can rest assured that the controller has no NFS-specific functionality in it. There are plans to move this controller library into its own repository so stay tuned.

##The Provisioner Interface

Ideally, all you should need to do to write your own provisioner is implement the `Provisioner` interface which has two methods: `Provision` and `Delete`. Then you can just pass it to the `ProvisionController`, which handles all the logic of calling the two methods. The signatures should be self-explanatory but we'll explain the methods in more detail anyhow. For this explanation we'll refer to the `ProvisionController` as the controller and the implementer of the `Provisioner` interface as the provisioner. The code can be found in the `controller` directory of the nfs-provisioner repo.

```go
Provision(VolumeOptions) (*v1.PersistentVolume, error)
```

`Provision` creates a storage asset and returns a `PersistentVolume` object representing that storage asset. The given `VolumeOptions` object includes information needed to create the PV: the PV's reclaim policy, PV's name, the PVC object for which the PV is being provisioned (which has in its spec capacity & access modes), & parameters from the PVC's storage class.

You should store any information that will be later needed to delete the storage asset here in the PV using annotations. It's also recommended that you give every instance of your provisioner a unique identity and store it on the PV using an annotation here, for reasons we will see soon.

`Provision` is not responsible for actually creating the PV, i.e. submitting it to the Kubernetes API, it just returns it and the controller handles creating the API object.

```go
Delete(*v1.PersistentVolume) error
```

`Delete` removes the storage asset that was created by `Provision` to back the given PV. The given PV will still have any useful annotations that were set earlier in `Provision`.

Special consideration must be given to the case where multiple controllers that serve the same storage class (that have the same `provisioner` name) are running: how do you know that *this* provisioner was the one to provision the given PV? This is why it's recommended to store a provisioner's identity on its PVs in `Provision`, so that each can remember if it was the one to provision a PV when it comes time to delete it, and if not, ignore it by returning `IgnoredError`.

`Delete` is not responsible for actually deleting the PV, i.e. removing it from the Kubernetes API, it just deletes the storage asset backing the PV and the controller handles deleting the API object.

##Writing a `hostPath` Dynamic Provisioner

Now that we understand the interface expected by the controller, let's implement it and create our own out-of-tree `hostPath` dynamic provisioner. This is for single node testing and demonstration purposes only - local storage is not supported in any way and will not work on multi-node clusters. This simple program has the power to delete and create local data on your node, so if you intend to actually follow along and run it, be careful!

We define a `hostPathProvisioner` struct. It will back every `hostPath` PV it provisions with a new child directory in `pvDir`, hard-coded here to `/tmp/hostpath-provisioner`. It will also give itself a unique `identity`.

```go
type hostPathProvisioner struct {
	// The directory to create PV-backing directories in
	pvDir string

	// Identity of this hostPathProvisioner, generated. Used to identify "this"
	// provisioner's PVs.
	identity types.UID
}

func NewHostPathProvisioner() controller.Provisioner {
	return &hostPathProvisioner{
		pvDir:    "/tmp/hostpath-provisioner",
		identity: uuid.NewUUID(),
	}
}
```

We implement `Provision`. It creates a directory with the name `options.PVName`, which is always unique to the PVC being provisioned for, in `pvDir`. It sets a custom `identity` annotation on the PV and fills in the other fields of the PV according to the `VolumeOptions` to satisfy the PVC's requirements. And the PV's `PersistentVolumeSource` is of course set to a `hostPath` volume representing the directory just created.

```go
// Provision creates a storage asset and returns a PV object representing it.
func (p *hostPathProvisioner) Provision(options controller.VolumeOptions) (*v1.PersistentVolume, error) {
	path := path.Join(p.pvDir, options.PVName)

	if err := os.MkdirAll(path, 0777); err != nil {
		return nil, err
	}

	pv := &v1.PersistentVolume{
		ObjectMeta: v1.ObjectMeta{
			Name: options.PVName,
			Annotations: map[string]string{
				"hostPathProvisionerIdentity": string(p.identity),
			},
		},
		Spec: v1.PersistentVolumeSpec{
			PersistentVolumeReclaimPolicy: options.PersistentVolumeReclaimPolicy,
			AccessModes:                   options.PVC.Spec.AccessModes,
			Capacity: v1.ResourceList{
				v1.ResourceName(v1.ResourceStorage): options.PVC.Spec.Resources.Requests[v1.ResourceName(v1.ResourceStorage)],
			},
			PersistentVolumeSource: v1.PersistentVolumeSource{
				HostPath: &v1.HostPathVolumeSource{
					Path: path,
				},
			},
		},
	}

	return pv, nil
}
```

We implement `Delete`. First it checks if this provisioner was the one that created the directory backing the given PV by looking at the identity annotation. If not, it returns an `IgnoredError`. Otherwise, it can safely delete it.

```go
// Delete removes the storage asset that was created by Provision represented
// by the given PV.
func (p *hostPathProvisioner) Delete(volume *v1.PersistentVolume) error {
	ann, ok := volume.Annotations["hostPathProvisionerIdentity"]
	if !ok {
		return errors.New("identity annotation not found on PV")
	}
	if ann != string(p.identity) {
		return &controller.IgnoredError{"identity annotation on PV does not match ours"}
	}

	path := path.Join(p.pvDir, volume.Name)
	if err := os.RemoveAll(path); err != nil {
		return err
	}

	return nil
}
```

Now all that's left is to connect our `Provisioner` with a `ProvisionController` and run the controller, all in `main`. This part will look largely the same regardless of how the provisioner interface is implemented. We'll write it such that it expects to be run as a pod in Kubernetes.

We need to create a couple of things the controller expects as arguments, including our `hostPathProvisioner`, before we create and run it. First we create a client for communicating with Kubernetes from within a pod. We use it to determine the server version of Kubernetes. Then we create our `hostPathProvisioner`. We pass all of these things into `NewProvisionController`, plus some other arguments we'll explain now. 

The second argument is the `resyncPeriod` of the controller which determines how often it relists PVCs and PVs to check if they should be provisioned for or deleted. The third is the `provisionerName` that storage classes will specify, "example.com/hostpath" here. The second-to-last is `exponentialBackOffOnError` which determines whether it should exponentially back off from calls to `Provision` or `Delete`, useful if either of those involves some API call. And the last is `failedRetryThreshold`, the threshold for failed `Provision` attempts before giving up. (There are many other possible parameters of the controller that could be exposed, please create an issue if you would like one to be.)

Finally, we create and `Run` the controller.

```go
const (
	resyncPeriod              = 15 * time.Second
	provisionerName           = "example.com/hostpath"
	exponentialBackOffOnError = false
	failedRetryThreshold      = 5
)
```
```go
func main() {
	flag.Parse()
	// Create an InClusterConfig and use it to create a client for the controller
	// to use to communicate with Kubernetes
	config, err := rest.InClusterConfig()
	if err != nil {
		glog.Fatalf("Failed to create config: %v", err)
	}
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		glog.Fatalf("Failed to create client: %v", err)
	}

	// The controller needs to know what the server version is because out-of-tree
	// provisioners aren't officially supported until 1.5
	serverVersion, err := clientset.Discovery().ServerVersion()
	if err != nil {
		glog.Fatalf("Error getting server version: %v", err)
	}

	// Create the provisioner: it implements the Provisioner interface expected by
	// the controller
	hostPathProvisioner := NewHostPathProvisioner()

	// Start the provision controller which will dynamically provision hostPath
	// PVs
	pc := controller.NewProvisionController(clientset, resyncPeriod, "example.com/hostpath", hostPathProvisioner, serverVersion.GitVersion, exponentialBackOffOnError, failedRetryThreshold)
	pc.Run(wait.NeverStop)
}
```

We're now done writing code. The code we wrote can be found [here](./hostpath-provisioner.go). The other files we'll use in the remainder of the walkthrough can be found in the same directory.

##Building and Running our `hostPath` Dynamic Provisioner

Before we can run our provisioner in a pod we need to build a Docker image for the pod to specify. Our hostpath-provisioner Go package has many dependencies so it's a good idea to use a tool to manage them. It's especially important to do so when depending on a package like [client-go](https://github.com/kubernetes/client-go#how-to-get-it) that has an unstable master branch. We'll use [glide](https://github.com/Masterminds/glide).

Our [glide.yaml](./glide.yaml) was created by manually setting the latest version of nfs-provisioner & setting the version of client-go to the same one that nfs-provisioner uses. We use it to populate a vendor directory containing dependencies.

```
$ glide install -v
...
```

Now we can use the [Go Docker image](https://hub.docker.com/_/golang/) to build & run our hostpath-provisioner.

```dockerfile
FROM golang:1.7.4-onbuild
```

We build our Docker image. Note that the Docker image needs to be on the node we'll run the pod on. So you may need to tag your image and push it to Docker Hub so that it can be pulled later by the node, or just work on the node and build the image there.

```console
$ docker build -t hostpath-provisioner:latest .
...
Successfully built 9bb0954337a0
```

Now we can specify our image in a pod. Recall that we set `pvDir` to `/tmp/hostpath-provisioner`. Since we are running our provisioner in a container as a pod, we should mount a corresponding `hostPath` volume there to serve as the parent of all provisioned PVs' `hostPath` volumes.

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: hostpath-provisioner
spec:
  containers:
    - name: hostpath-provisioner
      image: hostpath-provisioner:latest
      imagePullPolicy: "IfNotPresent"
      volumeMounts:
        - name: pv-volume
          mountPath: /tmp/hostpath-provisioner
  volumes:
    - name: pv-volume
      hostPath:
        path: /tmp/hostpath-provisioner
```

If SELinux is enforcing, we need to label `/tmp/hostpath-provisioner` so that it can be accessed by pods. We do this on the single node those pods will be scheduled to.

```console
$ mkdir -p /tmp/hostpath-provisioner
$ sudo chcon -Rt svirt_sandbox_file_t /tmp/hostpath-provisioner
```

##Using our `hostPath` Dynamic Provisioner

As said before, this dynamic provisioner is for single node testing purposes only. It has been tested to work with [hack/local-up-cluster.sh](https://github.com/kubernetes/kubernetes/blob/release-1.5/hack/local-up-cluster.sh) started like so.

```console
$ API_HOST_IP=0.0.0.0 $GOPATH/src/k8s.io/kubernetes/hack/local-up-cluster.sh
```

Once our cluster is running, we create the hostpath-provisioner pod.

```console
$ kubectl create -f pod.yaml
pod "hostpath-provisioner" created
```

Before proceeding, we check that it doesn't immediately crash due to one of the fatal conditions we wrote.

```console
$ kc get pod
NAME                   READY     STATUS    RESTARTS   AGE
hostpath-provisioner   1/1       Running   0          5s
```

Now we create a `StorageClass` & `PersistentVolumeClaim` and see that a `PersistentVolume` is automatically created.

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
  name: example-hostpath
provisioner: example.com/hostpath
```

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: hostpath
  annotations:
    volume.beta.kubernetes.io/storage-class: "example-hostpath"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
```

```console
$ kubectl create -f class.yaml
storageclass "example-hostpath" created
$ kubectl create -f claim.yaml
persistentvolumeclaim "hostpath" created
$ kubectl get pv
NAME                                       CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS    CLAIM              REASON    AGE
pvc-f41f0dfc-c7bf-11e6-8c5d-c81f66424618   1Mi        RWX           Delete          Bound     default/hostpath             8s
```

If we check the contents of `/tmp/hostpath-provisioner` on the node we should see the PV's backing directory.

```console
$ ls /tmp/hostpath-provisioner/
pvc-f41f0dfc-c7bf-11e6-8c5d-c81f66424618
```

Now let's do a simple test: have a pod use the claim and write to it. We create such a pod and see that it succeeds.

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: test-pod
spec:
  containers:
  - name: test-pod
    image: gcr.io/google_containers/busybox:1.24
    command:
      - "/bin/sh"
    args:
      - "-c"
      - "touch /mnt/SUCCESS && exit 0 || exit 1"
    volumeMounts:
      - name: hostpath-pvc
        mountPath: "/mnt"
  restartPolicy: "Never"
  volumes:
    - name: hostpath-pvc
      persistentVolumeClaim:
        claimName: hostpath
```
```console
$ kubectl create -f test-pod.yaml
pod "test-pod" created
$ kubectl get pod --show-all
NAME                   READY     STATUS      RESTARTS   AGE
hostpath-provisioner   1/1       Running     0          2m
test-pod               0/1       Completed   0          8s
```

If we check the contents of the PV's backing directory we should see the data it wrote.

```console
$ ls /tmp/hostpath-provisioner/pvc-f41f0dfc-c7bf-11e6-8c5d-c81f66424618
SUCCESS
```

When we delete the PVC, the PV it was bound to and the data will be deleted also.

```console
$ kubectl delete pvc --all
persistentvolumeclaim "hostpath" deleted
$ kubectl get pv
No resources found.
$ ls /tmp/hostpath-provisioner
```

Finally, we delete the provisioner pod when it has deleted all its PVs and it's no longer wanted.

```console
$ kubectl delete pod hostpath-provisioner
pod "hostpath-provisioner" deleted
```

## Extras
So as we can see, it can be easy to write a simple but useful dynamic provisioner. For something more complicated here are some various other things to consider...

We did not show how to parse `StorageClass` parameters. They are passed from the storage class as a `map[string]string` to `Provision`, so you can define and parse any arbitrary set of parameters you want. You must reject parameters that you don't recognize.

We made it so our hostpath-provisioner binary must run from within a Kubernetes cluster. But it's also possible to have it communicate with Kubernetes from [outside](https://github.com/kubernetes/client-go/blob/release-2.0/examples/out-of-cluster/main.go). nfs-provisioner can do this and defines this (and other) behaviour using flags/arguments.

Note that the errors returned by Provision/Delete are sent as events on the PVC/PV and this is the primary way of communicating with the user, so they should be understandable.

If there is some behaviour of the controller you would like to change, feel free to open an issue. There are many parameters that could easily be made configurable but aren't because it would be too messy. The controller is written to follow the [proposal](https://github.com/kubernetes/kubernetes/pull/30285) and be like the upstream PV controller as much as possible, but there is always room for improvement.

It's possible (but not pretty) to write e2e tests for your provisioner that look similar to kubernetes e2e tests by copying files from the e2e framework and fixing import statements. Like [here](https://github.com/kubernetes-incubator/nfs-provisioner/tree/master/e2e). Keep in mind the license, etc. In your case, unit & integration tests may be sufficient. 
