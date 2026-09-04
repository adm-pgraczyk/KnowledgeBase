## iSCSI + MPIO- Konfiguracja dla środowiska Hyper-V
# Wprowadzenie
Celem konfiguracji jest zapewnienie hostom Hyper-V wielu niezależnych ścieżek dostępu do współdzielonej pamięci masowej przy użyciu połączenia iSCSI
Takie rozwiązanie jest szczególnie istotne w środowiskach wykorzystujących **Hyper-V Failover Cluster**, gdzie wszystkie węzły muszą mieć dostęp do współdzielonego storage`u.

# Cel konfiguracji
- Zapewnienie redundancji połączenia
- Eliminacja pojedynczego punktu awarii
- Możliwość korzystania z wielu ścieżek iSCSI
- Możliwość wykorzystania mechanizmów load balancing MPIO



## Polecenie

```powershell
Get-NetAdapter
```

s

