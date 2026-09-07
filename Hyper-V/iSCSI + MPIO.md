# iSCSI + MPIO- Konfiguracja dla środowiska Hyper-V
## Wprowadzenie
Celem konfiguracji jest zapewnienie hostom Hyper-V wielu niezależnych ścieżek dostępu do współdzielonej pamięci masowej przy użyciu połączenia iSCSI
Takie rozwiązanie jest szczególnie istotne w środowiskach wykorzystujących **Hyper-V Failover Cluster**, gdzie wszystkie węzły muszą mieć dostęp do współdzielonego storage`u.

## Cel konfiguracji
- Zapewnienie redundancji połączenia
- Eliminacja pojedynczego punktu awarii
- Możliwość korzystania z wielu ścieżek iSCSI
- Możliwość wykorzystania mechanizmów load balancing MPIO

## [!UWAGA]
**MPIO zapewnia redundancję ścieżek I/O. Nie zastępuje redundancji sieci, kontrolerów storage, RAID ani systemu backupu**

# MPIO 
**Multipath I/O** pozwala systemowi Windows Server korzystać z wielu ścieżek prowadzących do tego samego urządzenia pamięci masowej

**Przykładowa architektura:**

```text
Hyper-V Host
     │
     ├──── Switch 1 ──── Storage Path 1
     │
     └──── Switch 2 ──── Storage Path 2
```
W zależności od konfiguracji MPIO może również rozdzielać operacje I/O pomiędzy dostępne ścieżki

**Przykładowa adresacja:**
| Host / urządzenie | iSCSI Path 1 | iSCSI Path 2 |
| ----------------- | ------------ | ------------ |
| Hyper-V 01        | `10.10.10.11`  | `10.10.20.11`  |
| Hyper-V 02        | `10.10.10.12`  | `10.10.20.12`  |
| Storage           | `10.10.10.100` | `10.10.20.100` |
| VLAN              | `110`          | `120`          |

## Wymagania
Przed rozpoczęciem konfiguracji należy upewnić się:
- Host posiada dedykowane interfejsy sieciowe iSCSI
- Istnieje komunikacja pomiędzy hostem a urządzeniem pamięci masowej
- Znane są adresy IP wszystkich target portal
- Wszystkie węzły klastra posiadają spójną konfigurację
- Dostępne są uprawnienia administratora

## Instalacja MPIO
MPIO można zainstalować za pomocą Server Manager lub Powershell
Instalacja
```powershell
Install-WindowsFeature -Name Multipath-IO
```
Stan funkcji 
```powershell
Get-WindowsFeature -Name Multipath-IO
```
Oczekiwany rezultat:
Install State : Installed

**Po zainstalowaniu Multipath I/O należy wykonać restart systemu Windows w celu sfinalizowania instalacji** 
# Włączenie obsługi iSCSI w MPIO
Po instalacji MPIO należy skonfigurować obsługę urządzeń iSCSI
Uruchom:
**mpiocpl**

Otworzy się: **MPIO Properties**

Przejdź do zakładki **Discover Multi-Paths**

Zaznacz: **Add support for iSCSI devices**

Zatwierdź konfigurację. **System ponownie wymaga ponownego uruchomienia**

## Konfiguracja iSCSI Initiator
Windows Server posiada wbudowany **Microsoft iSCSI Initiator.**

Uruchom: **iscsicpl**

Lub wyszukaj **Server Manager -> Tools -> iSCSI Initiator**

Sprawdzenie usługi 
```powershell
Get-Service -Name MSiSCSI
```
Jeśli usługa nie działa: 
```powershell
Start-Service -Name MSiSCSI
```
Ustaw automatyczny start usługi:
```powershell
Set-Service -Name MSiSCSI -StartupType Automatic
```

## Konfiguracja kart sieciowych
Hyper-V 01

NIC iSCSI 1
IP:   10.10.10.11
VLAN: 110

NIC iSCSI 2
IP:   10.10.20.11
VLAN: 120

Storage:

iSCSI Target 1
10.10.10.100

iSCSI Target 2
10.10.20.100

**Test połączenia**
```powershell
Test-NetConnection 10.10.10.100 -Port 3260
```
## [!UWAGA]

Przed rozpoczęciem konfiguracji warto potwierdzić podstawową łączność IP z każdym targetem. TCP/3260 jest domyslnym portem
## Dodanie portali iSCSI
Uruchom: **iscsicpl**

Przejdź do: **Discovery -> Discover Portal**

Dodaj pierwszy portal: 10.10.10.100

Następnie przejdź do: **Advanced Settings** 


Local Adapter -> Microsoft iSCSI Initiator

Initiator IP ->  NIC iSCSI 1

**Powiel krok dla pozostałych połączeń**

## Połączenie z iSCSI Target
Przejdź do zakładki: **Targets**

Po wykryciu targetu wybierz go i kliknij **Connect**

Zaznacz: **Enable multi-path**

Następnie wybierz: **Advanced**

Skonfiguruj pierwszą ścieżkę:
Local Adapter: **iSCSI NIC 1**
Initiator IP: **10.10.10.11**
Target Portal IP: **10.10.10.100**

Utwórz połączenie

## Dodanie kolejnej ścieżki
Ponownie wybierz: **Connect**

Dla tego samego targetu skonfiguruj drugą ścieżkę
Local Adapter: **iSCSI NIC 2**
Initiator IP: **10.10.20.11**
Target Portal IP: **10.10.20.100**

Docelowo:

```text
Hyper-V Host
     │
     ├──── 10.10.10.11 ──── 10.10.10.100
     │
     └──── 10.10.20.11 ──── 10.10.20.100
