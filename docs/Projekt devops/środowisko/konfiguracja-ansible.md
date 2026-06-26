# Konfiguracja Ansible i Komunikacji między Węzłami

## Cel

Celem etapu było przygotowanie centralnego systemu automatyzacji umożliwiającego zarządzanie wszystkimi serwerami klastra z jednego miejsca.

Do realizacji tego zadania wykorzystano Ansible działający na serwerze:

```
k8s-control-01
```

> [!NOTE] Kontekst
>
> Ten etap kontynuuje prace rozpoczęte w [[przygotowanie-automatyzacji|Przygotowaniu Środowiska Automatyzacji]], gdzie zainstalowano Ansible i wygenerowano klucz SSH.

---

## Struktura Projektu

Na serwerze zarządzającym utworzono katalog projektu:

```
/opt/devops-project
```

W ramach projektu przygotowano następującą strukturę katalogów:

```
/opt/devops-project
├── ansible
├── docs
├── kubernetes
├── monitoring
└── terraform
```

Poszczególne katalogi będą wykorzystywane odpowiednio do:

| Katalog     | Przeznaczenie                   |
| ----------- | ------------------------------- |
| `ansible`   | Playbooki i konfiguracja Ansible |
| `terraform` | Definicje Infrastructure as Code |
| `kubernetes` | Manifesty i konfiguracja Kubernetes |
| `monitoring` | Konfiguracja Prometheus i Grafana |
| `docs`      | Dokumentacja projektu            |

Struktura katalogów odzwierciedla architekturę docelową projektu opisaną w [[architektura|Architekturze Docelowej Rozwiązania]].

---

## Konfiguracja Uwierzytelniania SSH

W celu umożliwienia bezobsługowej komunikacji pomiędzy serwerem zarządzającym a pozostałymi węzłami skopiowano klucz publiczny wygenerowany w [[przygotowanie-automatyzacji|poprzednim etapie]] na wszystkie hosty zarządzane:

```bash
ssh-copy-id pawel@192.168.56.102
ssh-copy-id pawel@192.168.56.103
ssh-copy-id pawel@192.168.56.104
```

Sprawdzono poprawność logowania:

```bash
ssh pawel@192.168.56.102
ssh pawel@192.168.56.103
ssh pawel@192.168.56.104
```

Po wykonaniu operacji możliwe było logowanie bez podawania hasła na wszystkie węzły klastra.

---

## Konfiguracja Inventory

Utworzono plik:

```
/opt/devops-project/ansible/inventory.ini
```

Zawartość:

```ini
[k3s_server]
192.168.56.102

[k3s_agents]
192.168.56.103
192.168.56.104

[all:vars]
ansible_user=pawel
```

Grupy inventory odpowiadają rolom w klastrze:

| Grupa        | Hosty             | Rola       |
| ------------ | ----------------- | ---------- |
| `k3s_server` | `192.168.56.102`  | Control Plane |
| `k3s_agents` | `192.168.56.103`, `192.168.56.104` | Worker Node |

> [!INFO] Adresacja
>
> Adresy IP odpowiadają statycznej konfiguracji opisanej w [[sieć|Konfiguracji Sieci]].

---

## Konfiguracja group_vars

Utworzono katalog:

```
/opt/devops-project/ansible/group_vars
```

oraz plik:

```yaml
# group_vars/all.yml
ansible_user: pawel
ansible_become: true
ansible_become_method: sudo
```

Przeniesienie wspólnych parametrów do `group_vars` uprościło konfigurację inventory oraz pozwoliło na centralne zarządzanie ustawieniami Ansible.

Po tej zmianie plik `inventory.ini` nie wymagał już sekcji `[all:vars]`:

```ini
[k3s_server]
192.168.56.102

[k3s_agents]
192.168.56.103
192.168.56.104
```

---

## Weryfikacja Działania Ansible

W celu sprawdzenia komunikacji pomiędzy wszystkimi hostami wykonano polecenie:

```bash
ansible all -i inventory.ini -m ping
```

Wynik:

```
192.168.56.102 -> pong
192.168.56.103 -> pong
192.168.56.104 -> pong
```

Potwierdzono poprawne działanie:

- komunikacji SSH,
- konfiguracji inventory,
- środowiska Ansible,
- dostępu do wszystkich serwerów klastra.

---

## Testowe Playbooki

Utworzono zestaw playbooków weryfikujących poprawność środowiska:

| Playbook              | Przeznaczenie                      |
| --------------------- | ---------------------------------- |
| `playbooks/ping.yml`  | Test podstawowej komunikacji       |
| `playbooks/system-info.yml` | Pobieranie faktów systemowych |
| `playbooks/network-info.yml` | Weryfikacja konfiguracji sieci |

