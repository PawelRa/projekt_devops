# Lekcje Wyciągnięte (Lessons Learned)

## Cel

Niniejsza notatka agreguje najważniejsze doświadczenia zdobyte podczas konfiguracji środowiska laboratoryjnego i instalacji klastra K3s. Służy dwóm celom:

1. udokumentowaniu reasoningu kluczowych decyzji (odsyła do formalnych [[ADR-001 — Wymuszenie adresu IP Host-Only dla K3s|ADR-ów]]),
2. skróceniu dalszego cyklu iteracyjnego poprzez udostępnienie gotowej listy problemów i obejść (zobacz też [[Troubleshooting]]).

> [!INFO] Zakres
>
> Lekcje dotyczą środowiska zbudowanego w etapach:
> - [[przygotowanie-srodowiska|Przygotowanie Środowiska Laboratoryjnego]]
> - [[przygotowanie-wezlow|Przygotowanie Węzłów Klastra Kubernetes]]
> - [[przygotowanie-automatyzacji|Przygotowanie Środowiska Automatyzacji]]
> - [[konfiguracja-ansible|Konfiguracja Ansible i Komunikacji między Węzłami]]
> - [[przygotowanie-hostow-k3s|Przygotowanie Hostów pod Instalację K3s]]
> - [[instalacja-k3s-control-plane|Instalacja K3s Control Plane]]
> - [[automatyzacja-k3s-ansible|Automatyzacja K3s przy użyciu Ansible Roles]]
> - [[podstawowe-obiekty-kubernetes|Podstawowe Obiekty Kubernetes]]

---

## 1. K3s domyślnie wybiera nieprawidłowy interfejs sieciowy

W środowisku VirtualBox maszyny posiadają dwa interfejsy sieciowe:

| Interfejs | Adres IP      | Przeznaczenie |
| --------- | ------------- | ------------- |
| `enp0s3`  | `10.0.2.15`   | NAT           |
| `enp0s8`  | `192.168.56.x` | Host-Only     |

Podczas standardowej instalacji K3s serwer automatycznie wybiera adres `10.0.2.15` jako `Internal-IP`, co powoduje problem z komunikacją agentów oraz niestabilne działanie klastra.

### Rozwiązanie

Podczas instalacji serwera należy jawnie wskazać adres:

```bash
INSTALL_K3S_EXEC="server \
  --node-ip=192.168.56.102 \
  --advertise-address=192.168.56.102"
```

Podczas instalacji agentów analogicznie:

```bash
INSTALL_K3S_EXEC="agent --node-ip=<IP_NODE>"
```

Po poprawnej instalacji:

```bash
sudo k3s kubectl get nodes -o wide
```

powinno pokazywać `INTERNAL-IP` `192.168.56.102`, `192.168.56.103`, `192.168.56.104`.

> [!NOTE] Formalizacja
>
> Decyzja sformalizowana w [[ADR-001 — Wymuszenie adresu IP Host-Only dla K3s|ADR-001]]. Procedura naprawcza opisana w [[Troubleshooting]] sekcja „K3s serwer reklamuje adres NAT".

---

## 2. Ekran logowania Ubuntu może wprowadzać w błąd

Po uruchomieniu VM Ubuntu wyświetla:

```text
IPv4 address for enp0s3: 10.0.2.15
```

Nie oznacza to, że system nie posiada poprawnego adresu Host-Only. Komunikat odnosi się wyłącznie do pierwszego interfejsu.

Przed instalacją klastra należy zawsze sprawdzić pełną konfigurację sieciową:

```bash
hostname -I
```

lub:

```bash
ip addr
```

---

## 3. Komunikat „Starting k3s-agent" nie oznacza zawieszenia

Podczas instalacji agentów proces często zatrzymuje się na:

```text
[INFO] systemd: Starting k3s-agent
```

przez kilkadziesiąt sekund lub kilka minut. **Nie należy w tym momencie przerywać instalacji.**

W tle wykonywane są:

- uruchomienie containerd,
- pobieranie obrazów kontenerów,
- rejestracja noda w control-plane,
- konfiguracja CNI.

Weryfikację należy wykonywać na serwerze:

```bash
sudo k3s kubectl get nodes
```

---

## 4. Snapshoty VirtualBox znacząco przyspieszają eksperymenty

Przed każdym większym etapem konfiguracji warto utworzyć migawkę. Pozwala to wrócić do stabilnego stanu w kilkanaście sekund zamiast ponownej instalacji.

