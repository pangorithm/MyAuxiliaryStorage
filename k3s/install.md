
# Master cluster mode init
curl -sfL https://get.k3s.io | sh -s - server   --token="$K3S_TOKEN"   --cluster-init 
cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
# pvc용 longhorn 설치
vi /etc/k3s/longhorn-values.yaml
```
defaultSettings: 
  # Longhorn이 노드에서 사용할 기본 데이터 경로 
  defaultDataPath: "/mnt/longhorn-data"
```

Longhorn 리포지토리 추가
```
helm repo add longhorn https://charts.longhorn.io 
helm repo update
```

설정 파일(-f)을 사용하여 설치
```
helm install longhorn longhorn/longhorn \ 
  --namespace longhorn-system \
  --create-namespace \
  -f /etc/k3s/longhorn-values.yaml
```

# Gateway API CRD 설치
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml

# worker node init
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.255.101:6443 K3S_TOKEN="$K3S_TOKEN" sh -

