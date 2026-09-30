# Poly chart

- Poly là chart dùng chung, các project khai báo nó như dependency và chỉ viết values.yaml, không phải tự viết template.

- Vấn đề nó giải quyết: nếu không có poly, mỗi project phải copy đủ Deployment, Service, Ingress, HPA..

- Cách nó gom nhiều app trong một chart: mỗi key cấp 1 trong values là một app, mỗi app gộp default với config riêng.

# Các file helper của chart poly (v3)

## 1. Tổng quan

Các file helper là "linh kiện" để các template resource (`deployment.yaml`, `service.yaml`...) lắp lại. Luồng chạy:

```
main.yaml
  ├─ với mỗi app: gộp default + config app → đặt vào $.currentApp
  └─ gọi template resource (ví dụ poly.v3.deployment)
        ├─ commonMetadata   (_helpers)      → name + labels
        ├─ labelSelector    (_helpers)      → selector
        ├─ updateStrategy   (_helpers)      → strategy
        └─ podTemplate      (_podtemplate)  → phần Pod bên trong
              ├─ vault.annotations (_vault)
              ├─ container          → container chính / init container
              └─ pod.volumes, pod.schedule
```

| File | Vai trò |
|---|---|
| `_helpers.yaml` | Tên, label, chọn apiVersion theo bản Kubernetes, vài bộ phân tích chuỗi cho service/ingress |
| `_podtemplate.yaml` | Dựng phần Pod và container |
| `_vault.yaml` | Annotation cho Vault agent injector |
| `_jobspec.yaml` | Spec dùng chung cho Job và CronJob |

Một điểm xuyên suốt: poly dùng **"mini-DSL" dạng chuỗi ngăn bằng `:`** để values ngắn gọn, ví dụ `"15:15:10:30:1"` hay `/:Prefix:svc`. Nhiều helper chỉ làm một việc: cắt chuỗi đó ra rồi điền giá trị mặc định cho phần bị bỏ trống.

## 2. `_helpers.yaml`

### Tên và label

- `name`: tên app (key trong values, hoặc `nameOverride`).
- `fullname`: dùng `fullnameOverride` nếu có; nếu không thì `<release>-<name>` (nếu tên release đã chứa name thì dùng luôn tên release), cắt tối đa 63 ký tự.
- `labels`: gồm `helm.sh/chart`, `extraLabels`, `selectorLabels`, version, managed-by.
- `selectorLabels`: `app.kubernetes.io/name`, `app.kubernetes.io/instance` và `extraSelectorLabels`.
- `tpl-dict`: in ra `key: value`, giá trị có thể là template (chạy qua `tpl`).
- `commonMetadata`: gộp name + labels, dùng ở gần như mọi resource.

### Chọn apiVersion

`hpa`, `cj`, `ingress`, `istio`: so sánh `KubeVersion` để chọn `autoscaling/v2`, `batch/v1`, `networking.k8s.io/v1`... cho cluster cũ và mới.

### Phân tích chuỗi

| Helper | Định dạng chuỗi |
|---|---|
| `updateStrategy` | `"RollingUpdate:25%:25%"` → type, maxSurge, maxUnavailable |
| `svc.port` | `port:targetPort:protocol:name:nodePort` |
| `ingress.path` | `path:pathType:serviceName:servicePort` |

### Tên resource

`serviceAccountName`, `ingressName`, `prefixName` (thêm tên release vào trước nếu tên bắt đầu bằng `-`).

### Lưu ý

Ba helper `poly.v3.env`, `poly.v3.probes`, `poly.v3.lifecycleHooks` được định nghĩa nhưng không nơi nào trong poly gọi. Có thể sẽ gọi ở project cha.

## 3. `_podtemplate.yaml`

### `podTemplate`

Dựng phần Pod, theo thứ tự:

