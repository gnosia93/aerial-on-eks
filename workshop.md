워크샵 커리큘럼 (2-Day 코스 예시)

세션	주제	실습 내용	아키텍처
* Day 1	AI-Native PHY 설계	Sionna를 이용한 신경망 기반 수신기(Neural Receiver) 설계 및 BER 측정	x86 + GPU
* Day 2	Cloud-Native O-RAN	Graviton 노드에 L2 스택을 올리고, E2E 지연 시간 및 시스템 처리량 분석	x86/GPU + Graviton


#### 통합 인프라: 하이브리드 모드 (Helm Chart 전략) ####
연구원들이 플래그 하나로 "단순 시뮬레이션 모드"와 "O-RAN 시스템 모드"를 전환할 수 있도록 Helm 구성을 제안합니다.

values.yaml (통합 제어)
```
simulation_mode: "ORAN" # "BASIC" 또는 "ORAN" 선택

nodes:
  phy: { arch: "amd64", gpu: true }
  protocol_stack: { arch: "arm64", enabled: true } # ORAN 모드에서만 활성화
```

templates/deployment.yaml (조건부 배포)
```
{{- if eq .Values.simulation_mode "ORAN" }}
# Graviton용 L2/L3 Pod 정의
apiVersion: apps/v1
kind: Deployment
metadata:
  name: oran-du-cu-stack
spec:
  template:
    spec:
      nodeSelector:
        kubernetes.io/arch: arm64
      containers:
      - name: protocol-engine
        image: custom-l2-stack:arm64
{{- end }}

```

### 연구원을 위한 성능 모니터링 (Point-to-Point) ###
연구원들이 각 계층의 병목을 시각적으로 확인하도록 Grafana 대시보드에 두 가지 지표를 띄워줍니다.
* PHY Metric (L1): Eb/No 대비 BER 곡선. (논문용 데이터)
쿼리: sionna_ber_value{mode="neural_rx"}

* System Metric (L2/L3): Graviton CPU 사용량 vs 패킷 처리 지연(Latency). (시스템 최적화 데이터)
쿼리: avg(node_cpu_seconds_total{arch="arm64"})


연구원 대상 가이드 포인트
* 연구 가치 강조: "Sionna로 물리 계층만 보는 것이 아니라, AWS Graviton을 통해 실제 5G/6G 기지국의 가상화(vRAN) 비용 효율성을 직접 시뮬레이션할 수 있습니다."
* 멀티 아키텍처 실무: 대학원생들이 나중에 기업 연구소에 갔을 때 필수적인 Docker buildx를 이용한 멀티 아키텍처 빌드 과정을 실습에 포함하세요.



