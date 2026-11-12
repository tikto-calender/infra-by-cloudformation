# TikTo Infrastructure (CloudFormation)

Provisions the TikTo Calendar AWS platform in Sydney (`ap-southeast-2`). [runtime.yaml](runtime.yaml) creates EKS, networking, nodes and load balancers. [data.yaml](data.yaml) creates the Dev and Prod DynamoDB tables, which are retained when the stack is deleted.

You need AWS CLI v2 with credentials, an IAM role for EKS administration, your public IPv4 CIDR, kubectl and Helm. Clone [Manifest](https://github.com/tikto-calender/manifest) beside this repository. Run the examples from the directory containing `runtime.yaml`.

## 1. Prepare DynamoDB

If both tables are new, run this on **Linux/macOS or Windows PowerShell**.

```sh
aws cloudformation deploy --region ap-southeast-2 --stack-name tikto-data --template-file ./data.yaml
```

If the tables already exist, pass their names to the runtime stack. If only one exists, create the missing table with string partition key `scope` and sort key `id`. To import an existing Prod table, use [data-import.yaml](data-import.yaml) after coordinating ownership with its current IaC state. [AWS import guide](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/import-resources-manually.html)

## 2. Deploy runtime

Replace the example CIDR and IAM role. Use the actual table names if they differ from the defaults. Use separate resource names if another runtime already owns the default names.

**Linux/macOS**

```sh
admin_role='arn:aws:iam::123456789012:role/TikToEksOperator'
aws cloudformation deploy --region ap-southeast-2 --stack-name tikto-runtime \
  --template-file ./runtime.yaml --capabilities CAPABILITY_IAM \
  --parameter-overrides 'EksAdminCidr=203.0.113.5/32' "EksAdminRoleArn=$admin_role" \
    DevTableName=tikto-dev-eks-app ProdTableName=tikto-prod-app
```

**Windows PowerShell**

```powershell
$adminRole = 'arn:aws:iam::123456789012:role/TikToEksOperator'
aws cloudformation deploy --region ap-southeast-2 --stack-name tikto-runtime `
  --template-file ./runtime.yaml --capabilities CAPABILITY_IAM `
  --parameter-overrides 'EksAdminCidr=203.0.113.5/32' "EksAdminRoleArn=$adminRole" `
    DevTableName=tikto-dev-eks-app ProdTableName=tikto-prod-app
```

For a Dev-only runtime, add `EnableProdAlb=false` to the overrides. If updating an existing two-slot Prod stack, follow the [Backend migration steps](https://github.com/tikto-calender/manifest#migrate-an-existing-backend-deployment) before updating the listener rule.

## 3. Connect and deploy the app

List the stack outputs on **either platform**.

```sh
aws cloudformation describe-stacks --region ap-southeast-2 --stack-name tikto-runtime --query 'Stacks[0].Outputs[].[OutputKey,OutputValue]' --output table
```

Set the variables below from `ClusterName` and `VpcId`. Use the same admin role as above.

**Linux/macOS**

```sh
cluster='your-cluster-name'
vpc='your-vpc-id'
aws eks update-kubeconfig --region ap-southeast-2 --name "$cluster" --role-arn "$admin_role"
```

**Windows PowerShell**

```powershell
$cluster = 'your-cluster-name'
$vpc = 'your-vpc-id'
aws eks update-kubeconfig --region ap-southeast-2 --name $cluster --role-arn $adminRole
```

Omit `--role-arn` if AWS CLI already uses the admin role. Run the [controller installation commands](https://github.com/tikto-calender/infra#install-controllers), then [apply the Manifest applications](https://github.com/tikto-calender/manifest#deploy-with-argo-cd). Check the frontend and `/api/health` at the `DevUrl` or `ProdUrl` output.
