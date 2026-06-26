# Automatyzacja K3s przy użyciu Ansible Roles

## Cel

Celem etapu było pełne zautomatyzowanie instalacji klastra K3s przy użyciu Ansible.

Automatyzacja obejmuje:

- instalację K3s Server (Control Plane),
- automatyczne pobieranie tokena klastra,
- instalację K3s Agent na workerach,
- automatyczne dołączanie workerów do klastra,
- konfigurację poprawnych adresów IP w sieci Host-Only,
- weryfikację działania klastra po wdrożeniu.

Dzięki temu możliwe jest odtworzenie klastra od zera za pomocą kilku poleceń Ansible.

> [!INFO] Kontekst
>
> Ten etap kontynuuje prace z [[instalacja-k3s-control-plane|Instalacji K3s Control Plane]], gdzie ręczna instalacja serwera pozwoliła zidentyfikować problem z wyborem adresu NAT (`10.0.2.15`) i wypracować rozwiązanie polegające na wymuszeniu adresu Host-Only. Modlitwa ta zostaje tu przeniesiona do ról Ansible.

---

## Struktura Ansible Roles

W projekcie utworzono dwie role:

```text
roles/
├── k3s_server
└── k3s_agent
```

Role zostały wygenerowane przy pomocy:

```bash
ansible-galaxy role init k3s_server
ansible-galaxy role init k3s_agent
```

Każda rola posiada standardową strukturę katalogów Ansible:

```text
tasks/
defaults/
handlers/
vars/
templates/
files/
meta/
tests/
```

---

## Konfiguracja Inventory

Plik:

```text
/opt/projekt_devops/ansible/inventory.ini
```

został rozszerzony o adresy IP używane przez K3s:

```ini
[k3s_server]
192.168.56.102 k3s_node_ip=192.168.56.102

[k3s_agents]
192.168.56.103 k3s_node_ip=192.168.56.103
192.168.56.104 k3s_node_ip=192.168.56.104
```

Dzięki temu każda maszyna posiada własny poprawny adres używany później przez K3s w roli jako `{{ k3s_node_ip }}`.

---

## Rola `k3s_server`

Playbook:

```text
playbooks/install-k3s-server.yml
```

wykorzystuje rolę:

```text
roles/k3s_server
```

Rola realizuje następujące zadania:

### 1. Pobranie instalatora K3s

```yaml
get_url:
  url: https://get.k3s.io
  dest: /tmp/k3s-install.sh
  mode: "0755"
```

### 2. Instalacja K3s Server z wymuszonym adresem Host-Only

Aby zapobiec wyborowi domyślnego adresu NAT `10.0.2.15`, instalacja uruchamiana jest z jawnie wskazanymi adresami:

```yaml
INSTALL_K3S_EXEC: >-
  server
  --node-ip={{ k3s_node_ip }}
  --advertise-address={{ k3s_node_ip }}
```

Dzięki temu serwer udostępnia API pod poprawnym adresem klastra:

```text
192.168.56.102
```

a nie błędnym:

```text
10.0.2.15
```

Sformalizowane w [[ADR-001 — Wymuszenie adresu IP Host-Only dla K3s|ADR-001]].

### 3. Idempotentność

Ponowne uruchomienie playbooka nie powoduje ponownej instalacji K3s. Wykorzystano mechanizm:

```yaml
creates: /usr/local/bin/k3s
```

Dzięki temu instalator uruchamiany jest tylko wtedy, gdy K3s nie jest jeszcze zainstalowany.

---

## Rola `k3s_agent`

Playbook:

```text
playbooks/install-k3s-agent.yml
```

wykorzystuje rolę:

```text
roles/k3s_agent
```

Rola realizuje następujące zadania:

### 1. Pobranie instalatora

```yaml
get_url:
  url: https://get.k3s.io
  dest: /tmp/k3s-install.sh
  mode: "0755"
```

### 2. Pobranie tokena z Control Plane

Token odczytywany jest automatycznie z hosta Control Plane przy użyciu modułu `slurp`:

```yaml
slurp:
  src: /var/lib/rancher/k3s/server/node-token
  delegate_to: "{{ groups['k3s_server'][0] }}"
  register: k3s_token_raw
```

### 3. Dekodowanie tokena

Zmienna pomocnicza dekoduje zawartość (slurp zwraca base64) i czyści białe znaki:

```yaml
set_fact:
  k3s_token: "{{ k3s_token_raw.content | b64decode | trim }}"
```

