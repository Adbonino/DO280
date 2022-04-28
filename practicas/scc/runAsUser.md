# SCC - Restricciones de Contexto de Seguridad

Tenemos un deployment en donde se le aplica un securityContext para que use el user: 1000
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
Vemos que eventos y hay un Warninng que indica que el user que quiere user el pods esta fuera del rango permitido

```
$ oc get events --field-selector type=Warning
15s         Warning   FailedCreate   replicaset/hello-openshift-65f4656587   Error creating: pods "hello-openshift-65f4656587-" is forbidden: unable to validate against any security context constraint: [spec.containers[0].securityContext.runAsUser: Invalid value: 1000: must be in the ranges: [1000840000, 1000849999]]
```

Vemos que scc permite solucionar esta restricción

```
$ oc get deploy hello-openshift -o yaml | oc adm policy scc-subject-review -f -
RESOURCE                     ALLOWED BY
Deployment/hello-openshift   anyuid
```
Creamos una cuenta de servicio, le asignamos el scc correspondiente y se la configuramos al deployment

```
$ oc create serviceaccount sa
serviceaccount/hello-sa created
$ oc adm policy add-scc-to-user anyuid -z sa
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:hello-sa added: "anyuid"
$ oc set serviceaccount deploy/hello-openshift sa
```
Aqui podemos ver que el pods se creo correctamente.
```
$ oc get pods
NAME                              READY   STATUS    RESTARTS   AGE
hello-openshift-c74c9b87f-fxdnz   1/1     Running   0          57s
```
Descripción del pod.
```
[root@bastion secret]# oc get pod hello-openshift-c74c9b87f-fxdnz -o yaml
apiVersion: v1
kind: Pod
metadata:
  ...
  name: hello-openshift-c74c9b87f-fxdnz
  namespace: test
  ownerReferences
  ...
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
  ...
```