1. `metadata`: name + labels, nhãn của Vault injector, annotation (gộp annotation Vault với `podAnnotations`).
2. `containers`: container chính (qua helper `container`), rồi `extraContainers`.
3. `ephemeralContainers`, `initContainers` (nếu bật) và `extraInitContainers`.
4. `imagePullSecrets`, `hostAliases`, `volumes`.
5. `schedule` (nodeSelector, affinity, tolerations), `podLifecycle` (restartPolicy...), `serviceAccountName`, `podExtraConfigs`.

### `container`

Dựng một container: tên, `image` (`repository` hoặc `quickConfigs.registry` + `tag`), `pullPolicy`, rồi gọi các helper con bên dưới, và `resources`. Với init container (`mode = init`) thì bỏ lifecycle và probes.

### Các helper con của container

| Helper | Làm gì |
|---|---|
| `container.cmd` | Nếu Vault bật thì lấy `command`/`args` từ `vault.runContainer`, không thì từ chính app |
| `container.ports` | Nhận chuỗi `containerPort:name:protocol` hoặc map; mặc định là `quickConfigs.port` |
| `container.env` | `KEY: value`, hoặc chuỗi có tiền tố: `cfm:key:name:optional` (ConfigMap), `scr:key:name:optional` (Secret), `fr:fieldPath:apiVersion` (fieldRef), `rfr:resource:containerName:divisor` (resourceFieldRef) |
| `container.envFrom` | `cfm:name:optional:prefix` (hoặc `scr:`) hoặc map; tên bắt đầu `-` sẽ được thêm tên release |
| `container.lifecycle` | `lifecycleOverride`, hoặc `postStart`/`preStop` qua `lifecycleHandler` (mặc định `preStop` là `sleep 30`) |
| `lifecycleHandler` | Dựng `exec`, `httpGet` (`path:port:host:scheme`) hoặc `tcpSocket` (`host:port`) |
| `container.probes` và `probe` | Dựng liveness/readiness/startup; `params` là `"initialDelay:period:timeout:failure:success"` (mặc định 15/15/10/30/1); có thể dùng `grpc` |
| `container.volumeMounts` | Chuỗi `name:mountPath:readOnly:subPath:mountPropagation` hoặc map, cộng thêm `pvcs` có `mountPath` và `customVolumeMounts` |

### Volumes và scheduling

- `pod.volumes`: gộp `volumes` với các `pvcs` có `mountPath`.
- `pod.schedule`: in nguyên `schedule` bằng `toYaml`.

### Vòng `fromYaml` → `toYaml`

Trong `podTemplate`, container được dựng như sau:

```yaml
containers: {{- toYaml (list (fromYaml (include "poly.v3.container" (dict ...)))) | nindent 4 }}
```

Các bước, từ trong ra ngoài:

1. `include "poly.v3.container"` trả về một đoạn chữ. Đoạn này do nhiều `{{ if }}` ghép nên có dòng trống và thụt lề không đều.
2. `fromYaml` đọc đoạn chữ thành dữ liệu (map), bỏ hết dòng trống và thụt lề lộn xộn.
3. `list (...)` bọc map thành danh sách, vì `containers:` là một danh sách.
4. `toYaml` in ra YAML mới, sạch, thụt lề chuẩn, key sắp theo alphabet.
5. `nindent 4` thụt cả đoạn vào cho khớp vị trí trong Pod template.

Đây là bước "làm sạch": nhờ nó mà các helper con không cần in text thật đẹp. Nếu bỏ đi thì phải chỉnh lại thụt lề từng helper con, và thứ tự key trong output đổi nên khó `diff` với bản cũ. Lợi ích tốc độ gần như bằng 0 nên không đáng bỏ.

## 4. `_vault.yaml`

- **`vault.annotations`**: nếu `vault.enabled`, thêm annotation cho Vault agent injector (role, auth path, server, CA cert, namespace).
  - `template.type = annotation`: nhúng luôn template secret vào annotation.
  - Loại khác: dùng `agent-configmap` trỏ tới ConfigMap do `vault.configmap.yaml` tạo.
  - `extraTemplates` thêm các secret phụ.
