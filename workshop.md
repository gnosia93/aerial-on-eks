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


### 전체 시스템 시퀀스 (Data Flow) ###

* TX Pod (x86 + GPU): Sionna로 신호 생성 → Little-Endian 바이너리 직렬화 → UDP 전송
* L2/L3 Pod (Graviton): UDP 수신 → 패킷 분석 및 스케줄링(L2 로직) → 제어 헤더 부착 → UDP 포워딩
* RX Pod (x86 + GPU): UDP 수신 → 헤더 분리 및 시퀀스 확인 → Sionna RT/Neural RX로 신호 복원

---

결론부터 말씀드리면, 네트워크를 탄다고 해서 데이터(Payload)의 엔디안이 자동으로 바뀌지 않습니다.
연구원들이 가장 흔히 하는 오해 중 하나인데, 아래 3단계로 명확히 구분해서 설명해 주시면 좋습니다.

### 1. 데이터(Payload)는 건드리지 않는다 ###
사용자님이 보낸 Sionna IQ 샘플(복소수 데이터)은 네트워크 장비(스위치, 라우터) 입장에서 그냥 '의미 없는 바이트 덩어리'일 뿐입니다.
네트워크 카드는 데이터를 비트 단위로 순서대로 밀어낼 뿐, "이게 숫자니까 엔디안을 바꿔야지"라고 생각하지 않습니다.
따라서 TX(x86)에서 리틀 엔디안으로 쏜 데이터는 Graviton(ARM)에 도착할 때도 여전히 리틀 엔디안 상태입니다.

### 2. 그럼 '네트워크 바이트 순서'는 뭔가요? ###
네트워크 표준이 빅 엔디안이라는 말은 패킷의 '헤더(Header)'에만 해당합니다.
IP 주소, 포트 번호 등 TCP/IP 헤더 정보는 규격에 따라 빅 엔디안으로 채워져야 라우터들이 읽을 수 있습니다.
하지만 우리가 직접 짜는 데이터 부분(UDP Payload)은 우리가 주인입니다. 리틀 엔디안으로 보내든 빅 엔디안으로 보내든 통신망은 관여하지 않습니다.

### 3. 왜 변환 없이 그냥 보내나요? (성능의 핵심) ###
연구원들에게 이 부분을 강조해 주세요.
* 오버헤드 방지: 수기가비트(Gbps)급의 IQ 데이터를 보낼 때, CPU가 모든 바이트를 뒤집는 연산을 수행하면 지연 시간(Latency)이 급격히 늘어납니다.
* Zero-copy: 현대 통신 시스템(O-RAN, Aerial 등)은 성능 최적화를 위해 호스트 CPU의 엔디안(리틀 엔디안)을 그대로 유지한 채 메모리에서 네트워크 카드로 바로 밀어넣습니다.


---

니다.
3. 우리 워크샵에서 중요한 이유
TX(x86): 리틀 엔디안 방식으로 신호를 생성해 메모리에 적재합니다.
Graviton(ARM): 역시 리틀 엔디안 방식이므로, 네트워크로 날아온 바이트 뭉치를 뒤집지 않고 그대로 읽으면 원본 숫자 0x12345678이 복원됩니다.

---

일반적으로 '중계기'는 물리적인 신호(L1)를 받아서 단순히 증폭하거나 전달하는 장치를 떠올리기 쉽습니다. 하지만 우리가 만든 Graviton Pod은 다음과 같은 고차원적인 작업을 합니다.
* L2 (MAC): 여러 사용자의 데이터를 어떻게 나눠 보낼지 결정(Scheduling).
* L3 (RRC): 단말기의 접속 상태를 관리하고 경로를 지정(Control Plane).
"여기 중간에 위치한 Graviton 기반의 L2/L3 스택은 단순한 중계기가 아닙니다. TX에서 생성된 원시 IQ 데이터를 받아서, 실제 통신 규격에 맞게 헤더를 붙이고 순서를 제어하는 가상화된 기지국(vRAN)의 핵심 제어부입니다."