Przykładowe punkty kontrolne wykorzystane w projekcie:

| Snapshot                  | Opis                              |
| ------------------------- | --------------------------------- |
| `clean-ubuntu-24.04`      | czysta instalacja Ubuntu          |
| `k3s-prepared`            | system przygotowany pod K3s       |
| `k3s-control-plane-ready` | działający serwer K3s             |
| `k3s-ansible-cluster-ready`| pełny klaster po automatyzacji    |

> [!NOTE] Formalizacja
>
> Decyzja sformalizowana w [[ADR-003 — Strategia snapshotów VirtualBox jako punkty kontrolne|ADR-003]].

---

## 5. Po restarcie VM należy sprawdzać stan klastra

Po przywróceniu maszyny lub restarcie hosta należy zweryfikować:

```bash
sudo systemctl status k3s
sudo k3s kubectl get nodes
```

Przez pierwsze kilkadziesiąt sekund node może mieć status `NotReady`. Jest to normalne podczas startu komponentów Kubernetes — po uruchomieniu kubeleta i sieci CNI stan powinien przejść na `Ready`.

---

## 6. Token klastra należy zachować przed instalacją agentów

Token znajduje się na serwerze:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

Jest wymagany do dołączania kolejnych workerów do klastra.

> [!NOTE] Automatyzacja
>
> W docelowym rozwiązaniu token pobierany jest automatycznie z serwera przez Ansible (`slurp` + `b64decode`). Detale w [[automatyzacja-k3s-ansible|Automatyzacji K3s]] oraz [[ADR-002 — Automatyczne pobieranie tokena klastra K3s przez Ansible|ADR-002]].

---

## 7. Git i snapshoty rozwiązują różne problemy

| Mechanizm            | Co zachowuje                                        |
| -------------------- | --------------------------------------------------- |
| Snapshot VirtualBox  | Cały stan maszyny (OS, oprogramowanie, konfiguracja) |
| Git                  | Kod projektu i pliki w repozytorium                 |

Przywrócenie starszej migawki może spowodować utratę:

- zainstalowanego Ansible,
- kluczy SSH,
- konfiguracji systemowej,
- działających usług K3s.

Nie powoduje jednak utraty kodu w repozytorium Git, pod warunkiem że wszystkie zmiany zostały wcześniej zapisane i wypchnięte do GitHub.

### Dobra praktyka

- **Przed wykonaniem snapshotu wykonaj commit.**
- **Po zakończeniu większego etapu wykonaj `git push`.**
- **Snapshot traktuj jako kopię środowiska.**
- **Git traktuj jako kopię kodu projektu.**

Pozwala to bezpiecznie eksperymentować z konfiguracją klastra i szybko wracać do stabilnych punktów kontrolnych.

---

## Przydatne polecenia weryfikacyjne

| Cel                                        | Polecenie                                            |
| ------------------------------------------ | --------------------------------------------------- |
| Stan nodów                                 | `sudo k3s kubectl get nodes -o wide`                |
| Stan podów we wszystkich namespace         | `sudo k3s kubectl get pods -A`                      |
| Stan usługi K3s                            | `sudo systemctl status k3s`                          |
| Konfiguracja IP węzła                       | `hostname -I` / `ip addr`                           |
| Sprawdzenie aktywnej pamięci swap          | `swapon --show`                                     |
| Pobranie tokena klastra (z Control Plane)  | `sudo cat /var/lib/rancher/k3s/server/node-token`   |

---

## Podsumowanie

Ukończono następujące etapy:

- przygotowanie środowiska laboratoryjnego (wirtualizacja VirtualBox),
- klonowanie i konfiguracja trzech węzłów Ubuntu,
- instalacja i konfiguracja Ansible,
- wyłączenie swap i przygotowanie hostów pod K3s,
- ręczna instalacja K3s Control Plane wraz z diagnozą problemów NAT,
- pełna automatyzacja instalacji K3s Ansible Roles,
- test odtworzeniowy klastra,
- demonstracja podstawowych obiektów Kubernetes (Namespace, Deployment, Service, Ingress).

Środowisko jest gotowe do kolejnego etapu — integracji z pipeline CI/CD (GitHub Actions) i wdrażania docelowej aplikacji przez Docker Hub na klaster K3s. Architektura docelowa opisana jest w [[architektura|Architekturze Docelowej Rozwiązania]].