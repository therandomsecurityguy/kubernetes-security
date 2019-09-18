### Pod Security Policies
Most Pods do not need privileged access or even host access, so it should be ensured that a Pod requesting such access needs to be white listed explicitly. By default no one should be able to request privileges above the default to avoid being vulnerable through misconfiguration or malicious content of a Docker image.

A basic setup consists of an unprivileged and a privileged policy. The unprivileged is called "default" and the privileged called "privileged".

With AppArmor:

```sh
kubectl apply -f https://raw.githubusercontent.com/therandomsecurityguy/kubernetes-security/master/PodSecurityPolicies/default.psp.yaml
kubectl apply -f https://raw.githubusercontent.com/therandomsecurityguy/kubernetes-security/master/PodSecurityPolicies/privileged.psp.yaml

```

Pod Security Policies are evaluated based on access to the policy. When multiple policies are available, the pod security policy controller selects policies in the following order:

1. If any policies successfully validate the pod without altering it, they are used.
2. If it is a pod creation request, then the first valid policy in alphabetical order is used.
3. Otherwise, if it is a pod update request, an error is returned, because pod mutations are disallowed during update operations.

In order to use a policy, the requesting user **or** target pod's service account must be authorized to use the policy, by allowing the *use* verb on the policy. So, when defining access rules for policies, we need to think about which user is creating/updating the Pod. The user is different if a Pod is created directly through *kubectl* or through a deployment.

First, make sure that any user has access to the *default* policy, which ensures that Pods will be unprivileged by default:

```sh
kubectl apply -f https://raw.githubusercontent.com/therandomsecurityguy/kubernetes-security/master/PodSecurityPolicies/default-psp.clusterrolebinding.yaml
kubectl apply -f https://raw.githubusercontent.com/therandomsecurityguy/kubernetes-security/master/PodSecurityPolicies/default-psp.clusterrole.yaml
```

Some Pods in the cluster, especially if *kube-apiserver*, *kube-controller-manager*, *kube-scheduler* or *etcd* are running inside the cluster, need privileged access. To ensure those services will still start after introducing the *PodSecurityPolicy* controller, we need to grant cluster nodes and the legacy kubelet user access to the privileged policy for the *kube-system* namespace. For this to work, make sure you've `--authorization-mode=Node,RBAC`:

```sh
kubectl apply -f https://raw.githubusercontent.com/therandomsecurityguy/kubernetes-security/master/PodSecurityPolicies/privileged-psp.clusterrole.yaml
kubectl apply -f https://raw.githubusercontent.com/therandomsecurityguy/kubernetes-security/master/PodSecurityPolicies/privileged-psp-nodes.rolebinding.yaml
```

Your network provider will also need privileged access. Depending on which you're using the used service account is different. For Calico, you need to create the following role binding:

```sh
kubectl create -n kube-system -f - <<EOF
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: privileged-psp-calico
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: privileged-psp
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: calico
  namespace: kube-system
EOF
```

Before enabling the *PodSecurityPolicy* controller you should check your namespaces for Pods requiring privileged access and create role bindings accordingly. If you're certain all Pods are covered you can add "PodSecurityPolicy" to `--admission-control=...` of your *kube-apiserver* configuration and restart the API.

You can test your setup by creating a deployment like this:

```sh
kubectl create -f -<<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: privileged
spec:
  replicas: 1
  selector:
    matchLabels:
      name: privileged
  template:
    metadata:
      labels:
        name: privileged        
    spec:
      containers:
        - name: pause
          image: k8s.gcr.io/pause
          securityContext:
            privileged: true
EOF
```

If all is fine, the *privileged* deployment should fail to create the Pod.