### 4. Instalacja agenta

```yaml
K3S_URL: "https://{{ k3s_server_ip }}:6443"
K3S_TOKEN: "{{ k3s_token }}"

INSTALL_K3S_EXEC: >-
  agent
  --node-ip={{ k3s_node_ip }}
```

Dzięki temu workery automatycznie dołączają do klastra, łącząc się z Control Plane pod poprawnym adresem `192.168.56.102:6443` i zgłaszając swój własny adres Host-Only.

> [!NOTE] Automatyzacja tokena
>
> Podejście eliminuje konieczność ręcznego kopiowania tokena między serwerami, co znacząco upraszcza proces dołączania nowych workerów. Decyzja sformalizowana w [[ADR-002 — Automatyczne pobieranie tokena klastra K3s przez Ansible|ADR-002]].

---

## Uruchomienie

### Weryfikacja komunikacji Ansible

```bash
ansible all -m ping -K
```

Wynik:

```text
192.168.56.102 | SUCCESS
192.168.56.103 | SUCCESS
192.168.56.104 | SUCCESS
```

Potwierdziło poprawne działanie:

- SSH,
- privilege escalation (sudo),
- komunikacji Ansible.

### Instalacja Control Plane

```bash
ansible-playbook -i inventory.ini \
  playbooks/install-k3s-server.yml -K
```

### Instalacja workerów

```bash
ansible-playbook -i inventory.ini \
  playbooks/install-k3s-agent.yml -K
```

---

## Weryfikacja Klastra

Sprawdzenie stanu nodów:

```bash
sudo k3s kubectl get nodes -o wide
```

Wynik:

```text
NAME             STATUS   ROLES           INTERNAL-IP
k8s-control-01   Ready    control-plane   192.168.56.102
k8s-node-01      Ready                    192.168.56.103
k8s-node-02      Ready                    192.168.56.104
```

Wszystkie nody uzyskały status `Ready` z poprawnymi adresami `INTERNAL-IP` (Host-Only), co potwierdza poprawne działanie klastra oraz prawidłowe wymuszenie adresacji IP.

---

## Test Odtworzenia Środowiska

Przeprowadzono pełny test odtworzenia klastra od zera, weryfikujący poprawność automatyzacji.

### Kroki

1. Przywrócenie snapshotów VirtualBox (`k3s-control-plane-ready`).
2. Ponowne sklonowanie repozytorium Git z GitHub.
3. Brak ręcznych zmian w kodzie.
4. Uruchomienie playbooka instalacji agentów.
5. Weryfikacja stanu klastra.

### Wynik

```bash
sudo k3s kubectl get nodes -o wide
```

```text
k8s-control-01   Ready
k8s-node-01      Ready
k8s-node-02      Ready
```

Wszystkie węzły poprawnie dołączyły do klastra wyłącznie na podstawie kodu z repozytorium, co potwierdza poprawność i kompletność przygotowanej automatyzacji.

---

## Migawka (Snapshot)

Po zakończeniu etapu utworzono snapshot na wszystkich maszynach:

```
k3s-ansible-cluster-ready
```

### Stan środowiska

| Host           | Rola          | Status |
| -------------- | ------------- | ------ |
| k8s-control-01 | Control Plane | Ready  |
| k8s-node-01    | Worker        | Ready  |
| k8s-node-02    | Worker        | Ready  |

Migawka stanowi punkt przywracania dla kolejnych etapów (wdrażanie aplikacji, monitoring). Sformalizowana w [[ADR-003 — Strategia snapshotów VirtualBox jako punkty kontrolne|ADR-003]].

---

## Stan Projektu po Zakończeniu Etapu

Ukończono:

- przygotowanie hostów ([[przygotowanie-hostow-k3s|Przygotowanie Hostów pod K3s]]),
- instalację K3s Control Plane ([[instalacja-k3s-control-plane|Instalacja K3s Control Plane]]),
- automatyczne dołączanie workerów,
- automatyzację przy użyciu Ansible Roles,
- test odtwarzania środowiska.

Środowisko jest gotowe do kolejnego etapu — wdrażania aplikacji na klaster Kubernetes.

---

## Następny Etap

Wdrażanie podstawowych obiektów Kubernetes (Namespace, Deployment, Service, Ingress) weryfikujących działanie klastra z poziomu aplikacji opisane jest w [[podstawowe-obiekty-kubernetes|Podstawowych Obiektach Kubernetes]].