-----

# K8s 모니터링 및 자동 스케일링 알림 스택 (OpenStack 이전 가이드)

이 저장소는 Kubernetes 클러스터의 리소스를 모니터링하고, 특정 조건(예: CPU 임계치 초과)이 충족될 때 Jenkins 웹훅을 호출하여 자동 스케일링을 트리거하는 모니터링 스택을 배포하기 위한 YAML 파일들을 포함합니다.

Ubuntu 환경에서 OpenStack 환경으로 마이그레이션하는 것을 목표로 하며,  **제공된 파일 순서대로 배포하는 것을 권장합니다.**

## 🚀 아키텍처 개요

이 스택은 다음과 같은 구성 요소로 이루어집니다.

1.  **Node Exporter:** 각 K8s 노드에서 하드웨어 및 OS 메트릭을 수집합니다.
2.  **Prometheus:** Node Exporter 등으로부터 메트릭을 수집(Scrape)하고 저장합니다. 또한, 정의된 알림 규칙(Alert Rule)을 평가합니다.
3.  **Alertmanager:** Prometheus로부터 알림을 받아 중복을 제거하고, 그룹화한 뒤, 설정된 수신자(Receiver)에게 알림을 보냅니다.
4.  **Grafana:** Prometheus에 저장된 데이터를 시각화하는 대시보드 툴입니다.

알림 흐름은 다음과 같습니다.
**Node Exporter (메트릭 제공) → Prometheus (수집 및 알림 규칙 평가) → Alertmanager (알림 라우팅) → Jenkins (웹훅 수신 및 조치)**

-----

## migration="" check-list="" (오픈스택="" 이전="" 시="" 필수="" 확인)=""\>

> ⚠️ **중요:** 새 OpenStack 클러스터에 배포하기 전에, **반드시** 다음 항목들을 확인하고 수정해야 합니다.

  * [ ] **`3-configmaps.yml` 파일**

      * `alertmanager-config` ConfigMap 내부의 `alertmanager.yml` 내용을 확인합니다.
      * `url: 'http://<jenkins_ip>:8080/job/AutoHealingJob/buildWithParameters?token=YOUR_TOKEN'` 부분에서,
          * `<jenkins_ip>`를 실제 운영 환경의 Jenkins 서버 IP 주소 또는 도메인으로 변경하세요.
          * `YOUR_TOKEN`을 Jenkins 파이프라인에 설정된 토큰으로 변경하세요.

  * [ ] **`4-storage-pvc.yml` 파일**

      * `prometheus-pvc`와 `grafana-pvc`의 `storageClassName: local-path` 설정을 확인합니다.
      * OpenStack 환경에서는 Cinder를 스토리지로 사용하는 경우가 많습니다. 클러스터 관리자에게 사용 가능한 **StorageClass** 이름을 확인하고, `storageClassName` 값을 해당 이름(예: `cinder-sc`, `standard` 등)으로 변경하세요.
      * 이 부분을 수정하지 않으면 PVC가 `Pending` 상태에 머물러 Prometheus와 Grafana 파드가 시작되지 않습니다.

  * [ ] **`6-prom-svc.yml` (또는 서비스 파일)**

      * `prometheus-service`와 `grafana-service`가 `type: NodePort`로 설정되어 있습니다.
      * OpenStack 환경에서는 `type: LoadBalancer`로 변경하여 OpenStack의 로드밸런서를 통해 외부 IP를 할당받는 것을 고려할 수 있습니다. 이는 운영 정책에 따라 선택하세요.

-----

## 🛠️ 배포 순서 및 파일 설명

파일은 의존성 순서에 가깝게 정렬되어 있습니다. 아래 순서대로 배포하는 것을 권장합니다.

### 1\. `1-node-exporter.yml`

  * **배포 대상:** Node Exporter (메트릭 수집기)
  * **포함된 리소스:**
      * `DaemonSet`: 클러스터의 **모든 노드에** `node-exporter` 파드를 하나씩 실행하여, 각 노드의 CPU, 메모리 등 하드웨어/OS 메트릭을 수집합니다.
      * `Service`: Prometheus가 `node-exporter` 파드들을 쉽게 찾아서 메트릭을 가져갈 수 있도록 `metrics` 포트를 노출합니다.

### 2\. `2-prometheus-rbac.yml`

  * **배포 대상:** Prometheus용 권한 설정 (RBAC)
  * **포함된 리소스:**
      * `ServiceAccount (prometheus-sa)`: Prometheus 파드가 사용할 K8s 내의 "계정"입니다.
      * `ClusterRole (prometheus-role)`: K8s 클러스터 전역에서 노드(nodes), 파드(pods), 서비스(services) 등의 정보를 조회할 수 있는 "권한 목록"을 정의합니다.
      * `ClusterRoleBinding (prometheus-rb)`: 위에서 만든 `prometheus-sa` 계정에 `prometheus-role` 권한을 "연결"해줍니다.
  * **필요 이유:** 이 권한이 있어야 Prometheus가 K8s API에 접근하여 `node-exporter` 파드나 다른 서비스들을 자동으로 찾아낼 수 있습니다(Service Discovery).

