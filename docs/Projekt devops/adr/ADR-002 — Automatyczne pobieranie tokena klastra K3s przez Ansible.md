# ADR-002: Automatyczne pobieranie tokena klastra K3s przez Ansible

> **Status:** Zaakceptowana  
> **Data:** 2026-06-22  
> **Autor:** Paweł  
> **Zastępuje:** —

---

## Kontekst

K3s zabezpiecza komunikację między Control Plane a workerami tokenem (cluster token) przechowywanym na serwerze w `/var/lib/rancher/k3s/server/node-token`. Token jest wymagany do dołączenia agenta do klastra (`K3S_TOKEN`). W środowisku laboratoryjnym początkowo kopiowano token ręcznie między serwerami, co polegało na kilkukrotnym podawaniu haseł i edycjach poleceń instalacyjnych.

Decyzja architektoniczna musi określić sposób, w jaki Ansible pozyskuje i przekazuje token do ról `k3s_agent` na workerach.

---

## Decyzja

Token jest **pobierany automatycznie z Control Plane w trakcie uruchamiania playbooka `install-k3s-agent.yml`**, wykorzystując moduł `slurp` działający na serwerze Control Plane (`delegate_to`), a następnie dekodowany (base64) w pamięci i przekazywany do instalatora na workerach.

```yaml
- name: Pobranie tokena z Control Plane
  slurp:
    src: /var/lib/rancher/k3s/server/node-token
  delegate_to: "{{ groups['k3s_server'][0] }}"
  register: k3s_token_raw

- name: Dekodowanie tokena
  set_fact:
    k3s_token: "{{ k3s_token_raw.content | b64decode | trim }}"
```

Token jest przekazywany instalatorowi jako zmienna środowiskowa `K3S_TOKEN`. Nie jest zapisywany na dysku workerów ani w repozytorium Git.

---

## Rozważane Alternatywy

| Opcja | Zalety | Wady | Powód odrzucenia |
|---|---|---|---|
| Ręczne kopiowanie tokena (SSH + paste) | Brak konfiguracji Ansible | Błędy, trudności powtarzania, psuje idempotentność | Niedopuszczalne w docelowym rozwiązaniu automatycznym |
| Hardkodowanie tokena w `group_vars/all.yml` | Centralna konfiguracja | Token w repozytorium = sekret w Git; zmiana przy przebudowie klastra | Narusza zasady bezpieczeństwa, token krótko żyjący |
| Przechowywanie tokena przez Ansible Vault | Standardowe bezpieczeństwo Ansible | Wymaga rozproszenia pliku vault między wykonawcami i zarządzania hasłem | W overkill dla laboratorium; token i tak żyje krótko |
| **Automatyczne pobieranie `slurp` + `delegate_to`** | Całkowita automatyzacja, brak sekretów w repo, [[ADR-003 — Strategia snapshotów VirtualBox jako punkty kontrolne\|odtworzenie]] klastra jednym poleceniem, token nie trafia na dysk niezarządzający | Token przesyłany przez Ansible w pamięci (akceptowalne w labie) | **Wybrano** — najprostsze w pełni automatyczne rozwiązanie |

---

## Konsekwencje

### Pozytywne

- Instalacja workerów odbywa się jednym poleceniem `ansible-playbook` bez ręcznej interwencji.
- Token nie jest przechowywany w repozytorium Git ani w `group_vars`.
- Test odtworzeniowy klastra od zera (snapshot + clone repo + playbook) prawidłowo dołączył workery bez dodatkowych przygotowań.
- Brak ryzyka wycieku tokena w systemie kontroli wersji.

### Negatywne / Ryzyka

- Worker musi łączyć się z Control Plane przez użytkownika SSH pasującym na obu maszynach (realizowane przez konfigurację SSH w [[konfiguracja-ansible|Konfiguracji Ansible]]).
- Token żyje w pamięci Ansible podręcznie executora na czas wykonania playbooka — akceptowalne w laboratorium, w produkcji rozważyć Ansible Vault lub system sekretów (np. HashiCorp Vault).

### Wymagane działania

- [x] Zachować kolejność: najpierw playbook `install-k3s-server.yml`, potem `install-k3s-agent.yml`.
- [x] Gdy Control Plane jest przebudowany (nowy token), ponowne uruchomienie `install-k3s-agent.yml` pobierze go automatycznie.
- [x] W produkcji zaplanować rotację i bezpieczne przechowywanie tokena.

---

## Powiązania

- [[automatyzacja-k3s-ansible|Automatyzacja K3s przy użyciu Ansible Roles]] — implementacja.
- [[instalacja-k3s-control-plane|Instalacja K3s Control Plane]] — pobieranie tokena po raz pierwszy.
- [[lessons-learned|Lekcje Wyciągnięte]] sekcja 6.