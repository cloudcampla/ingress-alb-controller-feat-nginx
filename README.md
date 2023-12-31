## 1. Obtenemos la política que debemos aplicar:

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
```

## 2. Y creamos esa política dentro de AWS:

```bash
aws iam create-policy
--policy-name AWSLoadBalancerControllerIAMPolicy
--policy-document file://iam_policy.json
```

[Documentación oficial AWS
](https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html
)
## 3. Creamos un role en AWS y un service account en EKS con su respectiva relación de confianza:

```bash
eksctl create iamserviceaccount
--cluster=agosto-eks-cluster
--namespace=kube-system
--name=aws-load-balancer-controller
--role-name AmazonEKSLoadBalancerControllerRole
--attach-policy-arn=arn:aws:iam::628924030472:policy/AWSLoadBalancerControllerIAMPolicy
--approve
```

## 4. A través de HELM referenciamos el repositorio donde está el Chart de aws-load-balancer-controller y lo instalamos:

```bash
helm repo add eks https://aws.github.io/eks-charts

helm repo update eks

helm install aws-load-balancer-controller eks/aws-load-balancer-controller
-n kube-system
--set clusterName=agosto-eks-cluster
--set serviceAccount.create=false
--set serviceAccount.name=aws-load-balancer-controller
```

[Documentación oficial AWS LB Controller
](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.5/deploy/installation/
)
## 5. Añadir el siguiente TAG:

Private subnets tags:

- key: kubernetes.io/role/internal-elb
- value: 1

Public subnets tags:

- key: kubernetes.io/role/elb
- value: 1

Both the public and private subnets:

- key: kubernetes.io/cluster/agosto-eks-cluster
- value: shared

[Subnet Discovery Documentación
](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.5/deploy/subnet_discovery/
)
## 6. Instalar ingress nginx con el soporte para AWS ALB Controller:

```bash
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx \
  --set controller.service.type=LoadBalancer \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-name"=apps-ingress \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-type"=nlb \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-backend-protocol"=tcp \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-cross-zone-load-balancing-enabled"=true \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-nlb-target-type"=ip \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-scheme"=internet-facing \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-healthcheck-protocol"=TCP \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-healthcheck-path"=/healthz \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-healthcheck-port"=10254 
```

[Annotations for Service AWS LB Controller
](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.5/guide/service/annotations/)


## 7. Ejemplos de Ingress ALB y Nginx:

ALB:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-2048
  namespace: game-2048
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: service-2048
                port:
                  number: 80
```

NGINX:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingres-2048-nginx
  namespace: game-2048
annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: service-2048
                port:
                  number: 80
          - path: /test
            pathType: Prefix
            backend:
              service:
                name: service-2048-2
                port:
                  number: 80
```
