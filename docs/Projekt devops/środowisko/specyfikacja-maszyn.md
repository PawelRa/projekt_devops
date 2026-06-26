# Specyfikacja Maszyn

## Host

| Element | Wartość |
|---|---|
| System operacyjny | Windows 11 Pro |
| Oprogramowanie wirtualizacyjne | Oracle VM VirtualBox 7.2.8 |

## Maszyny wirtualne

Wszystkie maszyny pracują na **Ubuntu Server 24.04.2 LTS**.

| Nazwa | CPU | RAM | Dysk | Typ dysku | Alokacja | System | Źródło |
|---|---|---|---|---|---|---|---|
| `k8s-control-01` | 2 vCPU | 4096 MB | 40 GB | VDI | Dynamiczna | Ubuntu Server 24.04.2 LTS | Instalacja z ISO |
| `k8s-node-01` | 2 vCPU | 6144 MB | 40 GB | VDI | Dynamiczna | Ubuntu Server 24.04.2 LTS | Klon `k8s-control-01` |
| `k8s-node-02` | 2 vCPU | 6144 MB | 40 GB | VDI | Dynamiczna | Ubuntu Server 24.04.2 LTS | Klon `k8s-control-01` |

Proces instalacji maszyny bazowej opisany jest w [[przygotowanie-srodowiska|Przygotowaniu Środowiska Laboratoryjnego]], a proces klonowania w [[przygotowanie-wezlow|Przygotowaniu Węzłów Klastra Kubernetes]].

## Role

| Maszyna | Rola w klastrze |
|---|---|
| `k8s-control-01` | Control Plane |
| `k8s-node-01` | Worker Node |
| `k8s-node-02` | Worker Node |

Opis ról w architekturze klastra znajduje się w [[architektura|Architekturze Docelowej Rozwiązania]].

## Opcje instalacji systemu

| Parametr | Wartość |
|---|---|
| Język instalatora | English |
| Układ klawiatury | Polish Legacy |
| OpenSSH Server | Zainstalowany |
| Uwierzytelnianie hasłem SSH | Włączone |
| Ubuntu Pro | Pominięto |
| Featured Server Snaps | Nie instalowano |
| Partycjonowanie | LVM (automatyczne) |

## Migawki

| Nazwa migawki | Maszyna | Opis |
|---|---|---|
| `clean-ubuntu-24.04` | `k8s-control-01` | Czysty system po instalacji — punkt przywracania |
| `before-clone` | `k8s-control-01` | Stan przed klonowaniem — punkt przywracania |
| `clean-ubuntu-24.04` | `k8s-node-01` | Czysty system po klonowaniu — punkt przywracania |
| `clean-ubuntu-24.04` | `k8s-node-02` | Czysty system po klonowaniu — punkt przywracania |

Migawka `clean-ubuntu-24.04` na maszynie `k8s-control-01` została utworzona w [[przygotowanie-srodowiska|Przygotowaniu Środowiska Laboratoryjnego]], a migawka `before-clone` przed rozpoczęciem klonowania opisanego w [[przygotowanie-wezlow|Przygotowaniu Węzłów Klastra Kubernetes]].
