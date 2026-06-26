# Troubleshooting

## Problemy z infrastrukturą

### VM nie uruchamia się

**Objawy:** {{opis}}

**Diagnostyka:**
```bash
VBoxManage list runningvms
```

**Rozwiązanie:**
1. Sprawdź zasoby hosta (RAM/CPU)
2. Sprawdź konflikty sieciowe (adaptery)
3. Uruchom VM w trybie headless: `VBoxManage startvm {{nazwa}} --type headless`

---

### Brak połączenia z VM

**Objawy:** `ping 192.168.56.x` nie odpowiada

**Diagnostyka:**
```bash
ip a show
ping 192.168.56.1
```

**Rozwiązanie:**
1. Sprawdź, czy adapter Host-Only jest włączony w VirtualBox
2. Sprawdź reguły firewalla: `sudo ufw status`

---

## Problemy z Kubernetes

### Węzeł w stanie NotReady

**Objawy:** `kubectl get nodes` pokazuje `NotReady`

**Diagnostyka:**
```bash
kubectl describe node {{nazwa}}
sudo systemctl status kubelet
sudo journalctl -u kubelet -n 50
```

**Rozwiązanie:**
1. Restart kubelet: `sudo systemctl restart kubelet`
2. Sprawdź wolne zasoby: `free -h`, `df -h`

> [!NOTE] NotReady tuż po restarcie VM
>
> W środowisku K3s na VirtualBox, przez pierwsze kilkadziesiąt sekund po restarcie maszyny node może mieć status `NotReady`. Jest to normalne podczas startu komponentów Kubernetes (kubelet + CNI). Sprawdź:
> ```bash
> sudo systemctl status k3s
> sudo k3s kubectl get nodes
> ```
> Stan powinien przejść na `Ready` po uruchomieniu kubeleta i sieci CNI. Patrz [[lessons-learned|Lekcje Wyciągnięte]] sekcja 5.

---

### K3s serwer reklamuje adres NAT (10.0.2.15)

**Objawy:** Workery nie mogą dołączyć do klastra:
```text
dial tcp 10.0.2.15:6443: connect: connection refused
```
Instalacja agenta kończy się sukcesem, ale `kubectl get nodes` pokazuje tylko Control Plane (lub w ogóle żadnego workera).

**Diagnostyka:**
```bash
sudo k3s kubectl get nodes -o wide
# INTERNAL-IP pokazuje 10.0.2.15 zamiast 192.168.56.x
```

**Przyczyna:** K3s w środowisku VirtualBox (NAT + Host-Only) domyślnie wybiera adres interfejsu NAT.

**Rozwiązanie:** Zainstaluj K3s z wymuszonym adresem sieci Host-Only:
```bash
# Serwer:
curl -sfL https://get.k3s.io | \
  INSTALL_K3S_EXEC="server --node-ip=192.168.56.102 --advertise-address=192.168.56.102" sh -

# Agent:
curl -sfL https://get.k3s.io | \
  K3S_URL=https://192.168.56.102:6443 K3S_TOKEN=<TOKEN> \
  INSTALL_K3S_EXEC="agent --node-ip=192.168.56.103" sh -
```
Decyzja sformalizowana w [[ADR-001 — Wymuszenie adresu IP Host-Only dla K3s|ADR-001]]. Patrz też [[lessons-learned|Lekcje Wyciągnięte]] sekcja 1.

---

### Komunikat „Starting k3s-agent" trwa kilka minut

**Objawy:** Instalacja agenta pozornie zawiesza się na:
```text
[INFO] systemd: Starting k3s-agent
```

**Przyczyna:** Nie jest to błąd. W tle wykonywane jest: uruchomienie containerd, pobieranie obrazów kontenerów, rejestracja noda w Control Plane, konfiguracja CNI. Czas trwania zależy od przepustowości łącza (pobieranie obrazów) i może sięgać kilku minut.

**Rozwiązanie:** **Nie przerywać instalacji.** Weryfikować stan na serwerze:
```bash
sudo k3s kubectl get nodes
```
Patrz [[lessons-learned|Lekcje Wyciągnięte]] sekcja 3.

---

### Ekran logowania Ubuntu pokazuje tylko adres 10.0.2.15

**Objawy:** Po uruchomieniu VM Ubuntu ekran logowania MOTD wyświetla:
```text
IPv4 address for enp0s3: 10.0.2.15
```
Brak informacji o adresie Host-Only — prowadzi do błędnego wniosku, że sieć私有 nie działa.

**Przyczyna:** Komunikat pokazuje tylko pierwszy interfejs (`enp0s3` NAT). Adres `enp0s8` (Host-Only) jest pomijany w MOTD.

**Rozwiązanie:** Sprawdzić pełną konfigurację sieciową:
```bash
hostname -I
# lub
ip addr
```
Powinny być widoczne oba adresy: `10.0.2.15` oraz `192.168.56.x`. Patrz [[lessons-learned|Lekcje Wyciągnięte]] sekcja 2.

---

### Pody w CrashLoopBackOff

**Objawy:** Pody ciągle restartują się

**Diagnostyka:**
```bash
kubectl get pods -n {{namespace}}
kubectl describe pod {{nazwa}} -n {{namespace}}
kubectl logs {{nazwa}} -n {{namespace}} --previous
```

**Rozwiązanie:**
1. Sprawdź, czy obraz jest dostępny w registry
2. Sprawdź zmienne środowiskowe i konfigurację
3. Sprawdź limity zasobów (CPU/RAM)

---

## Problemy z CI/CD

### Pipeline nie uruchamia się

**Objawy:** Commit nie triggeruje pipeline'a

**Diagnostyka:**
- Sprawdź webhook w repo
- Sprawdź plik konfiguracyjny pipeline'a (składnia)

**Rozwiązanie:**
1. Sprawdź logi CI/CD engine
2. Ręcznie wyzwól pipeline
3. Sprawdź uprawnienia tokena/service account

---

### Build się nie udaje

**Objawy:** Pipeline failuje na etapie build

**Diagnostyka:**
```bash
# Lokalnie:
{{komenda build}}
```

**Rozwiązanie:**
1. Sprawdź logi builda
2. Sprawdź wersje narzędzi (JDK, Node, itp.)
3. Wyczyść cache i uruchom ponownie

---

## Problemy z monitoringiem

### Brak danych w Prometheusie

**Objawy:** Puste dashboardy w Grafanie

**Diagnostyka:**
```bash
kubectl port-forward -n monitoring svc/prometheus 9090:9090
# Otwórz http://localhost:9090/targets
```

**Rozwiązanie:**
1. Sprawdź status targetów w Prometheusie
2. Sprawdź ServiceMonitory i PodMonitory
3. Sprawdź RBAC

---
<!-- Dodawaj kolejne wpisy w miarę napotykania problemów -->
