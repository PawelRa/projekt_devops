# Przygotowanie Środowiska Automatyzacji

## Cel

Celem etapu było przygotowanie środowiska umożliwiającego automatyzację zarządzania infrastrukturą oraz konfiguracją serwerów.

Do realizacji tego zadania wybrano narzędzie **Ansible**, które będzie wykorzystywane do zarządzania konfiguracją systemów operacyjnych oraz instalacji komponentów klastra Kubernetes. Rola Ansible w architekturze projektu opisana jest w [[architektura#Zarządzanie Konfiguracją (Ansible)|Architekturze Docelowej]].

> [!INFO] Kontekst
>
> Ten etap kontynuuje prace z [[przygotowanie-wezlow|Przygotowania Węzłów Klastra Kubernetes]], gdzie przygotowano trzy serwery połączone siecią Host-Only.

---

## Instalacja Ansible

Instalację wykonano na serwerze pełniącym rolę węzła zarządzającego:

```
k8s-control-01
```

Przed instalacją zaktualizowano listę pakietów systemowych:

```bash
sudo apt update
```

Następnie zainstalowano pakiet Ansible z oficjalnych repozytoriów Ubuntu:

```bash
sudo apt install -y ansible
```

Po zakończeniu instalacji zweryfikowano poprawność działania narzędzia.

Wynik polecenia:

```bash
ansible --version
```

Potwierdził instalację:

```
ansible [core 2.16.3]
python version = 3.12.3
```

---

## Koncepcja Wykorzystania Ansible

W projekcie Ansible będzie pełnił rolę centralnego narzędzia automatyzującego konfigurację środowiska.

Zakres zastosowania obejmuje:

- zarządzanie konfiguracją serwerów,
- instalację komponentów Kubernetes,
- przygotowanie systemów operacyjnych,
- wdrażanie zmian konfiguracyjnych na wielu serwerach jednocześnie.

Podejście to pozwala ograniczyć liczbę ręcznie wykonywanych operacji administracyjnych oraz zapewnia powtarzalność procesu wdrażania.

---

## Przygotowanie Uwierzytelniania SSH

W celu umożliwienia bezobsługowej komunikacji pomiędzy serwerem Ansible a zarządzanymi hostami wygenerowano dedykowaną parę kluczy SSH.

Generowanie klucza wykonano na serwerze:

```
k8s-control-01
```

Przy użyciu polecenia:

```bash
ssh-keygen -t ed25519 -C "ansible@k8s-control-01"
```

Zastosowano algorytm:

```
Ed25519
```

Ze względu na laboratoryjny charakter środowiska klucz prywatny nie został zabezpieczony dodatkowym hasłem, co umożliwi późniejsze bezobsługowe wykonywanie playbooków Ansible.

Po wygenerowaniu klucza utworzone zostały pliki:

| Plik                  | Przeznaczenie  |
| --------------------- | -------------- |
| `~/.ssh/id_ed25519`   | Klucz prywatny |
| `~/.ssh/id_ed25519.pub` | Klucz publiczny |

> [!NOTE] Bezpieczeństwo kluczy SSH
>
> W środowisku produkcyjnym zaleca się zabezpieczenie klucza prywatnego dodatkowym hasłem (passphrase) oraz wykorzystanie narzędzia `ssh-agent` do zarządzania uwierzytelnianiem. W laboratorium odstąpiono od tego wymogu dla uproszczenia automatyzacji.

---

## Stan Środowiska po Zakończeniu Etapu

Po zakończeniu prac środowisko posiada:

```
k8s-control-01
├── Ubuntu Server 24.04 LTS
├── OpenSSH Server
├── Ansible 2.16.3
└── Klucz SSH Ed25519
```

Środowisko jest przygotowane do konfiguracji komunikacji SSH pomiędzy wszystkimi węzłami klastra oraz dalszej automatyzacji z wykorzystaniem Ansible. Szczegółowa konfiguracja opisana jest w [[konfiguracja-ansible|Konfiguracja Ansible i Komunikacji między Węzłami]].