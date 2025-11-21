# Laboratório: Argo Rollouts Blue-Green Deployment
  
**Objetivo**: Entender e praticar como o Argo Rollouts implementa a estratégia Blue-Green

---

## Conceitos: Argo Rollouts Blue-Green

### **O que diferencia Argo Rollouts de Deployments comuns?**

```
Kubernetes Deployment
└─ Um ReplicaSet
   └─ Replicas de pods
   └─ Todos os pods da mesma versão
   └─ Controle de deployments tradicional e construção lógica manualmente.

Argo Rollouts (Blue-Green)
├─ ReplicaSet A (Blue) - versão atual/estável
├─ ReplicaSet B (Green) - nova versão
└─ Controle Automatico: não há necessidade de construir lógica manualmente.
```

### **Responsaveis:**

```
1. Rollout (CRD Custom Resource)
   ├─ Define estratégia: blueGreen
   ├─ Gerencia ReplicaSets
   └─ Coordena a transição

2. Service Ativo (web-color-active)
   ├─ Recebe tráfego de produção
   ├─ Apontado para versão estável
   └─ Argo alterna este selector automaticamente

3. Service Preview (web-color-preview)
   ├─ Recebe tráfego de teste
   ├─ Apontado para nova versão
   └─ Você pode testar antes de promover
```

Validando o CRD do Argo Rollouts:

```bash
kubectl get crd rollouts.argoproj.io
kubectl gdescribe crd rollouts.argoproj.io
```

### **Fluxo Blue-Green com Argo Rollouts**

```
PASSO 1: ESTADO INICIAL (Blue ativo)
├─ Blue ReplicaSet: 3 pods rodando
├─ Green ReplicaSet: não existe
├─ web-color-active → aponta para Blue
└─ web-color-preview → aponta para Blue

PASSO 2: NOVO DEPLOY (criar Green)
├─ Novo ReplicaSet criado (Green)
├─ Green pods: 3 pods sendo criados
├─ Blue pods: ainda rodando
├─ web-color-active → AINDA aponta para Blue (sem interrupção!)
└─ web-color-preview → alterado para apontar para Green

PASSO 3: TESTES (testar Green)
├─ Blue pods: 3 rodando (tráfego de produção)
├─ Green pods: 3 rodando (isolado para testes)
├─ web-color-active → Blue (produção não afetada)
└─ web-color-preview → Green (você testa aqui)

PASSO 4: PROMOÇÃO (trocar para Green)
├─ Você executa: kubectl argo rollouts promote
├─ Argo executa:
│  ├─ web-color-active → muda para apontar para Green
│  └─ Blue ReplicaSet → marcado para deletar
└─ Resultado: Green agora é o novo "Blue"

PASSO 5: SCALE DOWN (limpar versão antiga)
├─ Blue ReplicaSet: deletado após 2 segundos (scaleDownDelaySeconds)
├─ Green ReplicaSet: mantém 3 replicas
├─ web-color-active → Green (produção)
└─ Pronto para próximo deployment!
```

---

## Lab 1: Setup e Conceitos (não obrigatório)

### **1.1 Verificar pré-requisitos**

```bash
# Verificar se Argo Rollouts está instalado
kubectl get crd rollouts.argoproj.io

# Esperado:
# NAME                             CREATED AT
# rollouts.argoproj.io             2025-10-30T...

# Verificar namespace do Argo Rollouts
kubectl get ns | grep argo

# Se não existir, instalar:
# kubectl create namespace argo-rollouts
# kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/download/v1.8.3/install.yaml
```

### **1.2 Migrando de deployment para o Rollout**

Abra o arquivo `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-color-rollout
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-color
  template:
    metadata:
      labels:
        app: web-color
        version: green
    spec:
      containers:
      - name: app-container-green
        # Imagem com a versão blue - green
        image: bernardolankheet/lab-blue-green:green
        ports:
        - containerPort: 80
```

