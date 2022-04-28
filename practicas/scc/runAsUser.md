# SCC - Resticciones de Contexto de Seguridad

Tenemos un deployment en donde se le aplica un securityContsxt para qeu use el user: 1000
```
$ vim deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-openshift
spec:
  selector:
    matchLabels:
      app: hello-openshift
  template:
    metadata:
      labels:
        app: hello-openshift
    spec:
      containers:
      - command:
        - sh
        - -c
        - echo "Hola como estan" && sleep infinity
        image: ubi8/ubi-minimal
        imagePullPolicy: Always
        name: ubi8
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      # Agrego un contexto de seguridad
      securityContext:
        runAsUser: 1000
```

Aplicamos el deploymnents y vemos que no se crea el pods

```
$ oc apply -f deploy.yaml
$ oc get pods
No resources found in test namespace.
```
Vemos qeu dentro de los evenots hay un Warninng que indeica que el user que quiere user el pods esta fuera del rango permitido

```
$ oc get events --field-selector type=Warning
15s         Warning   FailedCreate   replicaset/hello-openshift-65f4656587   Error creating: pods "hello-openshift-65f4656587-" is forbidden: unable to validate against any security context constraint: [spec.containers[0].securityContext.runAsUser: Invalid value: 1000: must be in the ranges: [1000840000, 1000849999]]
```

Vemos qeu scc permite solucionar esta restricción

```
$ oc get deploy hello-openshift -o yaml | oc adm policy scc-subject-review -f -
RESOURCE                     ALLOWED BY
Deployment/hello-openshift   anyuid
```
Creamos una cueta de servicio, le asiganamos el scc correspondiente y se la configuramos el deployment

```
$ oc create serviceaccount hello-sa
serviceaccount/hello-sa created
$ oc adm policy add-scc-to-user anyuid -z hello-sa
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:hello-sa added: "anyuid"
$ oc set serviceaccount deploy/hello-openshift hello-sa
```
Aqui podemos ver qeu el pods se creo correctamente.
```
$ oc get pods
NAME                              READY   STATUS    RESTARTS   AGE
hello-openshift-c74c9b87f-fxdnz   1/1     Running   0          57s
```
Podemos ver una descripción del pod.
```
[root@bastion secret]# oc get pod hello-openshift-c74c9b87f-fxdnz -o yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    k8s.v1.cni.cncf.io/network-status: |-
      [{
          "name": "",
          "interface": "eth0",
          "ips": [
              "10.128.2.188"
          ],
          "default": true,
          "dns": {}
      }]
    k8s.v1.cni.cncf.io/networks-status: |-
      [{
          "name": "",
          "interface": "eth0",
          "ips": [
              "10.128.2.188"
          ],
          "default": true,
          "dns": {}
      }]
    openshift.io/scc: anyuid
  creationTimestamp: "2022-04-28T20:39:44Z"
  generateName: hello-openshift-c74c9b87f-
  labels:
    app: hello-openshift
    pod-template-hash: c74c9b87f
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:generateName: {}
        f:labels:
          .: {}
          f:app: {}
          f:pod-template-hash: {}
        f:ownerReferences:
          .: {}
          k:{"uid":"91260b18-7810-43bc-9069-fa73cc09ba2f"}:
            .: {}
            f:apiVersion: {}
            f:blockOwnerDeletion: {}
            f:controller: {}
            f:kind: {}
            f:name: {}
            f:uid: {}
      f:spec:
        f:containers:
          k:{"name":"ubi8"}:
            .: {}
            f:command: {}
            f:image: {}
            f:imagePullPolicy: {}
            f:name: {}
            f:resources: {}
            f:terminationMessagePath: {}
            f:terminationMessagePolicy: {}
        f:dnsPolicy: {}
        f:enableServiceLinks: {}
        f:restartPolicy: {}
        f:schedulerName: {}
        f:securityContext:
          .: {}
          f:runAsUser: {}
        f:serviceAccount: {}
        f:serviceAccountName: {}
        f:terminationGracePeriodSeconds: {}
    manager: kube-controller-manager
    operation: Update
    time: "2022-04-28T20:39:44Z"
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          f:k8s.v1.cni.cncf.io/network-status: {}
          f:k8s.v1.cni.cncf.io/networks-status: {}
    manager: multus
    operation: Update
    time: "2022-04-28T20:39:46Z"
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:status:
        f:conditions:
          k:{"type":"ContainersReady"}:
            .: {}
            f:lastProbeTime: {}
            f:lastTransitionTime: {}
            f:status: {}
            f:type: {}
          k:{"type":"Initialized"}:
            .: {}
            f:lastProbeTime: {}
            f:lastTransitionTime: {}
            f:status: {}
            f:type: {}
          k:{"type":"Ready"}:
            .: {}
            f:lastProbeTime: {}
            f:lastTransitionTime: {}
            f:status: {}
            f:type: {}
        f:containerStatuses: {}
        f:hostIP: {}
        f:phase: {}
        f:podIP: {}
        f:podIPs:
          .: {}
          k:{"ip":"10.128.2.188"}:
            .: {}
            f:ip: {}
        f:startTime: {}
    manager: kubelet
    operation: Update
    time: "2022-04-28T20:39:50Z"
  name: hello-openshift-c74c9b87f-fxdnz
  namespace: test
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicaSet
    name: hello-openshift-c74c9b87f
    uid: 91260b18-7810-43bc-9069-fa73cc09ba2f
  resourceVersion: "166402481"
  selfLink: /api/v1/namespaces/test/pods/hello-openshift-c74c9b87f-fxdnz
  uid: 216c567f-90cc-4140-b017-9b18504005a0
spec:
  containers:
  - command:
    - sh
    - -c
    - echo "Hola como estan" && sleep infinity
    image: ubi8/ubi-minimal
    imagePullPolicy: Always
    name: ubi8
    resources: {}
    securityContext:
      capabilities:
        drop:
        - MKNOD
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: sa-token-8n8gc
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  imagePullSecrets:
  - name: sa-dockercfg-955bl
  nodeName: prev-rkh2g-worker-jfklf
  preemptionPolicy: PreemptLowerPriority
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext:
    runAsUser: 1000
    seLinuxOptions:
      level: s0:c29,c14
  serviceAccount: sa
  serviceAccountName: sa
  terminationGracePeriodSeconds: 30
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  volumes:
  - name: sa-token-8n8gc
    secret:
      defaultMode: 420
      secretName: sa-token-8n8gc
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2022-04-28T20:39:44Z"
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: "2022-04-28T20:39:50Z"
    status: "True"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: "2022-04-28T20:39:50Z"
    status: "True"
    type: ContainersReady
  - lastProbeTime: null
    lastTransitionTime: "2022-04-28T20:39:44Z"
    status: "True"
    type: PodScheduled
  containerStatuses:
  - containerID: cri-o://215a612c15b2e302417118df4a74623b694e10591ec161068727e0717d6d9ec4
    image: registry.access.redhat.com/ubi8/ubi-minimal:latest
    imageID: registry.access.redhat.com/ubi8/ubi-minimal@sha256:d7ae367da43804656e144ba093daff1a0b62b2388021c6f797aeb684cd5b6781
    lastState: {}
    name: ubi8
    ready: true
    restartCount: 0
    started: true
    state:
      running:
        startedAt: "2022-04-28T20:39:49Z"
  hostIP: 10.54.154.220
  phase: Running
  podIP: 10.128.2.188
  podIPs:
  - ip: 10.128.2.188
  qosClass: BestEffort
  startTime: "2022-04-28T20:39:44Z"
```