- **`vault.cfmName`**: tên ConfigMap từ `template.name` (đổi `.` thành `-`, thêm tên release nếu bắt đầu `-`).

## 5. `_jobspec.yaml`

Chỉ phục vụ Job và CronJob. Nội dung rất ngắn: `template:` là `podTemplate`, sau đó thêm các field từ `workloadCommonConfigs.job` (`backoffLimit`, `ttlSecondsAfterFinished`...).

### Vì sao Job và CronJob dùng chung một helper

Hai loại này cần phần spec giống hệt nhau, chỉ khác chỗ đặt:

- **Job**: spec nằm trực tiếp dưới `spec:` (`job.yaml`, `nindent 2`).
- **CronJob**: spec nằm dưới `spec.jobTemplate.spec:` (`cronjob.yaml`, `nindent 6`).

Viết một lần trong helper thì cả hai gọi lại, chỉ khác số thụt lề.

### Vì sao tách khỏi `podTemplate`

Hai thứ này ở hai tầng khác nhau:

```
podTemplate  ← dùng bởi Deployment, StatefulSet, DaemonSet, Job, CronJob
   └─ bọc bởi jobSpec (+ backoffLimit...)  ← chỉ Job và CronJob
```

`podTemplate` là phần Pod, dùng chung cho mọi loại workload. `jobSpec` là phần bao quanh Pod template với các field chỉ Job mới có. Nếu nhét các field Job vào `podTemplate` thì Deployment và các loại khác cũng nhận nhầm. Với các loại khác, phần bao quanh nằm thẳng trong `deployment.yaml`, `statefulset.yaml`, `daemonset.yaml` vì mỗi loại một khác, không có gì để dùng chung.

## 6. Các file chính trong `templates/`
 
`templates/` có 3 lớp:
 
1. **`main.yaml`**: bộ điều phối, quyết định app nào được render và render gì.
2. **Workload** (`deployment`, `statefulset`, `daemonset`, `cronjob`, `job`): resource chạy Pod. Mỗi app đúng một loại, theo `deployType`.
3. **Resource đi kèm** (`service`, `ingress`, `hpa`, `configmap`, `pvc`...): bật hoặc tắt riêng cho từng app.
### 6.1. Nhóm quan trọng nhất
 
#### `main.yaml`
 
Lặp qua `.Values`, mỗi key cấp 1 là một app. Với mỗi app `enabled`:
 
1. Bỏ qua các key `default`, `apps`, `defaultApp`.
2. Đổi tên app sang kebab-case, gán vào `name`.
3. Gộp `default` (sao chép sâu) với config app bằng `mustMergeOverwrite`, kết quả đặt vào `$.currentApp`.
4. Gọi lần lượt các template resource theo thứ tự: workload → service → SA → ingress → configmap → pvc → hpa → servicemonitor → rollout → pdb → virtualservice → gateway.
Đây là file duy nhất biết thứ tự và điều kiện (ví dụ PDB không áp cho StatefulSet).
 
#### `deployment.yaml` (mẫu cho các workload)
 
Ghép các helper thành một Deployment: metadata, selector, `podTemplate`, `strategy`. Hai điểm đáng chú ý:
 
- `replicas` chỉ in khi **không** bật autoscaling (để HPA điều khiển), và bằng **0** nếu bật Rollout (vì Argo Rollout sẽ lo số bản).
- Cuối file nối thêm `workloadCommonConfigs.deployment` (`revisionHistoryLimit`, `progressDeadlineSeconds`...) bằng `toYaml`.
Các workload còn lại theo cùng khuôn:
 