Como ficaria o Rollout equivalente?
```yaml
apiVersion: argoproj.io/v1alpha1     # ← Objeto Custom do Argo
kind: Rollout                        # ← Objeto Custom do Argo
metadata:
  name: web-color-rollout
spec:
  replicas: 3                        # ← 3 pods (como Deployment), utilizando a logica que esta terá mais replicas.
  selector:
    matchLabels:
      app: web-color                 # ← Seleciona pods por label
  template:
    spec:
      containers:
      - name: app-container-green
        image: bernardolankheet/lab-blue-green:green  # ← Versão inicial em produção
        ports:
        - containerPort: 80
  
  strategy:
    blueGreen:                        # ← Estratégia Blue-Green
      activeService: web-color-active    # ← Service de produção
      previewService: web-color-preview  # ← Service de teste
      autoPromotionEnabled: false        # ← Manual (você escolhe promover)
      scaleDownDelaySeconds: 2           # ← Aguarda 2s antes deletar versão antiga
```

### **1.3 Entender os Services**

**web-color-active (produção)**:
```yaml
metadata:
  name: web-color-active             # Definimos apenas um nome para o serviço para caada.
```

Ambos começam iguais, mas o Argo vai:
- **active** → serviço referente a versão estável (em terioria a que está em produção);
- **preview** → serviço que irá ser a versão nova

---

## Lab 2: Aplicar Manifesto e Observar Inicial

### **2.1 Aplicar recursos**

```bash
# Ir para a pasta
cd rollouts-bluegreen

# Aplicar tudo (Rollout + Services + Ingress)
kubectl apply -k .

# Verificar aplicação
kubectl get rollout,svc,ingress
```

### **2.2 Monitorar o estado inicial**

**Terminal 1 - Rollout status**:
```bash
kubectl get rollout -w

# Esperado:
# NAME                DESIRED   CURRENT   UP-TO-DATE   AVAILABLE
# web-color-rollout   3         3         3            3
```

**Terminal 2 - ReplicaSets**:
```bash
kubectl get replicaset -o wide

# Esperado:
# NAME                              DESIRED   CURRENT   READY   AGE
# web-color-rollout-xxxxx           3         3         3       2m
# (Apenas um ReplicaSet no início)
```

**Terminal 3 - Pods**:
```bash
kubectl get pods -L version

# Esperado:
# NAME                              READY   STATUS    VERSION
# web-color-rollout-xxxxx-aaaaa     1/1     Running   blue
# web-color-rollout-xxxxx-bbbbb     1/1     Running   blue
# web-color-rollout-xxxxx-ccccc     1/1     Running   blue
```
**Terminal 4 - Endpoints dos Services**:
```bash 
kubectl get endpoints
```

### **2.3 Testar acesso**

```bash
# Port-forward para o service ativo
kubectl port-forward svc/web-color-active 8080:80

# Em outro terminal
curl http://localhost:8080

# Esperado: Página mostrando "Blue Version" ou "Green Version"
# (Depende da imagem configurada no Rollout)
```

---

## Lab 3: Entender Blue-Green do Argo

### **3.1 Descrever o Rollout**

```bash
kubectl describe rollout web-color-rollout

# Procure por:
# Status:
#   BlueGreen:
#     Active Selector: xxxxx (hash do ReplicaSet Blue)
#     Preview Selector: <empty> (ainda não existe)
```

### **3.2 Descrever o Status Completo**

```bash
kubectl get rollout web-color-rollout -o yaml

```

---

## Lab 4: Fazer Update e Acompanhar a Transição

### **4.1 Preparar para mudança**

Deixe os 4 terminais monitorando (Lab 2.2):
- Terminal 1: `kubectl get rollout -w`
- Terminal 2: `kubectl get replicaset -o wide`
- Terminal 3: `kubectl get pods -L version`
- Terminal 4: Watch dos endpoints

### **4.2 Fazer o Update (Mudar Imagem)**

```bash
# Mudar para nova versão
kubectl argo rollouts set image web-color-rollout \
  app-container-blue=bernardolankheet/lab-blue-green:green

# Ou editar direto:
kubectl edit rollout web-color-rollout
# Mudar imagem de "green" para "blue"
```

