- Preparation:
    - Certified Kubernetes Application Developer Study Guide: In-Depth Guidance and Practice by [[Benjamin Muschko]]
    - Github repo: 
        - [ckad-crash-course](https://github.com/bmuschko/ckad-crash-course)
        - [ckad-exercises](https://github.com/dgkanatsios/CKAD-exercises)
    - K8s Playground:
        - https://killercoda.com/killer-shell-ckad/scenario/playground
- ~/.bashrc:
    - ```shell
      # set alias or export following
      export do="--dry-run=client -oyaml"
      export now="--grace-period 0 --force"
      ```
- Notes:
    - configmap:
        - use all configmap values as environment variables:
            - ```yaml
              apiVersion: v1
              kind: Pod
              metadata:
                name: nginx
              spec:
                containers:
                - image: nginx
                  name: nginx
                  envFrom:
                  - configMapRef:
                      name: <my-configmap>
              ```
        - use single value from configmap as environment variable:
            - ```yaml
              apiVersion: v1
              kind: Pod
              metadata:
                name: nginx
              spec:
                containers:
                - image: nginx
                  name: nginx
                  env:
                    - name: USERNAME
                      valueFrom:
                        configMapKeyRef:
                          name: <my-configmap>
                          key: username
              ```
        - mount configmap as volume/file:
            - ```yaml
              apiVersion: v1
              kind: Pod
              metadata:
                name: nginx
              spec:
                volumes:
                - name: config-dir
                  configMap: 
                    name: <my-configmap>
                containers:
                  - name: nginx
                    image: nginx
                    volumeMounts:
                    - name: config-dir
                      mountPath: /etc/config
              ```
        - mount configmap to specific path in volume:
            - ```yaml
              # mount configmap SPECIAL_LEVEL to /etc/config/keys
              apiVersion: v1
              kind: Pod
              metadata:
                name: nginx
              spec:
                volumes:
                - name: config-dir
                  configMap: 
                    name: <my-configmap>
                    items:
                    - key: SPECIAL_LEVEL
                      path: keys
                containers:
                  - name: nginx
                    image: nginx
                    volumeMounts:
                    - name: config-dir
                      mountPath: /etc/config
              ```
    - secrets:
        - use all secrets as environment variables:
            - ```yaml
              apiVersion: v1
              kind: Pod
              metadata:
                name: nginx
              spec:
                containers:
                - image: nginx
                  name: nginx
                  envFrom:
                  - secretRef:
                      name: <my-secret>
              ```
        - use single value from configmap as environment variable:
            - ```yaml
              apiVersion: v1
              kind: Pod
              metadata:
                name: nginx
              spec:
                containers:
                - image: nginx
                  name: nginx
                  env:
                    - name: DB_PASSWORD
                      valueFrom:
                        secretKeyRef:
                          name: <my-password>
                          key: db_password
              ```
        - mount secret as volume:
            - ```yaml
              apiVersion: v1
              kind: Pod
              metadata:
                name: nginx
              spec:
                volumes:
                - name: secret-dir
                  secret:
                    secretName: <my-secret>
                containers:
                - image: nginx
                  name: nginx
                  volumeMounts:
                  - name: secret-dir
                    mountPaht: /env/secret-volume
              ```
    - annotations and labels:
        - annotate multiple pods:
            - ```shell
              kubectl annotate pod nginx{1..3} description='my description'
              ```
        - remove annotation from multiple pods:
            - ```shell
              kubectl annotate pod nginx{1..3} description-
              ```
        - add new label to specific labelled pods:
            - ```shell
              kubectl label pod -l "app in(v1,v2)" tier=web
              ```
        - add new label as column to output:
            - ```shell
              kubectl get po -L app
              ```
        - Label pod, overwrite existing label and remove label:
            - ```shell
              # add new label
              kubectl label pod my-pod region=cac

              # modify/overwrite existing label
              kubectl label pod my-pod region=use --overwrite

              # remove label
              kubectl label pod my-pod region-
              ```
        - labels vs annotations:
            - labels:
                - max character length is 63
                - used for querying data or k8s objects
                    - sample query:
                        - ```shell
                          kubectl get pods --show-labels -l 'region in (cac, cae),env=prod'
                          ```
                - similar to tags applied on a blog post
            - annotations:
                - ```yaml
                  apiVersion: v1
                  kind: Pod
                  metadata:
                    annotations:
                      release: prod
                      author: 10x Engineer
                      commitID: xxxxx
                  spec:
                    containers:
                    - name: nginx
                      image: nginx
                  ```
                - used for descriptive metadata that cannot be fit into labels
                - CANNOT be used for querying
                - you would use annotation for: SCM commit ID, releaseVersion and/or contact details
                - represent description metadata
                - can be more than 63 characters
    - requests, limits and autoscaling:
        - sample pods with limits:
            - ```yaml
              apiVersion: v1
              kind: Pod
              metadata:
                name: nginx
              spec:
                containers:
                - image: nginx
                  name: nginx
                  resources:
                    requests:
                      memory: "64Mi"
                      cpu: "250m"
                    limits:
                      memory: "128Mi"
                      cpu: "500m"
              ```
        - enable autoscaling on a pod:
            - [[VerticalPodAutoscaler]]
                - these can only work if you cloud provider support working of these /shrug
            - [[HorizontalPodAutoscaler]]
                - requires metrics server to be enabled
                - ```shell
                  # enable scaling based on CPU average of 70%
                  k autoscale deployment/my-deploy --min=3 --max=5 --cpu-percent=70

                  # check HPA
                  k describe hpa
                  ```
    - service:
        - ```shell
          # create a new pod + expose it via service
          k run nginx --image=nginx --restart=Never --port --expose=80
          # service/nginx created
          # pod/nginx created
          ```
        - a service will always have two ports:
            - incoming port (where it will accept requests) `--port`
            - outgoing port (where to send request to), normally this is where container is listening on `--targetPort`
        - types of service:
            - ClusterIP [internal]:
                - means only accessible from inside the cluster
            - NodePort [external]:
                - meant for external traffic and can be accessed from outside
    - deployments:
        - provides replication, scaling and rollouts
        - deployments under the hood: 
            - {{mermaid}}
                - flowchart TB
                    - Deployment ---|create 3 replicas| ReplicaSet
                    - ReplicaSet ---|maintain 3 pods| Pod1
                    - ReplicaSet --> Pod2
                    - ReplicaSet --> Pod3
        - rollouts:
            - by default k8s preserves maximum of latest 10 revisions
            - limit can be changed by assigning a different value to `deployments.spec.revisionHistoryLimit`
    - jobs:
        - Job:
            - a job is one time process or a batch job, used for running import/export of batch processes that are IO extensive
            - job remains in k8s cluster unless explicitly removed/deleted for debugging purposes
            - you can however specify `jobs.spec.ttlSecondsAfterFinished` to have k8s cluster auto delete the jobs
        - [[CronJob]]:
            -  a job but on interval
    - Always use `--grace-period=0 --force` when terminating/killing pod(s), this help ensure that pod is removed ASAP using SIGTERM rather than waiting for graceful shutdown which is 30s by default
    - You are NOT supposed to touch anything in `kube-` prefixed namespaces :-)
    - Run a temporary container to check if other pods is responding and in good shape, you can use this for DNS querying and other stuff as well:
        - ```shell
          kubectl run busybox --image=busybox --restart=Never --rm -it -- wget -O- <POD_IP>
          ```
    - Check for pod events:
        - ```shell
          kubectl get pod nginx | grep Events -C 10
          ```
    - Always create docker image that executes program other than default/root user(0)
    - Namespace by default is all open with no CPU/Memory quota defined, you can really mess up this if no limits are present on your namespaces.
    - Container patterns:
        - Init Containers (multiple containers)
        - Sidecar:
            - alerts based on logs, sidecar container monitors the directory/log files and then action upon events
        - Adapter:
            - logfile translation
                - sidecar container can translate modify these so that other processes can pick it up
        - Ambassador: 
            - request querying/rate limiting
            - act as a proxy (think of istio)
    - Observability:
        - Probes:
            - readiness:
                - pod ready to accept network traffic
            - liveness:
                - this app is alive or not
            - startup:
                - long startup time for app that needs a good amount of time to start
            - Probes Options:
                - httpGet: run HTTP get request
                - exec: run a comma
                - TCP: check if port is up and TCP can bind a connection
        - Troubleshooting:
            - run a temporary pod to test DNS:
                - `kubectl create tmp --image=busybox --restart=Never --rm -it -- wget -O- google.com`
            - check endpoints for a service
    - Pod Networking:
        - NetworkPolicies:
            - pracitce from these:
                - https://github.com/ahmetb/kubernetes-network-policy-recipes
    - State Persistence:
        - ```shell
          # list existing storageclasses
          k get storageclass
          ```
        - {{mermaid}}
            - flowchart RL
            - PersistentVolume ---> PersistentVolumeClaim
            - PersistentVolumeClaim ---> Pod
