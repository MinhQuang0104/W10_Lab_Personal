# 🧪 Test Gatekeeper Constraints

## 📋 Điều kiện tiên quyết

⚠️ **Hãy đảm bảo:**
- ✅ Đã cài đặt Gatekeeper trên K8s cluster
- ✅ Đã apply tất cả Constraint templates và constraints
- ✅ Tất cả constraints ở chế độ `enforcementAction: deny`

---

## 🚀 Hướng dẫn chạy Test

### 📂 Bước 1: Navigate vào thư mục test-gatekeeper

```bash
# Từ thư mục W10_lab
cd test-gatekeeper
```

Chạy lần lượt các lệnh dưới đây trên terminal:

### ❌ Test 1: Pod dùng image tag `:latest`

```bash
kubectl apply -f test-gatekeeper/test-image-latest.yaml
```

**Kết quả kỳ vọng:**
```
Error from server (Forbidden): error when creating "test-image-latest.yaml":
admission webhook denied the request: ...latest tag not allowed...
```

---

### ❌ Test 2: Pod thiếu `resources.limits`

```bash
kubectl apply -f test-gatekeeper/test-missing-limits.yaml
```

**Kết quả kỳ vọng:**
```
Error from server (Forbidden): error when creating "test-missing-limits.yaml":
admission webhook denied the request: ...resources.limits required...
```

---

### ❌ Test 3: Pod chạy quyền Root

```bash
kubectl apply -f test-gatekeeper/test-root-user.yaml
```

**Kết quả kỳ vọng:**
```
Error from server (Forbidden): error when creating "test-root-user.yaml":
admission webhook denied the request: ...runAsUser: 0 not allowed...
```

---

### ❌ Test 4: Pod bật `hostNetwork`

```bash
kubectl apply -f test-gatekeeper/test-host-network.yaml
```

**Kết quả kỳ vọng:**
```
Error from server (Forbidden): error when creating "test-host-network.yaml":
admission webhook denied the request: violation ["block-host-network"]
...hostNetwork: true not allowed...
```

---

### ✅ Test 5: Pod hợp lệ (PASS)

```bash
kubectl apply -f test-gatekeeper/test-pod-valid.yaml
```

**Kết quả kỳ vọng:**
```
pod/test-pod-valid created
```

---

## 📊 Bảng tóm tắt

| Test # | File | Status | Lý do | Kết quả |
|--------|------|--------|-------|---------|
| 1️⃣ | `test-image-latest.yaml` | ❌ REJECT | Image tag `:latest` | Không cho phép |
| 2️⃣ | `test-missing-limits.yaml` | ❌ REJECT | Thiếu `limits` | Không cho phép |
| 3️⃣ | `test-root-user.yaml` | ❌ REJECT | `runAsUser: 0` | Không cho phép |
| 4️⃣ | `test-host-network.yaml` | ❌ REJECT | `hostNetwork: true` | Không cho phép |
| 5️⃣ | `test-pod-valid.yaml` | ✅ PASS | Tuân thủ tất cả | Pod được tạo |

---

## 💡 Ghi chú

- Nếu Pod không bị REJECT, kiểm tra:
  - Constraint template có được apply chưa?
  - `enforcementAction` có để `deny` hay `dryrun`?
  - Gatekeeper webhook đã enable chưa?

- Xóa test Pod: `kubectl delete pod test-*`