### **4.3 Observar o que acontece**

**Timeline esperada**:

```
T+0s: VOCÊ EXECUTA O UPDATE
     └─ Rollout recebe nova imagem

T+1s: NOVO REPLICASET CRIADO
     ├─ Terminal 2: Novo ReplicaSet aparece com "0" pods
     ├─ Terminal 3: Pods antigos (Blue) ainda rodando (3)
     └─ Descrição: "Creating ReplicaSet"

T+2s: PODS VERDES SENDO CRIADOS
     ├─ Terminal 3: Vê "0/3" Running da nova versão
     ├─ Imagem está sendo puxada
     └─ Terminal 2: DESIRED=3, CURRENT=1, READY=0

T+5s: PODS VERDES INICIALIZANDO
     ├─ Terminal 3: Vê pods Green em "ContainerCreating"
     ├─ Terminal 1: Mostra "Active: Blue (3), Preview: Creating (1-2/3)"
     └─ Descrição: "Waiting for ReplicaSet to fully rollout"

T+10s: TODOS OS PODS VERDES PRONTOS
     ├─ Terminal 3: 3 pods Blue + 3 pods Green (6 no total!)
     ├─ Terminal 4: Endpoints ATIVA = Blue (3), PREVIEW = Green (3)
     ├─ Terminal 1: Mostra Blue ativo, Green em preview
     └─ Descrição: "Waiting for user to promote"
```

### **4.4 Verificações**

```bash
# Ver ReplicaSets (agora tem 2!)
kubectl get replicaset -o wide

# Esperado:
# NAME                              DESIRED   CURRENT   READY
# web-color-rollout-blue-xxxxx      3         3         3      ← Blue (antigo)
# web-color-rollout-green-yyyyy     3         3         3      ← Green (novo)

# Ver Rollout está em pausa aguardando promoção
kubectl get rollout -o yaml | grep -A 5 "phase:"

# Esperado:
# Phase: Progressing (ou Paused)
```

### **4.5 Testar a versão Preview**

```bash
# Port-forward para preview (nova versão)
kubectl port-forward svc/web-color-preview 8081:80

# Em outro terminal
for i in {1..5}; do echo "Request $i:"; curl -s http://localhost:8081 | grep -o "Blue\|Green"; done

# Ou
for i in {1..10}; do curl -s https://app.bernardolankheet.com.br/version | grep -o "blue\|green"; done

# Esperado: Todas requisições mostram nova versão (Green)
for i in {1..19}; do curl -s https://app.bernardolankheet.com.br/version | grep -o "blue\|green"; done
blue
blue
green
blue
blue
blue
green
blue
blue
blue
green
blue
blue
blue
green
blue
blue
blue
green
```

---

## Lab 5: Promotion e Rollback

### **5.1 Promover Green para Produção**

```bash
# Promover: Green vira o novo "Blue"
kubectl argo rollouts promote web-color-rollout

# Ou sem plugin:
kubectl patch rollout web-color-rollout --type merge \
  -p '{"status":{"promotionRef":{""}}}' 
```

### **5.2 Observar a Transição**

**Timeline esperada**:

```
T+0s: VOCÊ EXECUTA PROMOTE
     └─ Argo inicia transição

T+1s: SERVICE ACTIVE MUDA
     ├─ Terminal 4: Endpoints ATIVA agora aponta para Green!
     ├─ Terminal 1: Mostra "Active: Green, Preview: Empty"
     └─ Blue ainda rodando (vai deletar em breve)

T+2s: BLUE COMEÇANDO A DELETAR
     ├─ Terminal 3: Pods Blue começam em "Terminating"
     ├─ Terminal 1: ReplicaSet Blue mostra "0" replicas
     └─ ReplicaSet Green agora é o ativo

T+3s: BLUE DELETADO
     ├─ Terminal 2: ReplicaSet Blue desapareceu
     ├─ Terminal 3: Apenas 3 pods Green restantes
     └─ Rollout em estado estável
```

