# Demo ao vivo — Talos Linux

Roteiro de comandos para executar no terminal durante a demo (slide "🖥️ Demo ao vivo").
Cluster local via provisioner **docker** — todos os comandos abaixo funcionam nesse modo (sem VMs, sem bare metal).

## 1. Subir o cluster

Sobe um cluster Talos completo usando um único container Docker como "nó" — control plane e worker no mesmo node, pra manter a demo rápida e leve. Por baixo dos panos o `talosctl` já faz o bootstrap do etcd, aplica a MachineConfig e espera o cluster ficar saudável, tudo isso sem nenhum passo manual.

Com um nó só, o control plane sobe com o taint padrão que bloqueia agendamento de pods comuns — e o `talosctl` só considera o cluster "saudável" quando o CoreDNS estiver `Running`, o que não acontece enquanto o taint existir. O `--config-patch` abaixo já remove esse taint na criação (`cluster.allowSchedulingOnControlPlanes: true`), então o health-check não fica preso esperando — sem isso, o comando pode parecer travado por vários minutos:

```bash
# Garante que não sobrou estado de uma execução anterior (o destroy sem --name mira o "talos-default")
talosctl cluster destroy --name talos-devops-days-bsb

talosctl cluster create docker --name talos-devops-days-bsb --workers 0 \
  --config-patch '{"cluster":{"allowSchedulingOnControlPlanes":true}}'
```

> Por padrão o `cluster create docker` sobe **1 control plane + 1 worker** (`--workers 1`). O `--workers 0` mantém um nó só, como descrito acima.

Assim que o comando termina, o `talosctl` já deixa o `talosconfig` configurado (contexto do `talosctl` em si), mas sem nó padrão definido — por isso todo comando abaixo passa `-n <ip>` explicitamente. O `kubeconfig` **não** vem automático nessa versão — é um passo explícito, feito puxando ele do próprio nó via API (sem SSH, como tudo aqui):

```bash
talosctl kubeconfig -n 10.5.0.2
kubectl config use-context admin@talos-devops-days-bsb
talosctl config info
kubectl config current-context
```

### Se o CoreDNS ficar `Pending`

Sintoma: `0/N nodes are available: N node(s) had untolerated taint(s)`. Veja qual taint está bloqueando:

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

* `node-role.kubernetes.io/control-plane:NoSchedule` — o patch não foi aplicado (geralmente cluster recriado em cima de estado antigo). Remova na hora:
  ```bash
  kubectl taint nodes --all node-role.kubernetes.io/control-plane-
  ```
* `node.kubernetes.io/not-ready` — o CNI (flannel) ainda não subiu. Aguarde 1–2 min e, se persistir, veja `kubectl -n kube-system logs ds/kube-flannel`.

## 2. Ver os nós sem SSH (API gRPC via talosctl)

Esses três comandos já mostram a mecânica central da palestra: tudo passa pela API gRPC do `machined`, autenticada por certificado — nenhum deles abre uma sessão no nó.

```bash
# Lista os nós do cluster e seu status (via gRPC, não via SSH)
talosctl -n 10.5.0.2 get members

# Versão do Talos rodando nesse nó especificamente
talosctl -n 10.5.0.2 version

# IPs conhecidos pelo nó (equivalente a um "ip addr" remoto, sem shell)
talosctl -n 10.5.0.2 get nodeaddresses
```

## 3. Dashboard e diagnóstico sem SSH

O `dashboard` é o momento mais visual da demo: uma TUI completa — CPU, memória, rede, logs em tempo real — entregue via API, sem agente instalado no nó e sem shell algum.

```bash
talosctl -n 10.5.0.2 dashboard
```

Depois de sair do dashboard (`Ctrl+C`), os mesmos dados de diagnóstico dão pra puxar direto no terminal: `health` roda os checks internos do cluster, `dmesg` traz o log do kernel, e `logs machined` mostra o log do próprio PID 1 — tudo isso é o que, num Linux tradicional, você só conseguiria com um SSH aberto.