### 3\. `3-configmaps.yml`

  * **배포 대상:** Prometheus 및 Alertmanager 설정 파일
  * **포함된 리소스:**
      * `ConfigMap (prometheus-config)`:
          * `prometheus.yml`: Prometheus의 핵심 설정. `node-exporter`를 자동으로 찾아내도록 `kubernetes_sd_configs`가 설정되어 있습니다.
          * `alerts.yml`: **알림 규칙** 정의. (예: 5분간 CPU 80% 초과 시 `HighCpuUsageAutoScalingNeeded` 알림 발생)
      * `ConfigMap (alertmanager-config)`:
          * `alertmanager.yml`: 알림 라우팅 규칙 정의. `critical-auto` 심각도 알림을 `jenkins-autoscale` 수신자(웹훅)로 보내도록 설정합니다.
          * ⚠️ **(마이그레이션 확인\!)** `jenkins-autoscale` 수신자의 `url`을 실제 Jenkins 주소로 변경해야 합니다.

### 4\. `4-storage-pvc.yml`

  * **배포 대상:** Prometheus, Grafana 데이터용 영구 저장 공간
  * **포함된 리소스:**
      * `PersistentVolumeClaim (prometheus-pvc)`: Prometheus가 수집한 시계열 데이터를 저장할 10Gi 공간을 요청합니다.
      * `PersistentVolumeClaim (grafana-pvc)`: Grafana 대시보드 및 설정을 저장할 5Gi 공간을 요청합니다.
  * **필요 이유:** 파드가 재시작되어도 수집한 데이터나 설정한 대시보드가 사라지지 않도록 영구적으로 보관합니다.
  * ⚠️ **(마이그레이션 확인\!)** `storageClassName`을 OpenStack 환경에 맞게 수정해야 합니다.

### 5\. `5-deployments.yml`

  * **배포 대상:** 핵심 애플리케이션 배포
  * **포함된 리소스:**
      * `Deployment (prometheus)`: Prometheus 애플리케이션을 실행합니다. `2-prometheus-rbac.yml`에서 만든 `prometheus-sa` 계정을 사용하고, `3-configmaps.yml`와 `4-storage-pvc.yml`에서 만든 설정/스토리지를 마운트합니다.
      * `Deployment (alertmanager)`: Alertmanager 애플리케이션을 실행합니다. `3-configmaps.yml`의 `alertmanager-config`를 마운트합니다.
      * `Deployment (grafana)`: Grafana 애플리케이션을 실행합니다. `4-storage-pvc.yml`의 `grafana-pvc`를 마운트합니다.

### 6\. `6-prom-svc.yml` (서비스 파일)

  * **배포 대상:** 애플리케이션 외부/내부 노출
  * **포함된 리소스:** (파일명과 달리 여러 서비스가 포함될 수 있습니다.)
      * `Service (prometheus-service)`: Prometheus 웹 UI 및 API를 외부에 노출합니다. (`type: NodePort` 30090)
      * `Service (alertmanager-service)`: Prometheus가 Alertmanager와 통신할 때 사용하는 **내부** 서비스입니다. (`type: ClusterIP`)
      * `Service (grafana-service)`: Grafana 웹 UI를 외부에 노출합니다. (`type: NodePort` 30000)
  * ⚠️ **(마이그레이션 확인\!)** `NodePort` 대신 `LoadBalancer` 타입 사용을 고려할 수 있습니다.

-----

## 💻 배포 및 접근 방법

### 배포 명령어

터미널에서 이 파일들이 있는 디렉토리로 이동한 후, 순서대로 적용합니다.

```bash
# 1. Node Exporter 배포
kubectl apply -f 1-node-exporter.yml

# 2. Prometheus 권한 설정
kubectl apply -f 2-prometheus-rbac.yml

# 3. 설정 파일 배포 (Jenkins URL 수정 확인!)
kubectl apply -f 3-configmaps.yml

# 4. 저장 공간 요청 (StorageClass 수정 확인!)
kubectl apply -f 4-storage-pvc.yml

# 5. 애플리케이션 배포
kubectl apply -f 5-deployments.yml

# 6. 서비스 노출
kubectl apply -f 6-prom-svc.yml

# (선택) 전체 배포 확인
kubectl get all
```

### 서비스 접근

배포가 완료되고 파드들이 `Running` 상태가 되면, 다음 주소로 각 서비스에 접근할 수 있습니다. (`<OpenStack_Node_IP>`는 K8s 워커 노드 중 하나의 공인 또는 사설 IP입니다.)

  * **Grafana:** `http://<OpenStack_Node_IP>:30000` (기본 ID/PW: admin/admin)
  * **Prometheus:** `http://<OpenStack_Node_IP>:30090` (여기서 메트릭 수집 대상(Targets)과 알림 규칙(Alerts) 상태 확인 가능)
