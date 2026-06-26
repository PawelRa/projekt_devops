# Instalacja K3s Control Plane

## Cel

Celem etapu było uruchomienie pierwszego węzła klastra K3s pełniącego rolę Control Plane oraz weryfikacja poprawności działania podstawowych komponentów Kubernetes.

> [!INFO] Kontekst
>
> Ten etap kontynuuje prace z [[przygotowanie-hostow-k3s|Przygotowania Hostów pod Instalację K3s]], gdzie wyłączono swap i zainstalowano narzędzia administracyjne na wszystkich trzech serwerach.

---

## Środowisko

| Hostname       | Rola          | Adres IP       |
| -------------- | ------------- | -------------- |
| k8s-control-01 | Control Plane | 192.168.56.102 |
| k8s-node-01    | Worker        | 192.168.56.103 |
| k8s-node-02    | Worker        | 192.168.56.104 |

- System operacyjny: Ubuntu Server 24.04 LTS
- Platforma wirtualizacyjna: Oracle VirtualBox
- Sieć prywatna: 192.168.56.0/24

Topologia hosta (dwa interfejsy):

| Interfejs | Adres IP     | Przeznaczenie                          |
| --------- | ------------ | -------------------------------------- |
| `enp0s3`  | `10.0.2.15`  | NAT — dostęp do Internetu              |
| `enp0s8`  | `192.168.56.x` | Host-Only — komunikacja wewnątrz klastra |

---

## Problem z domyślną instalacją K3s

### Objaw

Po wykonaniu domyślnej instalacji K3s serwer reklamował adres:

```text
10.0.2.15
```

czyli adres interfejsu NAT VirtualBox, zamiast adresu sieci prywatnej Host-Only.

W efekcie workery próbowały komunikować się z Control Plane pod nieprawidłowym adresem:

```text
dial tcp 10.0.2.15:6443: connect: connection refused
```

Mimo że instalacja agentów kończyła się pozornym sukcesem, węzły nie mogły poprawnie dołączyć do klastra — ich ruch trafiał na adapter NAT, na którym port 6443 nie był wystawiony.

> [!WARNING] Źródło problemu
> K3s podczas standardowej instalacji automatycznie wybiera adres interpunkcji sieciowej — w tym wypadku adres `10.0.2.15` interfejsu `enp0s3`. W środowisku z wieloma kartami sieciowymi (NAT + Host-Only) konieczne jest jawne wskazanie adresu używanego przez klaster.

---

## Rozwiązanie — wymuszenie adresu Host-Only

Podczas instalacji serwera jawnie wskazano adres sieci prywatnej, używając zmiennych instalatora K3s:

```bash
curl -sfL https://get.k3s.io | \
  INSTALL_K3S_EXEC="server \
  --node-ip=192.168.56.102 \
  --advertise-address=192.168.56.102" \
  sh -
```

Flagi instalatora:

| Flaga                          | Przeznaczenie                                                       |
| ------------------------------ | -------------------------------------------------------------------- |
| `--node-ip=192.168.56.102`     | Adres IP węzła zgłaszany do API servera                              |
| `--advertise-address=192.168.56.102` | Adres, pod którym API server udostępnia usługę (port 6443)    |

Po tej zmianie K3s poprawnie udostępnia API Kubernetes pod adresem:

```text
192.168.56.102:6443
```

Decyzja ta została sformalizowana w [[ADR-001 — Wymuszenie adresu IP Host-Only dla K3s|ADR-001]].

---

## Weryfikacja Control Plane

### Stan węzła

```bash
sudo k3s kubectl get nodes -o wide
```

Wynik:

```text
NAME             STATUS   ROLES           VERSION
k8s-control-01   Ready    control-plane   v1.35.5+k3s1
```

### Komponenty systemowe

```bash
sudo k3s kubectl get pods -A -o wide
```

Uruchomione komponenty:

| Namespace | Komponent                      | Status |
| --------- | ------------------------------ | ------ |
| kube-system | CoreDNS                      | Running |
| kube-system | Metrics Server               | Running |
| kube-system | Local Path Provisioner       | Running |
| kube-system | Traefik Ingress Controller   | Running |
| kube-system | Service LoadBalancer (svclb-traefik) | Running |

Wszystkie pody posiadały status `Running` lub `Completed`, co potwierdza poprawne uruchomienie klastra.

---

## Token dołączania workerów

Token został pobrany z serwera:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

Przykład (wartość zanonimizowana):

```text
K10XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX::server:YYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYY
```

> [!WARNING] Bezpieczeństwo tokena
>
> Token jest wymagany do dołączania kolejnych workerów do klastra. W środowisku laboratoryjnym trzymano go w historii powłoki; w docelowym rozwiązaniu został zautomatyzowany pobieraniem przez Ansible (zobacz [[automatyzacja-k3s-ansible|Automatyzację K3s]] i [[ADR-002 — Automatyczne pobieranie tokena klastra K3s przez Ansible|ADR-002]]).

---

## Migawka (Snapshot)

Po potwierdzeniu poprawnego działania Control Plane utworzono migawkę na wszystkich maszynach:

```
k3s-control-plane-ready
```

Stan migawki:

- K3s Server działa poprawnie
- Kubernetes API dostępne pod `192.168.56.102:6443`
- Control Plane w stanie `Ready`
- Workery nie są jeszcze dołączone

Migawka stanowi punkt startowy do przygotowania pełnej automatyzacji instalacji agentów K3s przy użyciu Ansible.

---

## Stan Środowiska po Zakończeniu Etapu

```
k8s-control-01 (Control Plane)
├── K3s Server v1.35.5+k3s1
├── Node status: Ready
├── CoreDNS, Metrics Server, Traefik, svclb — Running
└── K3s API: 192.168.56.102:6443

k8s-node-01, k8s-node-02 (Workery)
└── nie dołączone do klastra (oczekują na instalację agenta)
```

---

## Następny Etap

Control Plane działa poprawnie. Kolejny etap — pełna automatyzacja instalacji K3s przy użyciu Ansible Roles, w tym automatyczne pobieranie tokena i dołączanie workerów opisany jest w [[automatyzacja-k3s-ansible|Automatyzacji K3s przy użyciu Ansible Roles]].