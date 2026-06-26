# ADR-001: Wymuszenie adresu IP sieci Host-Only dla K3s

> **Status:** Zaakceptowana  
> **Data:** 2026-06-22  
> **Autor:** Paweł  
> **Zastępuje:** —

---

## Kontekst

Maszyny wirtualne VirtualBox w projekcie posiadają dwa interfejsy sieciowe:

| Interfejs | Adres IP      | Przeznaczenie                          |
| --------- | ------------- | -------------------------------------- |
| `enp0s3`  | `10.0.2.15`   | NAT — dostęp do Internetu              |
| `enp0s8`  | `192.168.56.x` | Host-Only — komunikacja w klastrze    |

Podczas standardowej instalacji K3s serwer automatycznie wybiera adres interfejsu NAT (`10.0.2.15`) jako `Internal-IP` i reklamuje go jako adres API servera (`advertise-address`). W efekcie agenci (workery) próbują łączyć się z API pod `10.0.2.15:6443` i otrzymują:

```text
dial tcp 10.0.2.15:6443: connect: connection refused
```

rutyna trafia w adapter NAT, na którym port 6443 nie jest wystawiony. Węzły nie mogą dołączyć do klastra, mimo że instalacja agentów kończy się pozornym sukcesem.

Decyzja architektoniczna musi określić sposób konfiguracji adresacji IP dla K3s w środowisku wielointerfejsowym, tak aby klaster działał stabilnie i był powtarzalnie instalowany przez Ansible.

---

## Decyzja

Podczas instalacji K3s Server i K3s Agent **jawnie wymusza się adres IP sieci Host-Only** (`192.168.56.0/24`) poprzez flagi instalatora:

- **Server:** `--node-ip` oraz `--advertise-address` ustawione na adres Host-Only węzła (`192.168.56.102`),
- **Agent:** `--node-ip` ustawiony na adres Host-Only danego workera (`192.168.56.103`, `192.168.56.104`).

Adresy IP są przekazywane z `inventory.ini` przez zmienną hosta `k3s_node_ip`, co pozwala na jednoоконą konfigurację z poziomu Ansible.

```yaml
INSTALL_K3S_EXEC: >-
  server
  --node-ip={{ k3s_node_ip }}
  --advertise-address={{ k3s_node_ip }}
```

```yaml
INSTALL_K3S_EXEC: >-
  agent
  --node-ip={{ k3s_node_ip }}
```

---

## Rozważane Alternatywy

| Opcja | Zalety | Wady | Powód odrzucenia |
|---|---|---|---|
| Domyślna instalacja K3s bez flag adresowych | Najprostsza | Niestabilny klaster, agents nie łączą się z API | Powoduje opisany problem `connection refused` |
| Zmiana domyślnej trasy routingu (default route) przez Host-Only | Wymusza wybór adresu Host-Only przez K3s | Psuje dostęp do Internetu po NAT, wymaga dodatkowej konfiguracji sieci | Zbyt inwazyjne dla laboratoryjnego środowiska |
| Konfiguracja `flannel-iface` w K3s | Pozwala wskazać interfejs CNI | Rozwiązuje tylko problem flannel, ale nie `advertise-address` API | Niepełne rozwiązanie, wciąż wymaga wymuszenia `advertise-address` |
| **Wymuszenie `--node-ip` i `--advertise-address` na adres Host-Only** | Precyzyjne, idempotentne, pasuje do modelu Ansible, spójne przez Server i Agent | Wymaga jawnego podania adresów w inventory | **Wybrano** — najprostsze, w pełni kontrolowane i powtarzalne |

---

## Konsekwencje

### Pozytywne

- Stabilna komunikacja wewnątrz klastra przez przewidywalne adresy Host-Only.
- Powtarzalna instalacjaansible idempotentna — identyczna konfiguracja na wszystkich węzłach.
- Czytelna diagnostyka: `kubectl get nodes -o wide` pokazuje poprawne `INTERNAL-IP` `192.168.56.x`.
- Brak zależności od adresu `10.0.2.15`, który jest stały we wszystkich klonach VirtualBox (mógłby kolidować w rozbudowie środowiska).

### Negatywne / Ryzyka

- Wymaga utrzymania zmiennej hosta `k3s_node_ip` w `inventory.ini` dla każdego hosta.
- W przypadku zmiany podsieci Host-Only trzeba jednolicie aktualizować adresy w inventory i sieciowym VirtualBox — zostało to zminimalizowane poprzez statyczną adresację opisaną w [[sieć|Konfiguracji Sieci]].

### Wymagane działania

- [x] Utrzymywać `k3s_node_ip` w `inventory.ini` dla każdego hosta.
- [x] Weryfikować `INTERNAL-IP` po każdej instalacji: `sudo k3s kubectl get nodes -o wide`.
- [x] Procedura naprawcza opisana w [[Troubleshooting]] sekcja „K3s serwer reklamuje adres NAT".

---

## Powiązania

- [[instalacja-k3s-control-plane|Instalacja K3s Control Plane]] — opis diagnozy problemu.
- [[automatyzacja-k3s-ansible|Automatyzacja K3s przy użyciu Ansible Roles]] — implementacja.
- [[lessons-learned|Lekcje Wyciągnięte]] sekcja 1.