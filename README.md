**一、前提条件**

    部署 nfs-storageclass 存储
    
**二、编写rbac.yaml**

	# vim rbac.yaml
		apiVersion: v1
		kind: ServiceAccount
		metadata:
		  name: jenkins
		  namespace: kube-system
		---
		kind: ClusterRole
		apiVersion: rbac.authorization.k8s.io/v1beta1
		metadata:
		  name: jenkins
		rules:
		  - apiGroups: ["extensions", "apps"]
		    resources: ["deployments"]
		    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
		  - apiGroups: [""]
		    resources: ["services"]
		    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
		  - apiGroups: [""]
		    resources: ["pods"]
		    verbs: ["create","delete","get","list","patch","update","watch"]
		  - apiGroups: [""]
		    resources: ["pods/exec"]
		    verbs: ["create","delete","get","list","patch","update","watch"]
		  - apiGroups: [""]
		    resources: ["pods/log"]
		    verbs: ["get","list","watch"]
		  - apiGroups: [""]
		    resources: ["secrets"]
		    verbs: ["get"]
		---
		apiVersion: rbac.authorization.k8s.io/v1beta1
		kind: ClusterRoleBinding
		metadata:
		  name: jenkins
		roleRef:
		  apiGroup: rbac.authorization.k8s.io
		  kind: ClusterRole
		  name: jenkins
		subjects:
		  - kind: ServiceAccount
		    name: jenkins
		    namespace: kube-system
---------------------------------------------

**三、编写pvc存储pvc.yaml**

	kind: PersistentVolumeClaim
	apiVersion: v1
	metadata:
	  name: opspvc
	  namespace: kube-system
	spec:
	  accessModes:
	    - ReadWriteMany
	  resources:
	    requests:
	      storage: 20Gi
	  storageClassName: managed-nfs-storage              # 指定 sc
-------------------------------------------

**四、编写jenkins.yaml**

	# vim jenkins.yaml
		apiVersion: apps/v1
		kind: Deployment
		metadata:
		  name: jenkins
		  namespace: kube-system
		spec:
		  selector:
		    matchLabels:
		      app: jenkins
		  template:
		    metadata:
		      labels:
		        app: jenkins
		    spec:
		      terminationGracePeriodSeconds: 10
		      serviceAccountName: jenkins
		      containers:
		      - name: jenkins
		        image: jenkins/jenkins:lts
		        imagePullPolicy: IfNotPresent
		        ports:
		        - containerPort: 8080
		          name: web
		          protocol: TCP
		        - containerPort: 50000
		          name: agent
		          protocol: TCP
		        resources:
		          limits:
		            cpu: 1000m
		            memory: 1Gi
		          requests:
		            cpu: 500m
		            memory: 512Mi
		        livenessProbe:
		          httpGet:
		            path: /login
		            port: 8080
		          initialDelaySeconds: 60
		          timeoutSeconds: 5
		          failureThreshold: 12
		        readinessProbe:
		          httpGet:
		            path: /login
		            port: 8080
		          initialDelaySeconds: 60
		          timeoutSeconds: 5
		          failureThreshold: 12
		        volumeMounts:
		        - name: jenkinshome
		          subPath: jenkins
		          mountPath: /var/jenkins_home
		        env:
		        - name: LIMITS_MEMORY
		          valueFrom:
		            resourceFieldRef:
		              resource: limits.memory
		              divisor: 1Mi
		        - name: JAVAOPTS
		          value: -Xmx$(LIMITS_MEMORY)m -XshowSettings:vm -Dhudson.slaves.NodeProvisioner.initialDelay=0 -Dhudson.slaves.NodeProvisioner.MARGIN=50 -Dhudson.slaves.NodeProvisioner.MARGIN0=0.85 -Duser.timezone=Asia/Shanghai
		      securityContext:
		        fsGroup: 1000
		      volumes:
		      - name: jenkinshome
		        persistentVolumeClaim:
		          claimName: opspvc                   # 绑定 pvc
		---
		apiVersion: v1
		kind: Service
		metadata:
		  name: jenkins
		  namespace: kube-system
		  labels:
		    app: jenkins
		spec:
		  selector:
		    app: jenkins
		  type: NodePort
		  ports:
		  - name: web
		    port: 8080
		    targetPort: web
		    nodePort: 30002
		  - name: agent
		    port: 50000
		    targetPort: agent
----------------------------------------

**登录：http://139.198.21.75:30002**

**密码：**

    # cat  /data/nfs/kube-system-opspvc-pvc-3b95fb42-2d6f-4f37-8b89-ba519467da30/jenkins/secrets/initialAdminPassword