### Wynik playbooka network-info.yml

Uruchomiono:

```bash
ansible-playbook -i inventory.ini playbooks/network-info.yml
```

Wynik:

```
k8s-control-01
  10.0.2.15
  192.168.56.102

k8s-node-01
  10.0.2.15
  192.168.56.103

k8s-node-02
  10.0.2.15
  192.168.56.104
```

Interpretacja wyników:

| Interfejs | Adres            | Przeznaczenie                          |
| --------- | ---------------- | -------------------------------------- |
| `enp0s3`  | `10.0.2.15`      | Dostęp do Internetu przez NAT VirtualBox |
| `enp0s8`  | `192.168.56.x`   | Komunikacja między węzłami klastra     |

Adresy `192.168.56.x` będą wykorzystywane przez Kubernetes.

---

## Automatyczne Przygotowanie Hostów

Utworzono playbook:

```
playbooks/prepare-k3s.yml
```

Zakres działania:

- aktualizacja repozytoriów APT,
- aktualizacja pakietów systemowych,
- instalacja podstawowych narzędzi administracyjnych,
- weryfikacja konfiguracji systemu,
- sprawdzenie stanu pamięci swap.

Uruchomienie:

```bash
ansible-playbook -i inventory.ini playbooks/prepare-k3s.yml -K
```

Wynik:

```
k8s-control-01  OK
k8s-node-01     OK
k8s-node-02     OK
```

Podczas weryfikacji wykryto aktywny swap:

```
/swap.img 3.8G
```

> [!WARNING] Swap a Kubernetes
>
> Aktywny partition swap jest niezgodny z wymaganiami Kubernetes i musi zostać wyłączony przed instalacją klastra K3s. Informacja ta zostanie wykorzystana w kolejnym etapie przygotowania hostów do instalacji.

---

## Struktura Projektu Ansible

Po zakończeniu konfiguracji utworzono docelową strukturę katalogów:

```
/opt/devops-project/ansible
├── group_vars
│   └── all.yml
├── inventory.ini
├── playbooks
│   ├── ping.yml
│   ├── system-info.yml
│   ├── network-info.yml
│   └── prepare-k3s.yml
└── roles
    ├── k3s_server
    └── k3s_agent
```

Role `k3s_server` oraz `k3s_agent` zostały utworzone przy użyciu narzędzia:

```bash
ansible-galaxy role init k3s_server
ansible-galaxy role init k3s_agent
```

Przygotowano w ten sposób strukturę zgodną z dobrymi praktykami Ansible, umożliwiającą dalszą automatyzację instalacji klastra Kubernetes.

---

## Stan Środowiska po Zakończeniu Etapu

Na zakończenie etapu środowisko składa się z trzech serwerów Ubuntu Server 24.04 LTS połączonych siecią prywatną oraz centralnego serwera zarządzającego wyposażonego w Ansible.

```
k8s-control-01
├── Ubuntu Server 24.04 LTS
├── Ansible 2.16.3
├── SSH Key Authentication (Ed25519)
├── Inventory configured
└── Role: K3s Server

k8s-node-01
└── Przygotowany host K3s Agent

k8s-node-02
└── Przygotowany host K3s Agent
```

Mapowanie węzłów:

| Host           | Adres IP         | Rola          |
| -------------- | ---------------- | ------------- |
| k8s-control-01 | 192.168.56.102   | Control Plane |
| k8s-node-01    | 192.168.56.103   | Worker Node   |
| k8s-node-02    | 192.168.56.104   | Worker Node   |

> [!WARNING] Zmiana katalogu projektu
>
> W początkowych etapach projekt znajdował się w `/opt/devops-project`. W toku prac katalog został zmieniony na `/opt/projekt_devops/ansible`. Wszystkie dalsze notatki (od [[przygotowanie-hostow-k3s|Przygotowania Hostów pod K3s]]) używają nowej ścieżki. Zmiana nie wpływa na konfigurację Ansible z tego etapu — praktyki i struktura pozostają aktualne.

---

## Następny Etap

Środowisko jest gotowe do rozpoczęcia automatycznej instalacji klastra K3s przy użyciu Ansible. Kolejny etap — wyłączenie swap i przygotowanie hostów pod instalację K3s — opisany jest w [[przygotowanie-hostow-k3s|Przygotowaniu Hostów pod Instalację K3s]]. Przegląd doświadczeń z całego projektu znajduje się w [[lessons-learned|Lekcjach Wyciągniętych]].