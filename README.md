# Jenkins Single Kubernetes Cluster

Helm chart นี้ใช้เตรียมสิทธิ์ Kubernetes ให้ Jenkins controller หรือ Jenkins Kubernetes plugin สามารถสร้างและควบคุม agent pod ภายใน namespace ที่ติดตั้ง chart โดยสร้างเฉพาะ `ServiceAccount`, `Role` และ `RoleBinding`

**สรุป:** chart นี้ไม่ใช่การติดตั้ง Jenkins agent โดยตรง และไม่ได้สร้าง Jenkins controller, agent pod, Deployment หรือ container image ใด ๆ แต่เป็นส่วนประกอบด้าน RBAC ที่ Jenkins ใช้เมื่อต้องการสร้าง agent แบบชั่วคราวบน Kubernetes

## โครงสร้างโปรเจกต์

```text
helm/
└── chart/
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
        ├── role.yaml
        ├── rolebinding.yaml
        └── serviceaccount.yaml
```

## ข้อมูลของ Chart

| รายการ | ค่า |
| --- | --- |
| Chart name | `jenkins-slave` |
| Chart type | `application` |
| Chart version | `0.1.0` |
| App version | `1.16.0` |
| ค่า `team` เริ่มต้น | `test` |

ค่า `appVersion` เป็น metadata ของ chart เท่านั้น ปัจจุบัน chart ไม่มีการ deploy application หรือกำหนด Jenkins agent image

## ทรัพยากรที่สร้าง

ทรัพยากรทั้งหมดเป็นสิทธิ์แบบ namespace-scoped สำหรับ Jenkins โดย chart ไม่ได้สร้าง workload สำหรับรัน Jenkins หรือ agent

### ServiceAccount

สร้าง ServiceAccount ชื่อ `jenkins-<team>` เช่น `jenkins-test` เมื่อกำหนดค่า `team` โดย ServiceAccount นี้ตั้งใจให้ Jenkins ใช้ขอสิทธิ์ใน namespace ที่ติดตั้ง chart

### Role

สร้าง Role ชื่อ `jenkins-<team>` พร้อมสิทธิ์ดังนี้

- `pods`: `create`, `delete`, `get`, `list`, `patch`, `update`, `watch`
- `pods/exec`: `create`, `delete`, `get`, `list`, `patch`, `update`, `watch`
- `pods/log`: `get`, `list`, `watch`
- `secrets`: `get`

Role นี้เป็น namespace-scoped จึงไม่ให้สิทธิ์ข้าม namespace

### RoleBinding

สร้าง RoleBinding ชื่อ `jenkins-<team>` เพื่อผูก ServiceAccount กับ Role ใน namespace เดียวกัน

## ข้อกำหนดก่อนใช้งาน

- ติดตั้ง `kubectl` และ `helm`
- มี Kubernetes cluster ที่เข้าถึงได้จาก context ปัจจุบัน
- ตรวจสอบว่า context ชี้ไปยัง cluster ที่ต้องการใช้งานด้วย `kubectl config current-context`
- ไฟล์ metadata ของ chart ต้องชื่อ `helm/chart/Chart.yaml` ตาม convention ของ Helm

## ขั้นตอนการใช้งาน

### 1. ตรวจสอบไฟล์ที่ Helm จะ render

```bash
helm lint ./helm/chart
helm template jenkins ./helm/chart --namespace jenkins
```

หากต้องการใช้ชื่อทีมอื่น ให้ส่งค่าผ่าน `--set`:

```bash
helm template jenkins ./helm/chart \
  --namespace jenkins \
  --set team=platform
```

### 2. สร้าง namespace (ถ้ายังไม่มี)

```bash
kubectl create namespace jenkins
```

คำสั่งนี้จะแจ้งว่า namespace มีอยู่แล้วหากถูกสร้างไว้ก่อนหน้า

### 3. ติดตั้ง Chart

```bash
helm upgrade --install jenkins ./helm/chart \
  --namespace jenkins \
  --create-namespace \
  --set team=test
```

สำหรับการใช้งานจริง ควรเก็บค่าไว้ในไฟล์ เช่น `jenkins-values.yaml` แล้วใช้ `-f` แทนการส่งค่าบน command line:

```yaml
team: platform
```

```bash
helm upgrade --install jenkins ./helm/chart \
  --namespace jenkins \
  --create-namespace \
  -f jenkins-values.yaml
```

### 4. ตรวจสอบผลลัพธ์

```bash
helm status jenkins --namespace jenkins
kubectl get serviceaccount,role,rolebinding --namespace jenkins
kubectl describe role jenkins-test --namespace jenkins
```

ตรวจสอบว่า ServiceAccount มีสิทธิ์ตามที่ต้องการด้วย:

```bash
kubectl auth can-i create pods \
  --as=system:serviceaccount:jenkins:jenkins-test \
  --namespace jenkins
```

### 5. อัปเดตหรือลบการติดตั้ง

อัปเดต release ด้วยคำสั่งเดิม `helm upgrade --install` และเปลี่ยนค่าในไฟล์หรือ `--set` ตามต้องการ:

```bash
helm upgrade --install jenkins ./helm/chart \
  --namespace jenkins \
  -f jenkins-values.yaml
```

ลบ release และทรัพยากรที่ Helm สร้าง:

```bash
helm uninstall jenkins --namespace jenkins
```

## การตรวจสอบค่า `team`

ค่า `team` ใช้เป็นส่วนหนึ่งของชื่อทรัพยากร และต้องมีค่าเสมอ หากส่งค่าว่าง Helm จะหยุด render พร้อมข้อความ `team must be set` เพื่อป้องกันการสร้าง resource ที่ไม่มีชื่อ:

```bash
helm template jenkins ./helm/chart --set team=""
```

ทรัพยากรทั้งหมดใช้ชื่อเดียวกันตามรูปแบบ `jenkins-<team>` และใช้ `rbac.authorization.k8s.io/v1` ซึ่งรองรับใน Kubernetes รุ่นปัจจุบัน

## การถอนการติดตั้ง

คำสั่ง `helm uninstall` จะลบทรัพยากรที่ release นี้สร้าง แต่ไม่ลบ namespace:

```bash
helm uninstall jenkins --namespace jenkins
```

หากต้องการลบ namespace ด้วย ให้ตรวจสอบทรัพยากรภายใน namespace ก่อน แล้วจึงใช้:

```bash
kubectl delete namespace jenkins
```