# rbac    
create an serviceAccount and rbac for ingress-controller process to access kubernetes api.    
each namespace should have a ingress-controller serviceAccount.

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ingress-controller
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: ingress-controller
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - endpoints
      - nodes
      - pods
      - secrets
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - "extensions"
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch   
  - apiGroups:
      - ""
    resources:
      - events
    verbs:
      - create
      - patch
  - apiGroups:
      - "extensions"
    resources:
      - ingresses/status
    verbs:
      - update
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: ingress-controller
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - pods
      - secrets
      - namespaces
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - get
      - update
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - create
  - apiGroups:
      - ""
    resources:
      - endpoints
    verbs:
      - get
      - create
      - update
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ingress-controller
subjects:
  - kind: ServiceAccount
    name: ingress-controller
    namespace: default
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: ingress-controller
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ingress-controller
subjects:
  - kind: ServiceAccount
    name: ingress-controller
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: ingress-controller
```


# default-backend    
haproxy default page when not match any url rule(an 404 page)

```yaml
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: ingress-default-backend
spec:
  replicas: 1
  template:
    metadata:
      labels:
        run: ingress-default-backend
    spec:
      containers:
        - name: ingress-default-backend
          image: 20.26.28.55/kubernetes/defaultbackend:1.4
          ports:
          - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: ingress-default-backend
spec:
  ports:
  - port: 8080
  selector:
    run: ingress-default-backend
```

# test   
test service
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: test
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: test
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: test
    spec:
      containers:
      - command:
        - sh
        - /app/tomcat/bin/startup.sh
        env:
        - name: APPNAME
          value: ROOT.war
        - name: TZ
          value: Asia/Shanghai
        - name: JAVA_HOME
          value: /app/jdk
        image: 20.26.28.55/special/tomcat7:exporter
        imagePullPolicy: IfNotPresent
        livenessProbe:
          exec:
            command:
            - ls
          failureThreshold: 3
          initialDelaySeconds: 300
          periodSeconds: 20
          successThreshold: 1
          timeoutSeconds: 10
        name: test
        ports:
        - containerPort: 9100
          name: port0
          protocol: TCP
        resources:
          limits:
            cpu: 500m
            memory: 3Gi
        securityContext:
          runAsUser: 1000
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: test-svc
spec:
  selector:
    app: test
  ports:
  - port: 30091
    targetPort: 9100
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: test2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test2
  template:
    metadata:
      labels:
        app: test2
    spec:
      containers:
      - command:
        - sh
        - /app/tomcat/bin/startup.sh
        env:
        - name: APPNAME
          value: ROOT.war
        - name: TZ
          value: Asia/Shanghai
        - name: JAVA_HOME
          value: /app/jdk
        image: 20.26.28.55/special/tomcat7:exporter
        imagePullPolicy: IfNotPresent
        livenessProbe:
          exec:
            command:
            - ls
          failureThreshold: 3
          initialDelaySeconds: 300
          periodSeconds: 20
          successThreshold: 1
          timeoutSeconds: 10
        name: test2
        ports:
        - containerPort: 8080
          name: port0
          protocol: TCP
        resources:
          limits:
            cpu: 500m
            memory: 3Gi
        securityContext:
          runAsUser: 1000
---
apiVersion: v1
kind: Service
metadata:
  name: test2-svc
spec:
  selector:
    app: test2
  ports:
  - port: 30092
    targetPort: 8080
```

# haproxy-ingress-controller    
haproxy-ingress-controller with haproxy.

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    run: haproxy-ingress
  name: haproxy-ingress
spec:
  replicas: 1
  selector:
    matchLabels:
      run: haproxy-ingress
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        run: haproxy-ingress
    spec:
      nodeSelector:
        kubernetes.io/hostname: csv-dcosnew45   # select a node to run haproxy
      containers:
      - args:
        - --default-backend-service=$(POD_NAMESPACE)/ingress-default-backend  # default-backend
        - --default-ssl-certificate=$(POD_NAMESPACE)/tls-secret
        - --configmap=$(POD_NAMESPACE)/haproxy-ingress
        - --ingress-class=ha-test   # only fetch ingress rules with the same ingress-class on annotation
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
        image: 20.26.28.55/kubernetes/haproxy-ingress
        name: haproxy-ingress
        ports:
        - containerPort: 80
          hostPort: 80
          name: http
          protocol: TCP
        - containerPort: 443
          hostPort: 443
          name: https
          protocol: TCP
        - containerPort: 1936
          hostPort: 1936
          name: stat
          protocol: TCP
```

# ingress    

ingress, haproxy rules   
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: dashboard-kibana-ingress
  annotations:
    kubernetes.io/ingress.class: "ha-test"    # ingress-class, ingress-controller only fetch ingress with the same ingress-class
    ingress.kubernetes.io/ssl-redirect: "false"  # ssl-redirect must be false where not use ip instead of domainname in host
spec:
  rules:
    - http:
        paths:
        - path: /aaa
          backend:
            serviceName: test-svc
            servicePort: 30091
        - path: /bbb
          backend:
            serviceName: test2-svc
            servicePort: 30092
```
