# Kubernetes Static IP for Ingress Controller



## Introduction and motivation
Recently at work I had a requirement to provide clients static IPs for our kubernetes cluster. AWS managed load balancers are fantastic but due to their elastic scale-ability you can not guerentee what IP addresses your load balancer is using. Fortuantly AWS provides the Layer 3 Network LoadBalancer (NLB). The NLB allows for Elastic IPs to attached to it, providing static IPs.

Because we are using an ingress-controller, nginx in this case, a Layer 3 loadbalancer is completely adequate, as the ingress-controller will handle routing to k8s services on the application layer.

## A solution

Thanks to the great work of the kubernetes community this PR provides exactly the functionality required. https://github.com/kubernetes/kubernetes/pull/69263.

When I provision my nginx ingress controller service I supple two additional notations.
1. `service.beta.kubernetes.io/aws-load-balancer-eip-allocations` which takes a comma seperated list of EIPs IDs.
2. `service.beta.kubernetes.io/aws-load-balancer-subnets` another comma seperated list of subnet IDs.

Kubernetes will do the rest to create a NLB spread over three subnets which should be over three AZs each with its own EIP.

Now in terraform I can provision my 3 elastic IPs one for each Availability-Zone/Subnet

```tf
resource "aws_eip" "kube_nlb_ip" {
  count = 3
  vpc   = true
  
  tags = {
      Name        = "kube-cluster-nlb-${count.index + 1}"
      Description = "Elastic IPs for kube-cluster to provide static IPs for allowlist rules"
  }
}
```
Now let's apply that into my environment
```bash
terraform apply
```
I can use terraform outputs to grab my EIPs ids and my subnet IDs I've provisioned earlier. Or quickly look them up in the AWS console under the VPC section.

Finally in the ingress controller YAML I need to add the additional annotations. These are on lines 9 and 10.

{{< admonition type=note title="I've removed yaml for brevity " open=true >}}
Click below to view
{{< /admonition >}}
```yml
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: tcp
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: 'true'
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
    # provide my eip allocations
    service.beta.kubernetes.io/aws-load-balancer-eip-allocations: eip-id123abc, eip-id456def, eip-id789ghi
    service.beta.kubernetes.io/aws-load-balancer-subnets: subnet-id123abc, subnet-id456def, subnet-id789
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  type: LoadBalancer
  ...
```

I can `kube apply -f` this with the rest of the manifest for nginx ingress-controller. You can view the documentation here for more information on that: https://kubernetes.github.io/ingress-nginx/deploy/#network-load-balancer-nlb

## Conclusion
At this point you should have deployed a ingress controller with a service of `type=LoadBalancer` that has a AWS NLB with three EIPs attached. This allows you to supply IPs for clients to allowlist on their firewalls.
