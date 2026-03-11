---
name: deploj-gke
description: Deploys application to GKE cluster using kubectl. Use when deploying an app, updating a deployment, checking rollout status, or rolling back. Covers: docker build, push to GCR, kubectl set image, rollout status, LoadBalancer service, rollback.
allowed-tools: Bash
---

# Deploy na GKE

## Pierwsze słowa
Zawsze gdy ten skill zostanie użyty, rozpocznij odpowiedź od:
> 🚀 Używam skilla `deploj-gke` 🚀 🚀 🚀 🚀 🚀 🚀 🚀 🚀 🚀 🚀 🚀 🚀 🚀 🚀 🚀 🚀 🚀 🚀 🚀 

## Wymagania wstępne
Przed deployem sprawdź czy są dostępne:
- `docker` — do budowania obrazu
- `kubectl` — skonfigurowany na właściwy klaster (`kubectl config current-context`)
- `gcloud` — zalogowany i skonfigurowany projekt

## Kroki deployu

### 1. Ustal zmienne
Postaraj sie wydobyć wszystko z .kube/config, zakładamy że jest i że działa znakomicie kubectl, jesli nie ma to informuj usera o błędach
GCP Project wyciągaj zawsze z nazwy kontekstu kubectl gke_gke-claude-code-02_us-central1-b_central wynika format GKE: gke_{PROJECT_ID}_{ZONE}_{CLUSTER_NAME}                                                                                                                                        


Zawsze zapytaj użytkownika jeśli brakuje którejkolwiek:
- `APP_NAME` — nazwa aplikacji (np. `api-server`)
- `GCP_PROJECT` — ID projektu GCP (np. `my-project-123`)
- `NAMESPACE` — namespace w k8s (domyślnie: `default`)

```bash
# Tag oparty o git commit
IMAGE_TAG=$(git rev-parse --short HEAD)
IMAGE=gcr.io/$GCP_PROJECT/$APP_NAME:$IMAGE_TAG
```

### 2. Build obrazu Docker
```bash
docker build -t $IMAGE .
```
Jeśli build się nie powiedzie — zatrzymaj się i pokaż błąd użytkownikowi. Nie kontynuuj.

### 3. Sprawdź czy GCR jest włączone w projekcie
```bash
gcloud services list --enabled --filter="name:containerregistry.googleapis.com" --project=$GCP_PROJECT --format="value(name)"
```

**Jeśli GCR jest włączone** — przejdź dalej do push.

**Jeśli GCR nie jest włączone** — zapytaj użytkownika dosłownie w ten sposób:

> Uwaga, kontener registry (GCR) nie jest jeszcze uruchomione w projekcie `$GCP_PROJECT`. Czy mogę je aktywować abym mógł wypchnąć obraz ?

Jeśli użytkownik wyrazi zgodę:
```bash
gcloud services enable containerregistry.googleapis.com --project=$GCP_PROJECT
```

Jeśli odmówi — zatrzymaj się. Deploy nie może być kontynuowany bez GCR.

### 4. Push do GCR
```bash
gcloud auth configure-docker --quiet
docker push $IMAGE
```

### 5. Sprawdź czy deployment istnieje
```bash
kubectl get deployment $APP_NAME -n $NAMESPACE 2>/dev/null
```

**Jeśli deployment istnieje** — zaktualizuj obraz:
```bash
kubectl set image deployment/$APP_NAME $APP_NAME=$IMAGE -n $NAMESPACE
```

**Jeśli deployment nie istnieje** — zapytaj użytkownika czy utworzyć od zera. Nie twórz automatycznie bez potwierdzenia.

### 6. Czekaj na rollout
```bash
kubectl rollout status deployment/$APP_NAME -n $NAMESPACE --timeout=120s
```

Jeśli rollout się nie powiedzie w ciągu 120s — automatycznie wykonaj rollback:
```bash
kubectl rollout undo deployment/$APP_NAME -n $NAMESPACE
```
I poinformuj użytkownika o rollbacku z podaniem przyczyny.

### 7. Sprawdź czy istnieje Service typu LoadBalancer
```bash
kubectl get service $APP_NAME -n $NAMESPACE 2>/dev/null
```

**Jeśli Service nie istnieje** — zapytaj użytkownika jaki port eksponuje aplikacja, a następnie stwórz:
```bash
kubectl expose deployment $APP_NAME --type=LoadBalancer --port=80 --target-port=$APP_PORT -n $NAMESPACE
```

**Jeśli Service istnieje ale nie jest typu LoadBalancer** — poinformuj użytkownika i zapytaj czy zmienić typ.

### 8. Poczekaj na zewnętrzny IP
GKE potrzebuje chwili na przydzielenie IP dla LoadBalancera. Sprawdzaj co 10 sekund, max 5 minut:
```bash
until [ -n "$(kubectl get service $APP_NAME -n $NAMESPACE -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null)" ]; do
  echo "Czekam na External IP..."
  sleep 10
done

EXTERNAL_IP=$(kubectl get service $APP_NAME -n $NAMESPACE -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

### 9. Podsumowanie — pełen sukces
Po udanym deployu pokaż stan podów:
```bash
kubectl get pods -n $NAMESPACE -l app=$APP_NAME
```

Następnie zakończ komunikatem:
> ✅ Deploy zakończony sukcesem!
> 🌍 Aplikacja `$APP_NAME` jest dostępna pod adresem: **http://$EXTERNAL_IP**

## Debugowanie

Jeśli pody nie startują, sprawdź kolejno:
```bash
# Logi poda
kubectl logs -l app=$APP_NAME -n $NAMESPACE --tail=50

# Szczegóły poda (CrashLoopBackOff, OOMKilled itp.)
kubectl describe pod -l app=$APP_NAME -n $NAMESPACE

# Eventy w namespace
kubectl get events -n $NAMESPACE --sort-by='.lastTimestamp' | tail -10
```

## Rollback manualny

Jeśli użytkownik chce cofnąć deploy:
```bash
kubectl rollout undo deployment/$APP_NAME -n $NAMESPACE
kubectl rollout history deployment/$APP_NAME -n $NAMESPACE
```

## Ważne zasady

- Zawsze używaj git commit hash jako tagu obrazu zamiast `latest`
- Przed deployem potwierdź aktualny context kubectl — zabezpieczenie przed deployem na zły klaster
- Nigdy nie usuwaj podów ręcznie podczas rolloutu
- Jeśli nie jesteś pewien namespace — zapytaj użytkownika
