# Przygotowanie Węzłów Klastra Kubernetes

## Cel

Celem drugiego etapu było przygotowanie trzech niezależnych serwerów, które zostaną wykorzystane do budowy klastra Kubernetes składającego się z jednego węzła zarządzającego (Control Plane) oraz dwóch węzłów roboczych (Worker Nodes).

W celu przyspieszenia procesu przygotowania środowiska wykorzystano mechanizm klonowania wcześniej przygotowanej maszyny bazowej (zobacz [[przygotowanie-srodowiska|Przygotowanie Środowiska Laboratoryjnego]]).

---

## Klonowanie Maszyny Bazowej

Jako źródło klonowania wykorzystano wcześniej przygotowaną maszynę:

```
k8s-control-01
```

Przed wykonaniem klonowania utworzono migawkę umożliwiającą szybkie odtworzenie środowiska w przypadku wystąpienia problemów.

Utworzono dwie nowe maszyny wirtualne:

| Nazwa maszyny | Rola        |
| ------------- | ----------- |
| k8s-node-01   | Worker Node |
| k8s-node-02   | Worker Node |

Podczas procesu klonowania wybrano opcję wygenerowania nowych adresów MAC dla wszystkich kart sieciowych.

---

## Konfiguracja Nazw Hostów

Po pierwszym uruchomieniu sklonowanych maszyn zmieniono nazwy hostów tak, aby każda maszyna posiadała jednoznaczną identyfikację w środowisku.

Docelowa konfiguracja:

| Hostname       | Rola          |
| -------------- | ------------- |
| k8s-control-01 | Control Plane |
| k8s-node-01    | Worker Node   |
| k8s-node-02    | Worker Node   |

Zmiana została wykonana przy użyciu polecenia:

```bash
sudo hostnamectl hostname <nowa_nazwa_hosta>
```

Po zmianie nazwy wykonano restart systemu.

---

## Problem z Identyfikatorami machine-id

Po uruchomieniu wszystkich maszyn jednocześnie zaobserwowano nieprawidłowe działanie usługi DHCP.

Pomimo wygenerowania nowych adresów MAC wszystkie maszyny początkowo otrzymywały identyczne adresy IP w sieci Host-Only.

Przeprowadzona analiza wykazała, że podczas klonowania został skopiowany również identyfikator systemowy:

```
/etc/machine-id
```

Wszystkie maszyny posiadały identyczną wartość tego identyfikatora.

> [!WARNING] Konsekwencje zduplikowanego machine-id
>
> Może to powodować problemy z:
>
> - usługami DHCP,
> - systemd,
> - narzędziami automatyzacji,
> - klastrem Kubernetes.

---

## Regeneracja machine-id

W celu zapewnienia unikalnej identyfikacji każdego serwera wykonano regenerację identyfikatora systemowego.

Na każdej maszynie wykonano polecenia:

```bash
sudo rm /etc/machine-id
sudo systemd-machine-id-setup
sudo reboot
```

Po restarcie zweryfikowano poprawność operacji. Każda maszyna posiadała unikalny identyfikator machine-id.

Po wykonaniu tej operacji usługa DHCP przydzieliła różne adresy IP dla poszczególnych serwerów.

---

## Weryfikacja Konfiguracji Sieciowej

Zweryfikowano poprawność konfiguracji interfejsów sieciowych na wszystkich węzłach klastra.

Adresy przydzielone w sieci Host-Only:

| Host           | Adres IP       |
| -------------- | -------------- |
| k8s-control-01 | 192.168.56.102 |
| k8s-node-01    | 192.168.56.103 |
| k8s-node-02    | 192.168.56.104 |

Adresy zostały przydzielone automatycznie przez serwer DHCP VirtualBox. Szczegółowa konfiguracja sieci opisana jest w [[sieć|Konfiguracji Sieci]].

---

## Test Komunikacji Pomiędzy Węzłami

W celu potwierdzenia poprawności działania sieci przeprowadzono testy komunikacji przy użyciu polecenia `ping`.

Zweryfikowano komunikację pomiędzy wszystkimi węzłami klastra.

Przykładowe wyniki:

```
k8s-control-01 -> k8s-node-01    OK
k8s-control-01 -> k8s-node-02    OK
k8s-node-01    -> k8s-control-01 OK
k8s-node-01    -> k8s-node-02    OK
k8s-node-02    -> k8s-control-01 OK
k8s-node-02    -> k8s-node-01    OK
```

Nie stwierdzono utraty pakietów ani problemów z komunikacją pomiędzy maszynami.

---

## Weryfikacja Dostępu SSH

Zweryfikowano możliwość logowania do wszystkich węzłów z wykorzystaniem protokołu SSH.

Połączenia realizowano z systemu Windows 11 przy użyciu klienta OpenSSH.

Potwierdzono poprawne działanie dostępu administracyjnego do wszystkich serwerów.

---

## Stan Środowiska po Zakończeniu Etapu

Po zakończeniu prac środowisko laboratoryjne składa się z trzech serwerów Ubuntu Server 24.04 LTS połączonych wspólną siecią Host-Only.

Aktualna topologia:

```
Windows 11 Pro
│
└── VirtualBox
    │
    ├── k8s-control-01 (192.168.56.102)
    │
    ├── k8s-node-01    (192.168.56.103)
    │
    └── k8s-node-02    (192.168.56.104)
```

Środowisko zostało przygotowane do instalacji klastra Kubernetes oraz dalszej konfiguracji narzędzi automatyzujących zarządzanie infrastrukturą.

---

## Konfiguracja Statycznych Adresów IP

Po zakończeniu etapu klonowania zrezygnowano z adresacji przydzielanej dynamicznie przez DHCP. Dla zwiększenia przewidywalności konfiguracji oraz ułatwienia zarządzania klastrem Kubernetes skonfigurowano statyczne adresy IP dla interfejsów sieciowych Host-Only (enp0s8). Pełna specyfikacja sieciowa znajduje się w [[sieć|Konfiguracji Sieci]].

| Host           | Interfejs | Adres             |
| -------------- | --------- | ----------------- |
| k8s-control-01 | enp0s8    | 192.168.56.102/24 |
| k8s-node-01    | enp0s8    | 192.168.56.103/24 |
| k8s-node-02    | enp0s8    | 192.168.56.104/24 |

Interfejsy NAT (enp0s3) pozostawiono skonfigurowane przez DHCP w celu zapewnienia dostępu do Internetu.

---

## Następny Etap

Środowisko zostało przygotowane do instalacji narzędzi automatyzujących zarządzanie infrastrukturą. Dalsze prace opisane są w [[przygotowanie-automatyzacji|Przygotowaniu Środowiska Automatyzacji]].
