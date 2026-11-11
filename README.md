# TikTo on AWS CloudFormation

Two standalone stacks for the existing Helm and Argo CD manifests. Run the examples from this directory in PowerShell with AWS CLI credentials for the intended account. They target Sydney.

| Stack | Creates | When deleted |
| --- | --- | --- |
| data.yaml | Two on-demand DynamoDB tables with the existing scope/id keys. Prod has point-in-time recovery and deletion protection. | Both tables are retained. Storage, requests and enabled backup features can still incur charges. |
| runtime.yaml | VPC and three public subnets, EKS and managed EC2 workers, Pod Identity, Dev and Prod ALBs and target groups, and optional Container Insights. | Deletes EKS, workers, ALBs and the VPC together. The DynamoDB tables are independent. |

The defaults match the checked-in Helm target group and table names. Runtime has no NAT gateway, defaults to three On-Demand t3.medium nodes and disables optional Container Insights. EKS uses standard support and EKS-managed Kubernetes API data encryption. Runtime does not create a customer-managed KMS key. The two ALBs use HTTP to match the Terraform lab. Put real public traffic behind HTTPS before using this as a production endpoint.

## Create or reuse DynamoDB

If the current Terraform deployment already owns the two tables, skip this step and pass their names to runtime. Creating a stack with the same table names would fail. A retained table stays outside CloudFormation if its data stack is deleted. Reuse it on the next run, or import it explicitly if you want CloudFormation ownership again.

```powershell
$region = 'ap-southeast-2'
aws cloudformation deploy --region $region --stack-name tikto-data --template-file .\data.yaml
```

## Create runtime

Use the IAM role ARN that you will authenticate as for kubectl and your actual public CIDR. The example addresses and account ID are placeholders.

```powershell
$region = 'ap-southeast-2'
$adminCidr = '203.0.113.5/32'
$adminRole = 'arn:aws:iam::123456789012:role/TikToEksOperator'

aws cloudformation deploy --region $region --stack-name tikto-runtime `
  --template-file .\runtime.yaml --capabilities CAPABILITY_IAM `
  --parameter-overrides "EksAdminCidr=$adminCidr" "EksAdminRoleArn=$adminRole" `
    DevTableName=tikto-dev-eks-app ProdTableName=tikto-prod-app

aws cloudformation describe-stacks --region $region --stack-name tikto-runtime `
  --query 'Stacks[0].Outputs[].[OutputKey,OutputValue]' --output table
```

- For a short Dev-only test, pass EnableProdAlb=false and, if capacity is sufficient, NodeCount=2. Apply only the Dev Application. The default three nodes give both environments more room for surge pods.
- To test Prod Blue/Green, also supply PreviewRunnerCidr with the trusted runner's real fixed CIDR and GitHubOidcProviderArn with the existing provider ARN. Copy the runtime outputs into the manifest repository's prod Environment variables. Preview rules associate both target groups with the ALB so the inactive target group can report target health.
- For a rebuild, set ProdActiveSlot to the value in manifest's Prod backend-slot file. GitHub Actions changes the live listener action during promotion. Before updating or replacing this stack, keep ProdActiveSlot aligned with Git and verify the live rule afterwards. CloudFormation can reapply its declared listener action.
- If the old Terraform runtime is still running, do not create a second runtime with the same names. Either tear down the Terraform runtime under its own ownership first, or use another ProjectName and update the matching table names and target group names in the Helm environment values. Keep Dev and Prod DynamoDB table names distinct.

## Bootstrap the workloads

```powershell
aws eks update-kubeconfig --region $region --name tikto-prod-eks --role-arn $adminRole
kubectl config current-context
kubectl get nodes
```

Use the AWS Load Balancer Controller and Argo CD Helm commands in the [existing bootstrap guide](../Infrastructure/README.md). Use VpcId from the CloudFormation outputs for the controller's vpcId value. The runtime stack already associates the controller's service account and both tikto-app service accounts with Pod Identity. Then apply the namespaces and the enabled Argo Applications from [manifest/bootstrap](../manifest/bootstrap). Create the Telegram Secret separately if you need Telegram. The optional Argo Rollouts lab still needs its own controller installation.

## Stop runtime charges when done

For a live deployment, schedule downtime first. Confirm the kube context and remove the Argo Applications so they stop recreating bindings. Allow the load balancer controller to clean up before deleting EKS.

```powershell
kubectl config current-context
kubectl -n argocd delete application tikto-dev tikto-prod --ignore-not-found
kubectl -n tikto-dev delete targetgroupbindings --all --timeout=3m
kubectl -n tikto-prod delete targetgroupbindings --all --timeout=3m

aws cloudformation delete-stack --region $region --stack-name tikto-runtime
aws cloudformation wait stack-delete-complete --region $region --stack-name tikto-runtime
```

Run the namespace commands only for namespaces you created. If you added Kubernetes-managed Ingresses, LoadBalancer Services or PVCs, delete them and wait for their AWS resources to clear while controllers are still running. If stack deletion fails, check stack events before retrying.

Leave the data stack intact for the next run. Scaling EC2 down alone leaves the EKS control plane and ALBs billable. After deleting runtime, check for independently created load balancers, disks, retained CloudWatch logs, and any old Terraform resources if you are closing out the account.