- **`statefulset`**: thêm `serviceName` (trỏ tới service cùng tên, nên không được tắt service), dùng `updateStrategy`.
- **`daemonset`**: không có `replicas`.
- **`cronjob`**: `spec` = `workloadCommonConfigs.cronjob` (schedule, concurrencyPolicy...) + `jobTemplate` chứa `jobSpec`.
- **`job`**: `spec` = `jobSpec`.
#### `service.yaml`
 
Chỉ render khi app và service đều `enabled`. Tên là `svc.name` hoặc `fullname`; nếu tên bắt đầu bằng `-` thì ghép sau `fullname`. `type: none` sinh Service headless (`clusterIP: None`). Ports lấy từ `svc.port` (chuỗi kiểu `port:targetPort:protocol:name:nodePort`), mặc định là `quickConfigs.port`. Selector mặc định là `selectorLabels`. Template này được dùng cho service chính, các `extraServices` và service preview của Rollout.
 
#### `ingress.yaml`
 
Render khi `ingress.enabled` hoặc có `quickConfigs.domain` (đặt domain là tự tạo ingress). Tên lấy theo thứ tự: `name` → `domain` → host đầu tiên → `fullname`. Annotation chạy qua `tpl` nên dùng được template. Mặc định `ingressClassName: nginx`, `paths` mặc định `/:Prefix`, và có nhánh cho cluster cũ (`extensions/v1beta1`).
 
#### `rollout.yaml`
 
Tạo Argo Rollout kiểu **blue-green** dùng `workloadRef`: nó trỏ tới Deployment đã có (nên Deployment để `replicas: 0`). Trong cùng file nó cũng tự render **Service preview** và **Ingress preview** (rewrite đường dẫn `/preview`), cộng `postPromotionAnalysis` nếu bật `dynamicConfigMap`. Vì nhiều thứ dồn vào một file, đây là chỗ cần cẩn thận nhất khi sửa.
 
### 6.2. Nhóm quan trọng vừa
 
| File | Làm gì |
|---|---|
| `hpa.yaml` | Chỉ cho Deployment và StatefulSet; target là Rollout nếu bật Rollout; metrics CPU + memory (Utilization); có `behavior` cho scaleUp/scaleDown ở cluster ≥ 1.18 |
| `configmap.yaml` | Lặp qua `configMaps`, chỉ tạo cái có `create: true`; `isFileType` in mỗi key thành khối văn bản nhiều dòng |
| `vault.configmap.yaml` | Khi Vault dùng kiểu `configmap`: tạo ConfigMap chứa `config-init.hcl` cho Vault agent (hoặc lấy nguyên `contentFull`) |
| `pvc.yaml` | Tạo PVC cho mỗi mục trong `pvcs` không đánh `exists`; `size` mặc định `1Gi`, accessMode mặc định `ReadWriteOnce`. Pod tự mount nếu có `mountPath` |
| `serviceaccount.yaml` | Tạo SA khi `create: true` và chưa có `existingServiceAccount` |
| `poddisruptionbudget.yaml` | `minAvailable` mặc định 1; selector mặc định là `selectorLabels`; hỗ trợ `extraPdb` |
 
### 6.3. Nhóm ít quan trọng
 
| File | Ghi chú |
|---|---|
| `servicemonitor.yaml` | ServiceMonitor của Prometheus; `endpoints` và `extraConfig` |
| `virtualservice.yaml`, `gateway.yaml` | Istio, chỉ render khi `enabled`; hỗ trợ `extraVS`, `extraGW` |
| `NOTES.txt` | Chỉ in logo poly và thông tin maintainer sau khi cài |
 
### 6.4. Điểm cần nhớ khi đọc
 
- Mỗi template resource tự kiểm tra điều kiện (`enabled`...) bên trong. `main.yaml` gọi hết, template nào không cần sẽ không in gì.
- Resource nào có phiên bản "extra" (service, ingress, pdb, virtualservice, gateway) thì `main.yaml` phải gọi lặp; đó là phần đã gom bằng helper `renderWithExtras`.
- Trạng thái chuyền qua `$.currentApp` (`currentSvc`, `currentIngress`...), không qua tham số. `rollout.yaml` cũng ghi đè các field này khi render service và ingress preview.


