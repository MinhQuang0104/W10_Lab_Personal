# 🔐 Runbook: Xoay vòng Secret (Secrets Rotation)

## Mục đích
Hướng dẫn xoay vòng mật khẩu Database trên AWS Secrets Manager mà không làm gián đoạn hệ thống.

---

## 📋 Điều kiện tiên quyết
- ✅ Quyền truy cập AWS Secrets Manager
- ✅ Quyền kubectl trên K8s cluster
- ✅ ExternalSecret đã được deploy: `db-creds` trong namespace `demo`

---

## 🔄 Quy trình Xoay vòng Secret

### **Bước 1: Cập nhật mật khẩu mới trên AWS Secrets Manager**

#### Option A: Dùng AWS CLI
```bash
# 1. Tạo secret mới
aws secretsmanager update-secret \
  --secret-id demo/db/password \
  --secret-string "new_secure_password_12345" \
  --region us-east-1

# 2. Verify update thành công
aws secretsmanager get-secret-value \
  --secret-id demo/db/password \
  --region us-east-1 | jq '.SecretString'
```

#### Option B: Dùng AWS Console
1. Đăng nhập AWS Console
2. Tìm "AWS Secrets Manager"
3. Chọn secret `demo/db/password`
4. Click "Edit secret"
5. Cập nhật giá trị mới
6. Click "Save"

---

### **Bước 2: Kiểm tra ExternalSecret nhận giá trị mới**

```bash
# Kiểm tra status ExternalSecret
kubectl describe externalsecret db-creds -n demo

# Output sẽ hiển thị:
# Name:         db-creds
# ...
# Status:
#   Conditions:
#   - Type: Ready
#     Status: True
#     Message: ExternalSecret synced successfully
#   Last Sync Time: 2024-06-19T14:23:45Z  <-- Thời gian sync gần nhất
```

**Lưu ý:** ExternalSecret sẽ tự động sync mỗi **1 phút** (`refreshInterval: 1m`).

Nếu muốn force sync ngay lập tức:
```bash
kubectl annotate externalsecret db-creds \
  -n demo \
  force-sync-$(date +%s)=true \
  --overwrite
```

---

### **Bước 3: Xác minh K8s Secret được cập nhật**

```bash
# Xem K8s Secret được tạo bởi ExternalSecret
kubectl get secret db-secret -n demo -o yaml

# Output sẽ hiển thị:
# apiVersion: v1
# kind: Secret
# metadata:
#   name: db-secret
#   namespace: demo
# data:
#   password: bmV3X3NlY3VyZV9wYXNzd29yZF8xMjM0NQ==  # Base64 encoded
```

Giải mã để verify (optional):
```bash
kubectl get secret db-secret -n demo -o jsonpath='{.data.password}' | base64 -d
# Output: new_secure_password_12345
```

---

### **Bước 4: Kiểm tra ứng dụng đọc mật khẩu mới**

#### Check Pod logs
```bash
# Xem logs app để confirm connect DB thành công
kubectl logs -n demo -l app=api --tail=50 | grep -i "database\|connection"

# Output mong đợi:
# Database connection established successfully
# Connected to PostgreSQL 13.x at db.demo.svc.cluster.local:5432
```

#### Test kết nối DB từ Pod
```bash
# Truy cập shell của pod
kubectl exec -it -n demo pod/api-xxxxx -- /bin/bash

# Test kết nối DB bằng psql hoặc curl
psql -h db.demo.svc.cluster.local -U admin -d mydb -c "SELECT 1"
# Nếu thành công sẽ output: 1
```

---

### **Bước 5: Verify không có pod restart**

```bash
# Kiểm tra restart count
kubectl get pods -n demo -o wide | grep api

# Output ví dụ:
# api-xxxxx   1/1   Running   0          2d14h   10.x.x.x

# Nếu RESTARTS = 0, có nghĩa pod không restart ✅
```

Kiểm tra thời gian pod hoạt động:
```bash
kubectl get pods -n demo -o jsonpath='{.items[0].metadata.creationTimestamp}'
# Nếu timestamp cũ (>1 phút so với giờ hiện tại), pod đã không restart ✅
```

---

## ⚠️ Troubleshooting

| Vấn đề | Nguyên nhân | Cách khắc phục |
|--------|-----------|--------------|
| ExternalSecret status = **False** | AWS credentials sai, hoặc SecretStore không hoạt động | Kiểm tra `kubectl describe secretstore` |
| K8s Secret không update | ExternalSecret sync fail | Xem logs: `kubectl logs -n external-secrets deployment/external-secrets` |
| Pod connect DB fail | App chưa reload mật khẩu mới | Restart pod hoặc check lại secret value |
| Connection timeout | Network policy chặn | Check `kubectl get networkpolicy -n demo` |

---

## ✅ Checklist Hoàn tất
- [ ] Cập nhật secret mới trên AWS
- [ ] ExternalSecret status = **Ready**
- [ ] K8s Secret value được cập nhật
- [ ] Pod logs không có error
- [ ] Pod restart count = 0
- [ ] App kết nối DB thành công

---

## 📞 Liên hệ hỗ trợ
Nếu gặp sự cố, liên hệ DevOps team: `devops@company.com`
