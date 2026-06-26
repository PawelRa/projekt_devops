# ADR-003: Strategia snapshotów VirtualBox jako punkty kontrolne

> **Status:** Zaakceptowana  
> **Data:** 2026-06-22  
> **Autor:** Paweł  
> **Zastępuje:** —

---

## Kontekst

Projekt prowadzony jest na trzech maszynach wirtualnych VirtualBox. Konfiguracja klastra K3s składa się z wielokrotnych, zależnych od siebie etapów: przygotowanie systemu → Ansible → wyłączenie swap → instalacja Control Plane → instalacja agentów → wdrożenie aplikacji. W razie błędu lub eksperymentu powrót do stabilnego stanu poprzez ponowną instalację Ubuntu, Ansible, SSH i K3s trwa długo i jest powtarzalne żmudny.

Mamy dwa mechanizmy przywracania dostępne w projekcie:

| Mechanizm            | Co zachowuje                                        |
| -------------------- | --------------------------------------------------- |
| Snapshot VirtualBox  | Cały stan maszyny (OS, oprogramowanie, konfiguracja) |
| Git                  | Kod projektu i pliki w repozytorium                 |

Decyzja architektoniczna musi określić strategię używania snapshotów VirtualBox jako punktów kontrolnych w laboratorium.

---

## Decyzja

Przyjęto strategię **tworzenia migawek VirtualBox na zakończenie każdego stabilnego etapu konfiguracji**, traktowanych jako punkt przywracania laboratorium. Zasady:

- Snapshot VirtualBox tworzy pełną kopię dysku i pamięci VM, pozwalając wrócić do stabilnego stanu w kilkanaście sekund.
- **Przed wykonaniem snapshotu obowiązkowo wykonuje się `git commit` i `git push`** — snapshot traktowany jest jako kopia środowiska, Git jako kopia kodu projektu.
- Snapshot nie zastępuje Git — przywrócenie starej migawki może cofnąć zainstalowany Ansible, klucze SSH, konfigurację systemową lub działające usługi K3s, ale nie narusza repozytorium.
- Test odtworzeniowy klastra (przywrócenie snapshotu + klonowanie repo + uruchomienie playbooka) jest kanoniczną metodą weryfikacji idempotentności automatyzacji.

### Punkty kontrolne w projekcie

| Snapshot                  | Opis                              | Etap                                                |
| ------------------------- | --------------------------------- | --------------------------------------------------- |
| `clean-ubuntu-24.04`      | czysta instalacja Ubuntu          | [[przygotowanie-srodowiska\|Przygotowanie Środowiska Laboratoryjnego]] |
| `k3s-prepared`            | system przygotowany pod K3s       | [[przygotowanie-hostow-k3s\|Przygotowanie Hostów pod K3s]]           |
| `k3s-control-plane-ready` | działający serwer K3s (Control Plane) | [[instalacja-k3s-control-plane\|Instalacja K3s Control Plane]]       |
| `k3s-ansible-cluster-ready`| pełny klaster po automatyzacji    | [[automatyzacja-k3s-ansible\|Automatyzacja K3s przy użyciu Ansible Roles]] |

---

## Rozważane Alternatywy

| Opcja | Zalety | Wady | Powód odrzucenia |
|---|---|---|---|
| Brak snapshotów — reinstallacja w razie problemu | Brak zajętości dysku hosta | Koszt czasowy, błędy powtórzone, trudne eksperymentowanie | Niedopuszczalne przy iteracyjnym rozwoju klastra |
| Tylko Git, bez snapshotów | Wszystko jako kod | Nie odtwarza stanu VM, kluczy SSH, zainstalowanego K3s — pełna reinstalacja wciąż zbyt powolna | Git nie pokrywa warstwy systemowej |
| Kopie zapasowe w chmurze / obrazy OVF | Pełne, przenośne | Czasochłonne, zajętość miejsca, złożone zarządzanie | Przesada dla laboratorium |
| **Snapshoty VirtualBox + Git jako warstwa kodu** | Szybkie przywracanie środowiska, Git dla kodu, spójne z iteracyjnym procesem | Zajętość dysku hosta rośnie z liczbą snapshotów (akceptowalne) | **Wybrano** — zbalansowany kompromis dla laboratorium |

---

## Konsekwencje

### Pozytywne

- Eksperymenty z konfiguracją (K3s, Ansible, sieć) prowadzone bez strachu o utratę stabilizego stanu.
- Test odtworzeniowy jest powtarzalny i tani — jeden z kluczowych dowodów poprawności automatyzacji.
- Jasny podział ról: Git = kod, snapshot = środowisko.

### Negatywne / Ryzyka

- Zajętość dysku hosta rośnie z liczbą snapshotów — należy/usuwać zbędne.
- Snapshot nie jest kopią zapasową w sensie przenośności, jest powiązany z VirtualBox na bieżącym hoście.
- W razie korupcji VM wszystkie snapshoty są bezużyteczne — ale Git zachowuje kod projektu.

### Wymagane działania

- [x] Przed każdym snapshotem: `git commit && git push`.
- [x] Po udanym teście odtworzeniowym potwierdzić, że playbook odtwarza klaster z czystego stanu snapshotu bazowego.
- [x] Przeglądać i usuwać zbędne migawki okresowo.
- [x] W produkcji (chmura) stosować równoważniki (np. AMI, Packer images) zamiast ręcznego zarządzania snapshotami.

---

## Powiązania

- [[lessons-learned|Lekcje Wyciągnięte]] sekcje 4 i 7.
- [[specyfikacja-maszyn|Specyfikacja Maszyn]] — lista wszystkich snapshotów.
- Wszystkie pliki w `środowisko/` w sekcji „Migawka (Snapshot)".