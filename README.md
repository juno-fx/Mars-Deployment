<br />
<p align="center">
    <img src="/mars.png"/>
    <h3 align="center">Mars Deployment</h3>
    <p align="center">
        Mars microservice for managing network show storage.
    </p>
</p>

## Dependencies

- External Dependencies
  - [ArgoCD](https://argo-cd.readthedocs.io/en/stable/)
- Cluster Dependencies
  - [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
  - [Persistent Volume Claims](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
  - [Metrics Server](https://github.com/kubernetes-sigs/metrics-server)
- Juno Dependencies
  - [Titan]() - Coming soon

## Usage

Mars is a microservice that manages network show storage. It treats content as a flat group of "Capsules" which contain
"Crates". An example of this would look like this:

```shell
656796239c1b981568180614/
├── v001
│   ├── artifacts
│   │   └── Frames
│   │       └── Main
│   │           └── 3840x1634_exr
│   │               ├── 010_010_Ingest_plate_main_v001.####.exr
│   ├── publishes
│   └── scenes
└── v002
    ├── artifacts
    │   └── Frames
    │       └── Main
    │           └── 3840x1634_exr
    │               ├── 010_010_Ingest_plate_main_v002.####.exr
    ├── publishes
    └── scenes
```

In the example above, `656796239c1b981568180614` is the UUID of the database entry in the project management database. This
directory is called the "Capsule". Inside the Capsule, there are two versions, `v001` and `v002`. These are referred to as
"Crates". Inside each Crate, there are three directories: `artifacts`, `publishes`, and `scenes`. This removes the need for
a complex directory structure and allows for a more flexible way of storing and accessing content. Archiving is also substantially
easier as the entire Capsule can be zipped up and stored in a single file. This also allows for more flexible project management
and versioning.

```shell
<Capsule>/                   # Capsule directory, represents the database entry
└── <Crate>/                 # Crate directory, contains an iteration of work
    ├── artifacts            # Artifacts directory, contains all the work files. This can be organized however you want
    ├── publishes            # Publishes directory, contains all the published work files.
    └── scenes               # Scenes directory, scene files generate artifacts
```

This way of working makes the File System and content completely detached from the current structure of the project. It essentially
"flattens" the project structure and allows for a more flexible way of working. Juno Innovations provides a pipeline built around
this structure which includes viewers that allow you to browse this content using the database.


## Configuration

```yaml
mars:
  image:                            # Registry to pull the Mars server image from
  namespace: production             # Namespace to deploy the Mars server to

  show:                             # The show code used to grab the UID of the show user

  node_selector:                    # Node selector to deploy the Mars server to
    enable: true                    # Enable node selector
    labels:                         # Node selector to deploy the Mars server to
      kubernetes.io/arch: amd64


  autoscaling:
    min_replicas: 1                 # Minimum number of replicas to run
    max_replicas: 4                 # Maximum number of replicas to run

  mounts:                           # Mounts to attach to the Mars server
#    - name:                        # Name of the mount
#      claim_name:                  # Name of the persistent volume claim to mount
#      mount_path:                  # Where to mount the persistent volume into the Mars server
#      sub_path:                    # Sub path on the persistent volume to mount
```

### ArgoCD Application

Ideally, you will use an Application in ArgoCD to deploy Mars. Below is an example of how to set one up.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mars
  namespace: argocd
spec:
  project: <your project>
  destination:
    server: <target cluster>
    namespace: argocd
  
  # We use the multi source approach to allow for the use of a private values repo
  # https://argo-cd.readthedocs.io/en/stable/user-guide/multiple_sources/
  sources:
    - path: .
      repoURL: https://github.com/juno-fx/Mars-Deployment.git
      targetRevision: <the version of Mars you want to use>
      helm:
        valueFiles:
          - $values/mars.yaml

    - repoURL: https://<your values repo>.git
      targetRevision: <branch>
      ref: values
```

Once ArgoCD has synchronized the application, you should see the Mars server ready to go. 