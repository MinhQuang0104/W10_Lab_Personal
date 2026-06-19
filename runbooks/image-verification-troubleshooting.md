# 🔍 Runbook: Xử lý lỗi Admission Controller từ chối Image

## Mục đích
Hướng dẫn khắc phục lỗi khi **Sigstore Policy Controller** chặn deploy vì image chưa ký hoặc ký không hợp lệ.

---

## 📋 Dấu hiệu lỗi

### 1. Lỗi khi chạy `kubectl apply`
```bash
Error from server (Forbidden): error when creating "rollout.yaml":
admission webhook "policy.sigstore.dev" denied the request:
[sigstore] image mismatch: no signature found for 'ghcr.io/minhquang0104/w10-api:0.0.5'
```

### 2. Lỗi khi xem ArgoCD
```
OutOfSync → Degraded
Error: admission webhook denied the request: [sigstore] image ...
```

### 3. Kiểm tra event trên Pod
```bash
kubectl describe pod api-xxxxx -n demo
# Events:
# Normal  Pulling    5s   kubelet  pulling image "ghcr.io/..."
# Warning Failed     3s   kubelet  admission webhook denied the request
```

---

## 🔧 Quy trình Điều tra & Khắc phục

### **Bước 1: Xác nhận lỗi từ chối**

```bash
# Kiểm tra Policy Controller status
kubectl get deployment -n sigstore-systems

# Output mong đợi:
# sigstore-policy-controller-xxx   1/1   Running   0

# Xem logs từ Policy Controller
kubectl logs -n sigstore-systems \
  -l app=sigstore-policy-controller \
  --tail=30 | grep -i "deny\|reject\|mismatch"
```

---

### **Bước 2: Kiểm tra Cosign Signature**

#### 2.1 Verify image đã ký chưa
```bash
# Sử dụng public key để verify
cosign verify --key ./signing/cosign.pub \
  ghcr.io/minhquang0104/w10-api:0.0.5

# Nếu output hiển thị công bố (attestation), có nghĩa image ✅ đã ký
# Nếu error "no signature found", image ❌ chưa ký
```

#### 2.2 Kiểm tra pipeline CI đã chạy bước ký
```bash
# Truy cập GitHub Actions
# Repo → Actions → Tìm run cuối cùng

# Kiểm tra step "Sign Docker Image" - status có là ✅ PASS không?
# Nếu chỉ là ⏭️ Skipped hoặc ❌ FAILED → Pipeline không ký được image
```

---

### **Bước 3: Xác nhận ClusterImagePolicy cấu hình đúng**

```bash
# Xem policy hiện tại
kubectl get clusterimagepolicies

# Output ví dụ:
# NAME            AGE
# image-policy    5d

# Xem chi tiết policy
kubectl get clusterimagepolicy image-policy -o yaml

# Kiểm tra 3 điểm chính:
```

**✅ Điểm 1: Public key khớp không?**
```yaml
spec:
  images:
  - glob: 'ghcr.io/minhquang0104/w10-api:*'
  authorities:
  - name: cosign
    keySecret:
      name: cosign-pub  # Secret chứa public key
      namespace: sigstore-systems
```

Kiểm tra secret:
```bash
kubectl get secret cosign-pub -n sigstore-systems -o yaml | grep 'cosign.pub'
# Public key phải khớp với private key dùng để ký!
```

**✅ Điểm 2: Namespace có label policy.sigstore.dev/include=true không?**
```bash
# Kiểm tra label trên namespace demo
kubectl get namespace demo -o yaml | grep sigstore

# Output mong đợi:
# labels:
#   policy.sigstore.dev/include: "true"

# Nếu thiếu, thêm label:
kubectl label namespace demo policy.sigstore.dev/include=true --overwrite
```

**✅ Điểm 3: Policy enforcement mode là gì?**
```yaml
spec:
  mode: enforce  # "enforce" = chặn, "audit" = chỉ log

# Nếu chỉ là audit, policy không chặn. Change thành "enforce"
```

