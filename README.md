# TikTo Infrastructure · CloudFormation

Repo này dựng hạ tầng AWS cho TikTo Calendar bằng CloudFormation. Stack dữ liệu quản lý DynamoDB. Stack runtime tạo VPC, EKS, node EC2 và ALB cho ứng dụng. Bạn có thể xóa runtime và giữ bảng dữ liệu để dùng lại. Workload được Argo CD triển khai từ [repo Manifest](https://github.com/tikto-calender/manifest).

| Template | Dùng khi nào |
| --- | --- |
| [data.yaml](data.yaml) | Tạo mới hai bảng DynamoDB Dev và Prod. Cả hai được giữ khi xóa stack dữ liệu. |
| [data-import.yaml](data-import.yaml) | Đưa bảng Prod đã tồn tại vào một stack CloudFormation bằng change set `IMPORT`. |
| [runtime.yaml](runtime.yaml) | Tạo hạ tầng chạy ứng dụng. Mặc định có ba node `t3.medium` và hai ALB. |

## Chuẩn bị

- AWS CLI v2 với credentials cho đúng tài khoản, cùng kubectl và Helm.
- IAM role có thể dùng để quản trị EKS và CIDR IPv4 được phép truy cập Kubernetes API. Các ví dụ dùng vùng Sydney (`ap-southeast-2`).
- Clone Manifest cạnh repo này để dùng các file bootstrap.

**Linux/macOS và Windows PowerShell**

```sh
git clone https://github.com/tikto-calender/infra-by-cloudformation.git
git clone https://github.com/tikto-calender/manifest.git
cd infra-by-cloudformation
aws sts get-caller-identity
```

Nếu đã có mã nguồn, mở terminal ở thư mục chứa `runtime.yaml`. Dùng một công cụ quản lý cho cùng bộ tài nguyên. Nếu Terraform đang chạy runtime trùng tên, chuyển quyền quản lý hoặc chọn `ProjectName` và các tên tài nguyên riêng.

## 1. Chuẩn bị DynamoDB

Nếu cả hai bảng chưa tồn tại, chạy lệnh sau trên **Linux/macOS hoặc Windows PowerShell**.

```sh
aws cloudformation deploy --region ap-southeast-2 --stack-name tikto-data --template-file ./data.yaml
```

Nếu đã có cả hai bảng, dùng tên của chúng ở bước tạo runtime. Nếu chỉ có bảng Prod, chuẩn bị bảng Dev riêng vì `data.yaml` tạo cả hai tên mặc định. Hai bảng dùng khóa `scope` và `id` kiểu chuỗi.

Để CloudFormation quản lý một bảng Prod đã tồn tại, dùng [data-import.yaml](data-import.yaml) với change set `IMPORT` và ánh xạ logical ID `ProdTable` tới tên bảng. Đối chiếu khóa, billing, mã hóa, deletion protection và point-in-time recovery với bảng thật. Nếu bảng đang nằm trong Terraform state hoặc stack khác, hoàn tất việc chuyển quyền quản lý trước khi import. Sau import, chạy drift detection và dùng tên bảng đó làm `ProdTableName`. [Hướng dẫn import của AWS](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/import-resources-manually.html)

## 2. Tạo runtime

Thay CIDR và role ARN minh họa bằng giá trị của bạn. Sửa `DevTableName` và `ProdTableName` nếu dùng các bảng có tên khác. Các lệnh deploy tạo tài nguyên có tính phí trên AWS.

**Linux/macOS**

```sh
region='ap-southeast-2'
admin_cidr='203.0.113.5/32'
admin_role='arn:aws:iam::123456789012:role/TikToEksOperator'

aws cloudformation deploy --region "$region" --stack-name tikto-runtime \
  --template-file ./runtime.yaml --capabilities CAPABILITY_IAM \
  --parameter-overrides "EksAdminCidr=$admin_cidr" "EksAdminRoleArn=$admin_role" \
    DevTableName=tikto-dev-eks-app ProdTableName=tikto-prod-app
```

**Windows PowerShell**

```powershell
$region = 'ap-southeast-2'
$adminCidr = '203.0.113.5/32'
$adminRole = 'arn:aws:iam::123456789012:role/TikToEksOperator'

aws cloudformation deploy --region $region --stack-name tikto-runtime `
  --template-file ./runtime.yaml --capabilities CAPABILITY_IAM `
  --parameter-overrides "EksAdminCidr=$adminCidr" "EksAdminRoleArn=$adminRole" `
    DevTableName=tikto-dev-eks-app ProdTableName=tikto-prod-app
```

Một số tùy chọn thêm vào danh sách `--parameter-overrides`.

- **Chạy riêng Dev** dùng `EnableProdAlb=false` và chỉ triển khai Application Dev. Có thể dùng `NodeCount=2` nếu đủ sức chứa.
- **Backend blue/green** dùng `PreviewRunnerCidr` và `GitHubOidcProviderArn`. Hai preview rule giúp ALB kiểm tra cả slot đang chờ. Dùng stack outputs để cấu hình [GitHub Environment `prod` của Manifest](https://github.com/tikto-calender/manifest#configure-backend-bluegreen).
- **Cập nhật runtime đang chạy Prod** đặt `ProdActiveSlot` bằng giá trị đã xác minh trong `backend-slot` của Manifest. Kiểm tra lại listener sau cập nhật vì CloudFormation có thể áp lại slot được khai báo trong stack.

Xem outputs trên **Linux/macOS hoặc Windows PowerShell**.

```sh
aws cloudformation describe-stacks --region ap-southeast-2 --stack-name tikto-runtime --query 'Stacks[0].Outputs[].[OutputKey,OutputValue]' --output table
```

Runtime mặc định không có NAT gateway, dùng mã hóa dữ liệu Kubernetes do EKS quản lý và tắt Container Insights. ALB mẫu dùng HTTP. Dùng HTTPS nếu đưa lưu lượng thật ra Internet.

## 3. Kết nối EKS và cài controller

Giữ terminal từ bước tạo runtime. Nếu AWS CLI đã đăng nhập bằng đúng role quản trị EKS, có thể bỏ `--role-arn` khi cập nhật kubeconfig.

**Linux/macOS**

```sh
cluster=$(aws cloudformation describe-stacks --region "$region" --stack-name tikto-runtime \
  --query "Stacks[0].Outputs[?OutputKey=='ClusterName'].OutputValue | [0]" --output text)
vpc=$(aws cloudformation describe-stacks --region "$region" --stack-name tikto-runtime \
  --query "Stacks[0].Outputs[?OutputKey=='VpcId'].OutputValue | [0]" --output text)
aws eks update-kubeconfig --region "$region" --name "$cluster" --role-arn "$admin_role"
```

**Windows PowerShell**

```powershell
$cluster = aws cloudformation describe-stacks --region $region --stack-name tikto-runtime `
  --query "Stacks[0].Outputs[?OutputKey=='ClusterName'].OutputValue | [0]" --output text
$vpc = aws cloudformation describe-stacks --region $region --stack-name tikto-runtime `
  --query "Stacks[0].Outputs[?OutputKey=='VpcId'].OutputValue | [0]" --output text
aws eks update-kubeconfig --region $region --name $cluster --role-arn $adminRole
```

Kiểm tra context trỏ vào cụm vừa tạo, rồi chạy các lệnh dưới đây trên **Linux/macOS hoặc Windows PowerShell**. Runtime đã cấu hình Pod Identity cho controller và các service account của Backend.

```sh
kubectl config current-context
kubectl get nodes
kubectl apply -f ../manifest/bootstrap/namespaces.yaml
helm repo add eks https://aws.github.io/eks-charts
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller --namespace kube-system --version 1.14.0 --wait --set "clusterName=$cluster" --set "region=$region" --set "vpcId=$vpc" --set "serviceAccount.name=aws-load-balancer-controller"
helm upgrade --install argocd argo/argo-cd --namespace argocd --version 9.4.0 --wait --timeout 10m -f ../manifest/bootstrap/argocd-values.yaml
kubectl wait --for=condition=Established crd/targetgroupbindings.elbv2.k8s.aws --timeout=120s
```

Tiếp theo, áp dụng Application của các môi trường đã bật theo [hướng dẫn Manifest](https://github.com/tikto-calender/manifest#deploy-with-argo-cd).

## 4. Kiểm tra qua ALB

Sau khi Argo CD báo `Synced` và `Healthy`, lấy URL từ output `DevUrl`. Nếu bật Prod, lặp lại với `ProdUrl`.

**Linux/macOS**

```sh
url=$(aws cloudformation describe-stacks --region "$region" --stack-name tikto-runtime \
  --query "Stacks[0].Outputs[?OutputKey=='DevUrl'].OutputValue | [0]" --output text)
curl -fsS -o /dev/null -w '%{http_code}\n' "$url/"
curl -fsS "$url/api/health"
curl -fsS "$url/api/status"
```

**Windows PowerShell**

```powershell
$url = aws cloudformation describe-stacks --region $region --stack-name tikto-runtime `
  --query "Stacks[0].Outputs[?OutputKey=='DevUrl'].OutputValue | [0]" --output text
(Invoke-WebRequest -UseBasicParsing "$url/").StatusCode
Invoke-RestMethod "$url/api/health"
Invoke-RestMethod "$url/api/status"
```

Kỳ vọng trang web trả HTTP 200, `/api/health` báo `ok` và `/api/status` hiển thị trạng thái DynamoDB cùng Notification Service.

## Xóa runtime khi dùng xong

Việc xóa runtime làm ứng dụng ngừng phục vụ. Xác nhận đúng kube context và chỉ chạy các lệnh theo namespace đã triển khai. Với tài nguyên bổ sung như Ingress, LoadBalancer Service hoặc PVC, dọn chúng trong lúc controller vẫn chạy để AWS có thể xóa tài nguyên phụ thuộc.

**Linux/macOS và Windows PowerShell**

```sh
kubectl config current-context
kubectl -n argocd delete application tikto-dev tikto-prod --ignore-not-found
kubectl -n tikto-dev delete targetgroupbinding backend frontend --ignore-not-found --timeout=3m
kubectl -n tikto-prod delete targetgroupbinding backend backend-green frontend --ignore-not-found --timeout=3m
aws cloudformation delete-stack --region ap-southeast-2 --stack-name tikto-runtime
aws cloudformation wait stack-delete-complete --region ap-southeast-2 --stack-name tikto-runtime
```

Giữ stack dữ liệu để dùng lại. Nếu stack runtime xóa thất bại, xem stack events trước khi thử lại. Kiểm tra các ALB, ổ đĩa và log được tạo hoặc giữ riêng nếu muốn kết thúc chi phí.