```bash
talosctl -n 10.5.0.2 health
talosctl -n 10.5.0.2 dmesg
talosctl -n 10.5.0.2 logs machined
```

## 4. Containers e etcd por dentro do nó

Aqui dá pra mostrar a separação de namespaces que apareceu no diagrama de arquitetura: os containers internos do próprio Talos (`system`) ficam isolados dos containers do Kubernetes (`k8s.io`).

```bash
# Containers do próprio Talos (namespace system) — apid, trustd, udevd etc.
talosctl -n 10.5.0.2 containers

# Containers do Kubernetes (namespace k8s.io) — kubelet, etcd, kube-apiserver...
talosctl -n 10.5.0.2 containers -k
```

E aqui é o gancho direto com o slide do etcd: sem `etcdctl`, sem SSH — a gestão do etcd (membros do cluster, status de saúde) passa inteiramente pelo `talosctl etcd`.

```bash
talosctl -n 10.5.0.2 etcd members
talosctl -n 10.5.0.2 etcd status
```

## 5. Kubernetes funcionando normalmente

O ponto aqui é mostrar que, apesar de tudo que muda por baixo, a experiência de quem opera o Kubernetes em cima é 100% padrão — `kubectl` normal, sem nenhuma particularidade.

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

## 6. Provar a filosofia "sem SSH" na prática

O contraste mais direto da palestra: tentar entrar via SSH falha porque o serviço simplesmente não existe no nó — não é uma porta fechada por firewall, é a ausência real do daemon. Em seguida, mostro que dá pra ler qualquer arquivo do sistema (nesse caso, a versão do OS) sem shell nenhum, só com uma chamada de API.

```bash
# Tentativa de acesso via SSH — vai falhar
ssh root@10.5.0.2
# ssh: connect to host 10.5.0.2 port 22: Connection refused

# A forma correta: API via talosctl
talosctl -n 10.5.0.2 read /etc/os-release
```

## 7. Deploy de uma aplicação real (Super Mario)

Pra fechar a demo com algo tangível, subo uma aplicação de verdade no cluster — um Super Mario jogável no navegador, empacotado na imagem [`pengbai/docker-supermario`](https://hub.docker.com/r/pengbai/docker-supermario) (Tomcat servindo o jogo na porta `8080`). Como já removemos o taint do control plane na criação do cluster (passo 1), o pod agenda normalmente, sem passo extra aqui.

> A imagem tem ~650MB. O nó do Talos tem o próprio containerd, então um `docker pull` no host **não** adianta: rode este passo uma vez no cluster já criado antes da palestra (e apague só o deployment), pra imagem ficar em cache no nó e o pod subir na hora.

```bash
kubectl create deployment supermario --image=pengbai/docker-supermario --port=8080
kubectl expose deployment supermario --port=8080 --target-port=8080

# Espera o rollout terminar e confirma que o pod subiu
kubectl rollout status deployment/supermario
kubectl get pods -l app=supermario
```

O Service é `ClusterIP` — não tem LoadBalancer nem Ingress na frente, de propósito, pra manter a demo simples. Pra acessar do navegador, uso `port-forward`, que abre um túnel direto da minha máquina até o pod dentro do cluster:

```bash
kubectl port-forward svc/supermario 8080:8080
```

Com o port-forward rodando, abro no navegador: **http://localhost:8080** — e jogo uma fase rápida pra mostrar que a aplicação está mesmo de pé.

Se sobrar tempo, mostro os logs do Tomcat direto pela API do cluster:

```bash
kubectl logs deploy/supermario --tail=20
```

## Limpeza (pós-demo)

Remove a aplicação e destrói o cluster local, liberando os containers Docker usados como nós.

```bash
kubectl delete deployment,service supermario

talosctl cluster destroy --name talos-devops-days-bsb
```