## 7. Các thay đổi tối ưu poly
 
### 7.1. Ghim version dependency poly
 
**Vấn đề:** README của poly hướng dẫn khai báo dependency ở chart project với khoảng version `"*.*.*"` (lấy bản mới nhất):
 
```yaml
dependencies:
  - name: poly
    repository: oci://registry.ftech.ai/is-chart
    version: "*.*.*"
```
 
**Thay đổi:** thay bằng version cụ thể (ví dụ `version: "3.0.3"`) và commit `Chart.lock` (sinh bằng `helm dependency update`) vào repo của chart project. Muốn dùng bản poly mới thì sửa version và chạy lại `helm dependency update`.
 
**Tác dụng với ArgoCD:**
 
1. **Bỏ bước tìm "bản mới nhất" mỗi lần render.** Với khoảng version, Helm phải lấy danh sách tag trên OCI registry để chọn bản phù hợp. Khi repo có `Chart.lock`, ArgoCD chạy `helm dependency build` và kéo đúng bản đã khóa. Nếu không có lock thì lệnh chạy giống `helm dependency update`, phải resolve lại. Chart poly vẫn cần được tải, nhưng không còn bước dò bản mới.
2. **Output ổn định.** Với `*.*.*`, khi có bản poly mới trên registry, project có thể nhận bản đó ở lần render sau mà không ai sửa chart project, làm manifest đổi bất ngờ và app bị OutOfSync. Khi ghim, thay đổi ở poly chỉ ảnh hưởng project nào chủ động nâng version.
3. **An toàn khi sửa poly.** Nhờ vậy các thay đổi ở poly (như refactor ở mục 7.2) có thể đưa lên từng project một và kiểm tra bằng `helm template` trước khi nâng.
Mức nhanh hơn cụ thể chưa được đo; có thể so thời gian refresh và sync trong log của repo-server trước và sau khi ghim.
 
### 7.2. Refactor `main.yaml`: gom 5 block lặp thành helper `renderWithExtras`
 
Trước đây `main.yaml` lặp 5 lần một pattern cho service, ingress, pdb, virtualservice, gateway:
 
```yaml
{{- $_ := set $.currentApp "currentSvc" $.currentApp.service -}}
{{- include "poly.v3.service" $ -}}
{{- range $svc := $.currentApp.service.extraServices }}
{{- $_ := set $.currentApp "currentSvc" $svc -}}
{{- include "poly.v3.service" $ -}}
{{ end -}}
```
![](./image.png)

Nay mỗi chỗ chỉ còn một dòng gọi helper:
 
```yaml
{{- include "poly.v3.renderWithExtras" (dict "ctx" $ "field" "currentSvc" "main" $.currentApp.service "extras" $.currentApp.service.extraServices "template" "poly.v3.service") -}}
```
 
Helper nằm trong file mới `templates/helpers/v3/_render.yaml` và nhận các tham số:
 
| Tham số | Ý nghĩa |
|---|---|
| `ctx` | root context `$` |
| `field` | tên field của `currentApp` để đặt config (`currentSvc`, `currentIngress`, `currentPdb`, `currentVS`, `currentGateway`) |
| `main` | config chính (`service`, `ingress`...) |
| `extras` | danh sách extra (`extraServices`...) |
| `template` | tên template cần render (`poly.v3.service`...) |
 
Helper làm đúng 3 bước giống code cũ: đặt config chính vào `currentApp.<field>` rồi render; sau đó với mỗi phần tử extra, đặt vào `currentApp.<field>` và render lại.
  
**Lợi ích:** sửa logic "main + extras" ở một chỗ thay vì năm. 
 