```

## Weryfikacja MPIO
Po ustanowieniu połączeń sprawdź czy MPIO wykrywa wiele ścieżek

Podstawowe połączenie:
```powershell
mpclaim -s -d
```
Dla konkretnego urządzenia: 

```powershell
mpclaim -s -d <DeviceID>
```
W prawidłowo skonfigurowanym środowisku powinny być widoczne conajmniej dwie ścieżki

**Przykład**

| Path ID           | State 
| ----------------- | ------------ 
| 00000001          | Active/Optimized  
| 00000002          |Active/Optimized  

## [!NOTE]
Dokładne nazwy i stany ścieżek zależą od używanego storage'u, DSM oraz konfiguracji MPIO.

## MPIO i PowerShell
Windows Server udostępnia moduł PowerShell do zarządzania MPIO
**Aktywne sesje iSCSI**
```powershell
Get-IscsiSession
```
**Połączenia w ramach sesji iSCSI**
```powershell
Get-IscsiConnection
```
**Sprawdzenie ustawień**
```powershell
Get-MPIOSetting
```

**Urządzenia obsługiwane przez MPIO**
```powershell
Get-MPIOAvailableHW
```

**Urządzenia iSCSI**
```powershell
Get-MPIOAvailableHW -BusType iSCSI
```

## Load Balancing MPIO
MPIO może wykorzystywać różne polityki określające sposób obsługi dostępnych ścieżek:

| Polityka                     | Opis 
| -----------------            | ------------ 
| Fail Over Only               | Jedna ścieżka aktywna, pozostałe pełnią funkcję zapasową 
| Round Robin                  | Operacje I/O rozdzielane są pomiędzy dostępne ścieżki
| Round Robin with Subset      | Round Robin z wykorzystaniem określonego podzbioru ścieżek  
| Least Queue Depth            | Wybierana jest ścieżka z najmniejszą kolejką  
| Weighted Paths               | Ścieżki posiadają różne wagi 

**Sprawdzenie aktualnej polityki load balancing**
```powershell
Get-MSDSMGlobalDefaultLoadBalancePolicy
```

**Przykładowe ustawienie Round Robin**
```powershell
Set-MSDSMGlobalDefaultLoadBalancePolicy -Policy RR
```
##[!WARNING]
Nie należy wybierać polityki wyłącznie na podstawie potencjalnej wydajności. W pierwszej kolejności należy sprawdzić zalecenia producenta macierzy oraz używanego DSM.


## Konfiguracja hostów Hyper-V
W środowisku ** Failover Cluster** konfiguracja powinno być wykonana na każdym węźle
Przykład: 
```text
Hyper-V 01
     │
     ├──── iSCSI Path1
     │
     └──── iSCSI Path2

Hyper-V 02
     │
     ├──── iSCSI Path1
     │
     └──── iSCSI Path2
```

**Każdy host powinien posiadać:**
- Odpowiednie interfejsy iSCSI
- Poprawną adresację IP
- dostęp do wymaganych targetów
- skonfigurowane MPIO
- odpowiednią liczbę ścieżek
- zgodną konfigurację storage

## [!IMPORTANT]
Nie należy zakładać, że poprawna konfiguracja jednego węzła oznacza poprawną konfigurację całego klastra.
