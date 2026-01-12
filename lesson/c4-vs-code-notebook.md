EKS 클러스터에서 NVIDIA TensorFlow 컨테이너를 기반으로 VS Code(code-server)와 Jupyter Notebook을 동시에 사용할 수 있는 통합 환경 YAML 파일입니다.
이 설정은 Sionna 실행을 위한 GPU 할당 및 외부 접속을 위한 LoadBalancer 설정을 포함합니다.
📄 EKS 통합 개발 환경 설정 (sionna-dev.yaml)

```
cat << EOF | kubectl apply -f - 
apiVersion: v1
kind: Service
metadata:
  name: sionna-dev-service
spec:
  type: LoadBalancer
  # 아래에 허용할 IP 대역(CIDR 형식)을 입력한다.
  loadBalancerSourceRanges:
    - "122.36.213.114/32"              # 특정 IP 1개만 허용할 때
  selector:
    app: sionna-dev
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80                   # VS Code & Jupyter 통합 포트
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sionna-dev-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sionna-dev
  template:
    metadata:
      labels:
        app: sionna-dev
    spec:
      containers:
      - name: sionna-dev-container
        # VS Code가 포함된 최신 NVIDIA TensorFlow 이미지 사용
        image: codercom/code-server:latest 
        ports:
        - containerPort: 80
        env:
        - name: PASSWORD
          value: "code!@#"         # 접속 비밀번호
        resources:
          limits:
            nvidia.com: 1          # GPU 가속 활성화 (필요시)
            memory: "8Gi"
            cpu: "4"
        volumeMounts:
        - name: workspace-storage
          mountPath: /home/coder/project
      # Sionna 설치 및 환경 구성을 위한 초기화 (필요시 커스텀 이미지 권장)
      volumes:
      - name: workspace-storage
        emptyDir: {}
EOF
```
주요 설정 및 사용 안내
* 이미지 선택: 위 예시는 code-server 기본 이미지입니다. Sionna를 바로 쓰려면 NVIDIA NGC TensorFlow 이미지를 베이스로 code-server를 설치한 Custom Dockerfile을 사용하는 것이 가장 안정적입니다.
* GPU 할당: nvidia.com: 1 설정을 통해 EKS 노드의 GPU를 컨테이너가 점유합니다. (EKS에 NVIDIA Device Plugin이 설치되어 있어야 합니다.)
접속 방법:
* kubectl apply -f sionna-dev.yaml 실행 후
* kubectl get svc 명령어로 생성된 EXTERNAL-IP에 접속합니다.
* VS Code 터미널에서 pip install sionna를 실행하여 환경을 완성합니다


    
