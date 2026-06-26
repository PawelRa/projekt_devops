# Przygotowanie Hostów pod Instalację K3s

## Cel

Celem etapu było przygotowanie systemów operacyjnych trzech serwerów do instalacji lekkiej dystrybucji Kubernetes — K3s.

Przygotowanie obejmowało:

- aktualizację systemów operacyjnych,
- instalację podstawowych narzędzi administracyjnych,
- wyłączenie mechanizmu swap,
- weryfikację konfiguracji hostów.

> [!INFO] Kontekst
>
> Ten etap kontynuuje prace z [[konfiguracja-ansible|Konfiguracji Ansible i Komunikacji między Węzłami]], gdzie skonfigurowano Ansible, inventory oraz shadowy playbook `prepare-k3s.yml` wykrywający aktywny swap. Wykryty swap musiał zostać wyłączony przed dalszą instalacją klastra.

---

## Przestroga dotycząca katalogu projektu

> [!WARNING] Zmiana lokalizacji projektu
>
> Wãskeśnie w tym etapie katalog projektu uległ zmianie z `/opt/devops-project` na `/opt/projekt_devops/ansible`. Wszystkie dalsze ścieżki w dokumentacji odnoszą się do nowej lokalizacji. Szczegóły w [[konfiguracja-ansible|Konfiguracji Ansible]] (callout o zmianie katalogu).

---

## Playbook `prepare-k3s.yml`

Playbook znajduje się w:

```
/opt/projekt_devops/ansible/playbooks/prepare-k3s.yml
```

Zakres działania playbooka:

- aktualizacja repozytoriów APT,
- aktualizacja pakietów systemowych,
- instalacja podstawowych narzędzi administracyjnych,
- sprawdzenie stanu swap,
- wyłączenie swap,
- usunięcie wpisu swap z `/etc/fstab`.

### Instalowane narzędzia

| Pakiet     | Przeznaczenie                              |
| ---------- | ------------------------------------------ |
| `curl`     | Transfer danych (m.in. instalator K3s)    |
| `wget`     | Alternatywne pobieranie plików            |
| `vim`      | Edytor tekstu                              |
| `git`      | Kontrola wersji                            |
| `unzip`    | Rozpakowywanie archiwów ZIP               |
| `net-tools`| Klasyczne narzędzia sieciowe (`ifconfig`)  |
| `htop`     | Interaktywny monitor procesów             |
| `jq`       | Przetwarzanie JSON z CLI                   |

---

## Wyłączenie Swap

> [!WARNING] Swap a Kubernetes
>
> Aktywna pamięć wymiany jest niezgodna z wymaganiami Kubernetes. Kubelet zarządza przydziałem pamięci do podów i obecność swap może prowadzício do niestabilnego działania harmonogramu (scheduler) oraz operacji ewakuacji (eviction). Dlatego przed instalacją klastra K3s swap musi zostać trwale wyłączony.

Przed wykonaniem playbooka system Ubuntu posiadał aktywny swap:

```text
/swap.img  file  3.8G
```

### Automatyzacja wyłączenia

Playbook realizuje dwie operacje:

1. Natychmiastowe wyłączenie swap w bieżącej sesji:

```bash
swapoff -a
```

2. Trwałe wyłączenie swap po restarcie poprzez modyfikację pliku `/etc/fstab` — zakomentowanie wpisu odpowiedzialnego za montowanie pliku swap.

### Weryfikacja

Po wykonaniu playbooka przeprowadzono weryfikację na wszystkich hostach:

```bash
ansible all -i inventory.ini -a "swapon --show" -K
```

Wynik nie zwrócił żadnych wpisów, co potwierdziło poprawne wyłączenie pamięci wymiany na wszystkich serwerach.

---

## Uruchomienie Playbooka

```bash
ansible-playbook -i inventory.ini playbooks/prepare-k3s.yml -K
```

Wynik:

```text
k8s-control-01  OK
k8s-node-01     OK
k8s-node-02     OK
```

---

## Idempotentność

Ponowne uruchomienie playbooka nie wprowadza żadnych zmian w systemie —Ansible wykrywa, że swap jest już wyłączony, a pakiety zainstalowane. Potwierdza to idempotentność konfiguracji i pozwala bezpiecznie uruchamiać playbook wielokrotnie.

---

## Stan Środowiska po Zakończeniu Etapu

| Host           | Swap      | Narzędzia | System aktualny |
| -------------- | --------- | --------- | --------------- |
| k8s-control-01 | wyłączony | tak       | tak             |
| k8s-node-01    | wyłączony | tak       | tak             |
| k8s-node-02    | wyłączony | tak       | tak             |

Wszystkie hosty zostały przygotowane do instalacji klastra K3s.

---

## Migawka (Snapshot)

Po zakończeniu przygotowania hostów utworzono migawkę na wszystkich maszynach:

```
k3s-prepared
```

Migawka stanowi punkt przywracania przed instalacją klastra Kubernetes. Pełna lista migawek znajduje się w [[specyfikacja-maszyn|Specyfikacji Maszyn]].

---

## Następny Etap

Środowisko zostało przygotowane do instalacji klastra K3s. Pierwszym krokiem jest ręczna instalacja i weryfikacja Control Plane opisana w [[instalacja-k3s-control-plane|Instalacji K3s Control Plane]].