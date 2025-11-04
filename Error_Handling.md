## Error Handling 

While runnig the deployment i faced the error below :

AccessDenied: Not authorized to perform sts:AssumeRoleWithWebIdentity

and i was unable to fetch address to host the game . 

C:\Users\shrey\Desktop\AWS_EKS_GAME_Deploy> kubectl get ingress -n game-2048 

NAME        CLASS  HOSTS ADDRESS PORTS AGE 
ingress-2048  alb    *              80  66m

Though my IAM policy has all the ELB permissions, the controller cannot assume the IAM role via the service account. 
This is why the ALB never gets created and the Ingress ADDRESS remains empty.
To resolve this issue i have to create "IAM role trust relationship" :

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<account-id>:oidc-provider/oidc.eks.eu-west-1.amazonaws.com/id/<oidc-id>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.eu-west-1.amazonaws.com/id/<oidc-id>:sub": "system:serviceaccount:kube-system:aws-load-balancer-controller"
        }
      }
    }
  ]
}


After this Re-deploy controller / restart pods :

kubectl rollout restart deployment aws-load-balancer-controller -n kube-system

kubectl get ingress -n game-2048  : now you will be able to see the address .






