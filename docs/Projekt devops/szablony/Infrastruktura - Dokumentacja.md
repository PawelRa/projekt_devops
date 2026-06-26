# Infrastruktura — Dokumentacja

## Przegląd

Krótki opis infrastruktury, jej przeznaczenia i zakresu.

---

## Topologia

```
{{Diagram topologii — ASCII lub Mermaid}}
```

---

## Specyfikacja Maszyn

| Nazwa | CPU | RAM | Dysk | OS | Rola |
|---|---|---|---|---|---|

---

## Sieć

| Sieć | Podać adresy | Pula | DHCP | Typ |
|---|---|---|---|---|

| Maszyna | Adapter 1 | Adapter 2 |
|---|---|---|

---

## Kubernetes

| Komponent | Wersja | Maszyna | Opis |
|---|---|---|---|

### Namespace'y

| Namespace | Przeznaczenie |
|---|---|

### Storage

| StorageClass | Provisioner | Parametry |
|---|---|---|

---

## IaC (Infrastructure as Code)

| Narzędzie | Wersja | Opis użycia |
|---|---|---|

### Struktura katalogów IaC

```
terraform/
├── modules/
├── environments/
└── ...
```

---

## Backup i Disaster Recovery

- **Backup:** {{Opis harmonogramu i zakresu}}
- **RTO:** {{Recovery Time Objective}}
- **RPO:** {{Recovery Point Objective}}

---

## Dostęp

| Zasób | URL / IP | Port | Uwierzytelnianie |
|---|---|---|---|