---

### **Bước 4: Kiểm tra Private Key và Public Key khớp**

```bash
# Trích xuất public key từ private key
cosign public-key --key cosign.key > extracted.pub

# So sánh với public key trong cluster
kubectl get secret cosign-pub -n sigstore-systems \
  -o jsonpath='{.data.cosign\.pub}' | base64 -d > cluster.pub

# So sánh 2 file
diff extracted.pub cluster.pub

# Nếu output empty = khớp ✅
# Nếu khác nhau = public key trong cluster sai ❌
```

---

## 🔨 Các phương án khắc phục

### **Phương án 1: Chạy lại pipeline CI (được recommend)**

```bash
# Trên GitHub
1. Truy cập Repository
2. Actions → build-and-push workflow
3. Click "Run workflow"
4. Branch: main
5. Click "Run workflow"

# Wait cho workflow hoàn tất, kiểm tra step "Sign Docker Image" pass không
```

Sau khi pipeline thành công:
```bash
# Verify image đã ký
cosign verify --key ./signing/cosign.pub \
  ghcr.io/minhquang0104/w10-api:0.0.5

# Deploy lại
kubectl apply -f app-api/rollout.yaml -n demo
```

---

### **Phương án 2: Update ClusterImagePolicy (nếu key khác)**

Nếu đã tạo key pair mới (cosign.key/cosign.pub):

```bash
# 1. Cập nhật secret trong cluster
kubectl create secret generic cosign-pub \
  --from-file=cosign.pub=./signing/cosign.pub \
  -n sigstore-systems \
  --dry-run=client -o yaml | kubectl apply -f -

# 2. Restart policy controller để reload secret
kubectl rollout restart deployment sigstore-policy-controller \
  -n sigstore-systems

# 3. Wait 10 giây, verify
kubectl get deployment -n sigstore-systems
```

---

### **Phương án 3: Tạm thời bypass policy (khẩn cấp)**

⚠️ **Chỉ dùng khi troubleshooting, phải remove sau!**

```bash
# Đánh label exclude trên namespace để bypass policy
kubectl label namespace demo policy.sigstore.dev/include=false --overwrite

# Deploy lại
kubectl apply -f app-api/rollout.yaml -n demo

# Kiểm tra pod running
kubectl get pods -n demo

# ⚠️ SAU ĐÓ: Phải restore label và fix issue gốc!
kubectl label namespace demo policy.sigstore.dev/include=true --overwrite
```

---

## ✅ Verify khắc phục thành công

```bash
# 1. Policy Controller logs không có error
kubectl logs -n sigstore-systems \
  -l app=sigstore-policy-controller | tail -20

# 2. Pod deploy thành công (status = Running)
kubectl get pods -n demo | grep api

# 3. Pod event không có admission deny
kubectl describe pod api-xxxxx -n demo | grep -i "admission\|webhook"

# 4. Image signature verify thành công
cosign verify --key ./signing/cosign.pub \
  ghcr.io/minhquang0104/w10-api:0.0.5
```

---

## 🐛 Troubleshooting nâng cao

| Triệu chứng | Nguyên nhân | Giải pháp |
|-----------|----------|----------|
| `no signature found` | Image chưa ký hoặc repo khác | Chạy lại CI pipeline |
| `key mismatch` | Public key trong cluster sai | Update secret cosign-pub |
| `invalid signature` | Image bị sửa đổi sau ký | Re-sign image hoặc revert |
| Policy Controller không chạy | Pod crash | `kubectl describe pod -n sigstore-systems` |
| Permission denied | Service Account thiếu RBAC | Kiểm tra ClusterRole bindings |

---

## 📞 Liên hệ hỗ trợ
Nếu vẫn không giải quyết được, liên hệ DevOps: `devops@company.com`

**Cung cấp thông tin:**
- Error message đầy đủ
- Image tag đang deploy
- Kết quả của `cosign verify` command
