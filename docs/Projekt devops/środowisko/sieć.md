# Konfiguracja Sieci

## Adapter 1 — NAT (dostęp do Internetu)

| Parametr | Wartość |
|---|---|
| Typ adaptera | NAT |
| Interfejs | `enp0s3` |
| Pula adresów | `10.0.2.0/24` |
| Brama domyślna | `10.0.2.2` |

## Adapter 2 — Host-Only (sieć wewnętrzna klastra)

| Parametr | Wartość |
|---|---|
| Typ adaptera | Host-Only Adapter |
| Interfejs | `enp0s8` |
| Podsieć | `192.168.56.0/24` |
| Adres hosta | `192.168.56.1` |
| DHCP | Włączony |
| Adres serwera DHCP | `192.168.56.100` |
| Dolna granica puli adresów | `192.168.56.101` |

## Adaptery sieciowe maszyn

| Maszyna | Adapter 1 (NAT) | Interfejs | Adapter 2 (Host-Only) | Interfejs |
|---|---|---|---|---|
| `k8s-control-01` | NAT | `enp0s3` | Host-Only Adapter | `enp0s8` |
| `k8s-node-01` | NAT | `enp0s3` | Host-Only Adapter | `enp0s8` |
| `k8s-node-02` | NAT | `enp0s3` | Host-Only Adapter | `enp0s8` |

> Każda maszyna otrzymuje dostęp do internetu przez NAT (Adapter 1, `enp0s3`) oraz komunikację wewnętrzną przez Host-Only (Adapter 2, `enp0s8`). Specyfikacja maszyn znajduje się w [[specyfikacja-maszyn|Specyfikacji Maszyn]].

## Statyczne Adresy IP (Host-Only)

Po etapie klonowania i regeneracji `machine-id` (zobacz [[przygotowanie-wezlow|Przygotowanie Węzłów Klastra Kubernetes]]) skonfigurowano statyczne adresy IP dla interfejsów `enp0s8`:

| Host           | Interfejs | Adres             |
| -------------- | --------- | ----------------- |
| k8s-control-01 | enp0s8    | 192.168.56.102/24 |
| k8s-node-01    | enp0s8    | 192.168.56.103/24 |
| k8s-node-02    | enp0s8    | 192.168.56.104/24 |

> Interfejsy NAT (`enp0s3`) pozostawiono z dynamicznym przydziałem adresów przez DHCP dla zapewnienia dostępu do Internetu.
