# Przygotowanie Środowiska Laboratoryjnego

## Cel

Celem pierwszego etapu projektu było przygotowanie bazowego środowiska laboratoryjnego, które będzie wykorzystywane do budowy klastra Kubernetes oraz wdrażania narzędzi automatyzujących pracę administratora i inżyniera DevOps.

Środowisko zostało przygotowane lokalnie z wykorzystaniem wirtualizacji, co pozwala na pełną kontrolę nad konfiguracją oraz eliminuje koszty związane z wykorzystaniem usług chmurowych.

---

## Środowisko Hosta

| Element                        | Wartość                              |
| ------------------------------ | ------------------------------------ |
| System operacyjny hosta        | Windows 11 Pro                       |
| Oprogramowanie wirtualizacyjne | Oracle VM VirtualBox 7.2.8           |
| Typ środowiska                 | Lokalna infrastruktura laboratoryjna |

---

## Pobranie Systemu Operacyjnego

| Parametr     | Wartość        |
| ------------ | -------------- |
| Dystrybucja  | Ubuntu Server  |
| Wersja       | 24.04.2 LTS    |
| Architektura | amd64 (x86_64) |

Wybrano Ubuntu Server ze względu na szerokie wsparcie społeczności, dużą ilość dokumentacji oraz popularność tego systemu w środowiskach DevOps i Kubernetes.

---

## Utworzenie Maszyny Wirtualnej (Control Plane)

| Parametr            | Wartość                   |
| ------------------- | ------------------------- |
| Nazwa maszyny       | k8s-control-01            |
| System operacyjny   | Ubuntu Server 24.04.2 LTS |
| Procesory wirtualne | 2 vCPU                    |
| Pamięć RAM          | 4096 MB                   |
| Dysk wirtualny      | 40 GB                     |
| Typ dysku           | VDI                       |
| Alokacja dysku      | Dynamiczna                |

Pełna specyfikacja wszystkich maszyn znajduje się w [[specyfikacja-maszyn|Specyfikacji Maszyn]].

---

## Konfiguracja Sieci

Maszynę wyposażono w dwa interfejsy sieciowe.

### Adapter 1 — NAT

Przeznaczenie:

- dostęp do Internetu,
- pobieranie aktualizacji systemu,
- pobieranie pakietów i obrazów kontenerów.

Adres przydzielony po instalacji:

```
10.0.2.15/24
```

### Adapter 2 — Host-Only Network

Przeznaczenie:

- komunikacja pomiędzy węzłami klastra Kubernetes,
- połączenia SSH z systemu hosta,
- ruch administracyjny wewnątrz środowiska laboratoryjnego.

Konfiguracja sieci Host-Only:

```
Sieć: 192.168.56.0/24
Host: 192.168.56.1
DHCP: włączony
```

Adres przydzielony po instalacji:

```
192.168.56.101/24
```

---

Szczegółowa konfiguracja sieci opisana jest w [[sieć|Konfiguracji Sieci]].

---

## Instalacja Systemu Ubuntu Server

Podczas instalacji zastosowano następujące ustawienia:

| Parametr                    | Wartość         |
| --------------------------- | --------------- |
| Język instalatora           | English         |
| Układ klawiatury            | Polish Legacy   |
| Typ instalacji              | Ubuntu Server   |
| Ubuntu Pro                  | Pominięto       |
| OpenSSH Server              | Zainstalowano   |
| Import kluczy SSH           | Pominięto       |
| Uwierzytelnianie hasłem SSH | Włączone        |
| Featured Server Snaps       | Nie instalowano |

Dysk został skonfigurowany automatycznie przy użyciu mechanizmu **LVM**.

---

## Weryfikacja Poprawności Instalacji

Po zakończeniu instalacji wykonano podstawowe testy poprawności działania systemu.

Zweryfikowano:

- poprawność ustawienia nazwy hosta,
- poprawność konfiguracji interfejsów sieciowych,
- dostęp do Internetu,
- możliwość aktualizacji pakietów systemowych,
- działanie usługi OpenSSH,
- możliwość logowania przez SSH z systemu Windows.

Potwierdzono poprawne działanie połączenia SSH:

```bash
ssh pawel@192.168.56.101
```

---

## Wyniki Weryfikacji Konfiguracji Sieciowej

Po zakończeniu instalacji zweryfikowano konfigurację interfejsów sieciowych przy użyciu poleceń `ip -4 addr` oraz `ip route`.

| Interfejs | Adres IPv4        | Przeznaczenie                                     |
| --------- | ----------------- | ------------------------------------------------- |
| enp0s3    | 10.0.2.15/24      | Dostęp do Internetu (NAT)                         |
| enp0s8    | 192.168.56.101/24 | Sieć klastra i dostęp administracyjny (Host-Only) |

Domyślna trasa routingu została skonfigurowana przez interfejs NAT:

```
default via 10.0.2.2 dev enp0s3
```

Potwierdzono poprawny dostęp do sieci zewnętrznej oraz komunikację z hostem.

---

## Weryfikacja Systemu Operacyjnego

Zweryfikowano poprawność identyfikacji systemu przy użyciu polecenia `hostnamectl`.

| Parametr                  | Wartość                 |
| ------------------------- | ----------------------- |
| Hostname                  | k8s-control-01          |
| System operacyjny         | Ubuntu 24.04.4 LTS      |
| Kernel                    | Linux 6.8.0-124-generic |
| Architektura              | x86_64                  |
| Platforma wirtualizacyjna | Oracle VirtualBox       |

Potwierdzono poprawne ustawienie nazwy hosta oraz prawidłowe działanie systemu po aktualizacji.

---

## Migawka (Snapshot)

Po zakończeniu konfiguracji bazowej utworzono migawkę maszyny wirtualnej:

```
clean-ubuntu-24.04
```

Migawka stanowi punkt przywracania umożliwiający szybki powrót do stabilnej konfiguracji w przypadku problemów podczas dalszych etapów projektu. Lista wszystkich migawek znajduje się w [[specyfikacja-maszyn|Specyfikacji Maszyn]].

> [!NOTE] Zmiana adresu IP
>
> W wyniku procesu klonowania i regeneracji identyfikatorów `machine-id` w drugim etapie, adres `k8s-control-01` w sieci Host-Only uległ zmianie z `192.168.56.101` na `192.168.56.102`. Szczegóły opisano w [[przygotowanie-wezlow|Przygotowanie Węzłów Klastra Kubernetes]].

---

## Następny Etap

Przygotowanie maszyny bazowej zostało zakończone. Dalsze prace — klonowanie węzłów i konfiguracja sieci klastra — opisane są w [[przygotowanie-wezlow|Przygotowaniu Węzłów Klastra Kubernetes]].
