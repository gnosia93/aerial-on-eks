# aerial-on-eks

![](https://github.com/gnosia93/aerial-on-eks/blob/main/images/workshop-arch-aerial-spot.png)

### _O-RAN(Open Radio Access Network) 아키텍처_ ###

* Sionna GPU Pod -> gRPC -> PyAreal GPU Pod
* Sinnoa GPU Pod -> gRPC -> PyAreal GPU Pod -> Graviton L2/L3 stack Pod
* PyAreal GPU Pod -> Prometheus/Grafna dashboard (BER 측정/시각화)
* 지원 토폴로지 
  * Shared Memory (Zero-Copy Interface / Sionna 와 PyAreal 가 동일 Pod를 사용하는 경우만 지원)
  * UDP (eCPRI / Raw Socket 기반)
  * gRPC TPC/IP  
* Sionna (단말기): 1010 데이터를 무선 신호로 만들어 송신.
* PyAerial (기지국 L1/GPU):
* 신호 복구: 내부의 TensorFlow 모델이 오염된 신호를 받아 1010으로 복구.
* 지표 추출: 복구 성공 여부를 따져서 BER/BLER 계산.
* 외부 노출: Prometheus가 이 값을 긁어갈 수 있도록 코드 레벨에서 데이터 노출.
* Graviton (기지국 L2/L3): PyAerial이 넘겨준 1010 데이터를 받아 실제 네트워크 프로토콜 처리.
* Grafana: Prometheus에 저장된 PyAerial의 BER 데이터를 가져와 시각화

### _Topics_ ###

* [C1. VPC 생성하기](https://github.com/gnosia93/aerial-on-eks/tree/main/tf)

* [C2. EKS 클러스터 생성하기](https://github.com/gnosia93/aerial-on-eks/blob/main/lesson/c2-provision-eks.md)

* [C3. GPU 노드준비 하기](https://github.com/gnosia93/aerial-on-eks/blob/main/lesson/c3-prepare-gpu-node.md)

* [C4. 시그널 제너레이터 배포](https://github.com/gnosia93/aerial-on-eks/blob/main/apps/signal-generator.md)

* [C5. 채널 리시버 배포](https://github.com/gnosia93/aerial-on-eks/blob/main/apps/channel-receiver.md)

* [C6. 그라비톤 기반 L2/L3 스택 구현](https://github.com/gnosia93/aerial-on-eks/blob/main/apps/graviton-L2L3-stack.md)

* [C7. 그라파나 BER 시각화](https://github.com/gnosia93/aerial-on-eks/blob/main/lesson/c7-monitoring.md)

* [C8. Gitlab CI/CD]


### _Appendix_ ###

* [VS Code 노트북 실행하기](https://github.com/gnosia93/aerial-on-eks/blob/main/lesson/c4-vs-code-notebook.md)
* https://ec2spotworkshops.com/ec2-auto-scaling-with-multiple-instance-types-and-purchase-options/spot_resilience.html