### **5.3 Verificar novo estado**

```bash
# Ver Rollout completo
kubectl describe rollout web-color-rollout

# Procure:
# Status:
#   BlueGreen:
#     Active Selector: yyyyy (agora é Green!)
#     Preview Selector: <empty>

# Ver ReplicaSets (apenas um agora)
kubectl get replicaset

# Esperado: Apenas o novo ReplicaSet (Green)

# Ver Pods
kubectl get pods -L version

# Esperado: 3 pods com version=blue (mas rodando Green!)
```

### **5.4 Testar novo estado**

```bash
# Testar web-color-active (agora é Green)
for i in {1..10}; do echo "Request $i:"; curl -s http://localhost:8080 | grep -o "Blue\|Green"; done

# ou 
for i in {1..10}; do curl -s https://app.bernardolankheet.com.br/version | grep -o "blue\|green"; done

# Esperado: Todas requisições agora mostram Green
```

### **5.5 Fazer Rollback (Reverter)**

```bash
# Reverter para versão anterior
kubectl rollout history deployment web-color-rollout

# Ver revisões
kubectl describe rollout web-color-rollout | grep -A 5 "Conditions"

# Fazer rollback
kubectl argo rollouts undo web-color-rollout

# Observar o mesmo fluxo que na promoção inicial:
# Green vira preview, Blue é recriado e testado, depois promovido
```

---

## Lab 6: Análise Completa

### **6.1 Comparação: Deployment vs Argo Rollouts**

| Aspecto | Deployment | Argo Blue-Green |
|---------|-----------|-----------------|
| **Estratégia** | RollingUpdate/Recreate | Blue-Green manual |
| **Downtime** | Pode ter (Recreate) | Zero downtime |
| **Teste** | Não existe | Service Preview |
| **Rollback** | Histórico de revisões | Instant switch |
| **Controle** | Automático | Manual (promote) |
| **Replicas** | 1x replicas | 2x replicas simultaneamente |

### **6.2 Estados do Argo Blue-Green**

```
1. INITIAL (Inicial)
   ├─ Blue ativo
   └─ Green não existe

2. PROGRESSING (Durante update)
   ├─ Blue ainda ativo (tráfego)
   ├─ Green sendo criado (testes)
   └─ Aguardando promoção

3. PAUSED (Aguardando ação)
   ├─ Ambas versões prontas
   └─ Aguardando "kubectl argo rollouts promote"

4. PROMOTING (Promovendo)
   ├─ Green virou Blue
   ├─ Blue em processo de deleção
   └─ Transição em progresso

5. HEALTHY (Saudável)
   ├─ Green agora é o ativo
   ├─ Blue deletado
   └─ Pronto para próximo update
```

### **6.3 Comandos**

```bash
# Ver status
kubectl get rollout
kubectl describe rollout web-color-rollout

# Ver histórico
kubectl argo rollouts history web-color-rollout

# Fazer update
kubectl set image rollout web-color-rollout app-container-blue=<nova-imagem>

# Promover
kubectl argo rollouts promote web-color-rollout

# Reverter
kubectl argo rollouts undo web-color-rollout

# Pausar/Retomar
kubectl argo rollouts pause web-color-rollout
kubectl argo rollouts resume web-color-rollout

# Monitorar
kubectl argo rollouts get rollout web-color-rollout -w
```

---

## Tente:

### Abortando um rollout: **abort** 

```bash
# Durante um update, antes de promover:
kubectl argo rollouts abort web-color-rollout

# Resultado: Green é deletado, Blue continua ativo
```

### Parametro **postPromotionAnalysis**

O Rollout suporta análise automática (não configurado neste lab):

```yaml
strategy:
  blueGreen:
    activeService: web-color-active
    previewService: web-color-preview
    autoPromotionEnabled: false
    postPromotionAnalysis:        # ← Análise após promoção
      templates:
      - name: my-analysis
```

### Auto-Promotion**

Mudar para `autoPromotionEnabled: true` vai promover automaticamente após testes.

