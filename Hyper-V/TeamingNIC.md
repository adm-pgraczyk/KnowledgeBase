# Hyper-V Switch Embedded Teaming (SET)
## Wprowadzenie
**Switch Embedded Teaming (SET)** to mechanizm wbudowany bezpośrednio w **Hyper-V Virtual Switch**, który pozwala połączyć kilka fizycznych kart sieciowych w celu zapewnienia redundancji oraz rozłożenia ruchu sieciowego

SET jest dostępny w Windows Server od wersji Windows Server 2016 i jest rozwiązaniem przeznaczonym dla środowisk Hyper-V

W przeciwieństwie do klasycznego **LBFO NIC Teaming** nie tworzymy w pierwszej kolejności zespołu kart sieciowych a następnie nie podpinamy go pod Hyper-V vSwitch. Zamiast tego fizyczne karty sieciowe przekazujemy bezpośrednio podczas tworzenia vSwitch przy użyciu **PowerShell**

## Cel Konfiguracji
Celem konfiguracji jest utworzenie redundantnego vSwitcha Hyper-V wykorzystującego dwie lub więcej fizycznych kart sieciowych

**Przykładowa architektura**
```text
                         ┌──── Switch 1
                         │
Hyper-V Host ─── SET ────┤
                         │
                         └──── Switch 2
```
## [!NOTE]
Switch 1 i Switch 2 powinny być niezależnymi ścieżkami sieciowymi

**Przykładowe interfejsy:**
| Element               | Wartość
| -----------------     | ------------ 
| vSwitch               | vSwitch-SET
| NIC 1                 | Ethernet 1
| NIC 2                 | Ethernet 2
| Teaming               | SET 
| Teaming Mode          | Switch Independent
| Load Balancing        | Dynamic 

## [!NOTE]
Nazwy interfejsów Ethernet 1 i Ethernet 2 są przykładowe. Przed konfiguracją należy sprawdzić rzeczywiste nazwy kart za pomocą Get-NetAdapter.

## SET vs klasyczny NIC Teaming
W starszych konfiguracjach Hyper-V można było spotkać rozwiązanie oparte o LBFO (Load Balancing/Failover)

W przypadku współczesnych środowisk Hyper-V preferowanym rozwiązaniem jest SET

| LBFO              | SET
| -----------------     | ------------ 
| Osobny NIC Team               | Teaming wbudowany w Hyper-V vSwitch
| Team tworzony przed vSwitchtem                 | Team tworzony razem z vSwitchtem
| Klasyczne rozwiązanie                 | Rozwiązanie dedykowane dla Hyper-V
| Działa na poziomie systemu operacyjnego | Działa wewnętrz przełącznika wirtualnego Hyper-V
| Obsługuje tryby niezależne oraz zależne od przełącznika (np. LACP) | Wyłącznie tryb niezależny od przełącznika
| Od Windows Server 2022 nie można połączyć przełącznika Hyper-V z zespołem LBFO | Rozwiązanie dedykowane

## Gdzie stosować SET?
Switch Embedded Teaming (SET) jest rozwiązaniem przeznaczonym dla Hyper-V. Łączy fizyczne karty sieciowe bezpośrednio z Hyper-V Virtual Switch, zapewniając redundancję oraz możliwość wykorzystania wielu interfejsów sieciowych. 

Sprawdza się szczególnie w środowiskach:
- Hyper-V
- Failover Cluster
- Storage Spaces Direct (S2D)

W przypadku klastra warto rozdzielić logicznie poszczególne rodzaje ruchu np:

VLAN 10 -> Management
VLAN 11 -> Cluster
VLAN 12 -> Live Migration

Przykład: 
```text
              SET / vSwitch
                    │
       ┌────────────┼────────────┐
       │            │            │
  Management     Cluster     Live Migration
   VLAN 10       VLAN 11        VLAN 12
       │            │            │
       └────────────┼────────────┘
                    │
                VM Network
```

## [!NOTE]
**SET** nie zapewnia sam z siebie izolacji Management, Cluster, Live Migration czy Storage. Zapewnia warstwę redundantnych uplinków dla vSwitcha, natomiast separację ruchu realizuje się za pomocą VLAN-ów
**SET** działa w trybie Switch Independent i nie wymaga konfiguracji LACP ani agregacji portów na przełączniku fizycznym. Każdy fizyczny interfejs jest podłączony niezależnie do switcha, a mechanizmem redundancji i rozkładania ruchu zajmuje się Hyper-V Virtual Switch



Używaj **LBFO** na serwerach fizycznych (Bare Metal) bez roli Hyper-V jeśli potrzebujesz redundancji sieciowej

## Konfiguracja

**Sprawdzenie dostępnych kart sieciowych**
```powershell
Get-NetAdapter
```
**Utworzenie vSwitch**
```powershell
New-VMSwitch -Name "vSwitch" -NetAdapterName "NIC 1", "NIC 2" -EnableEmbeddedTeaming $true
```
Polecenie tworzy zewnętrzny Hyper-V vSwitch wykorzystujący dwie fizyczne karty sieciowe

**Ustawienie algorytmu LoadBalancing**
```powershell
Set-VMSwitchTeam -Name "vSwitch" -LoadBalancingAlgorithm Dynamic
```
## [!NOTE]

W przypadku Teamingu SET, dostępne są obecnie dwa algorytmy: HyperVPort i Dynamic. Domyślną wartością jest Dynamic

* **HyperVPort**- Przypisuje ruch do fizycznego NIC na podstawie adresu MAC maszyny wirtualnej, ścieżki są bardziej statyczne. Microsoft opisuje HyperVPort jako dystrybucję opartą o MAC wirtualnych adapterów

Przykład: 
```text
VM01 ──────► NIC 1
VM02 ──────► NIC 2
VM03 ──────► NIC 1
VM04 ──────► NIC 2
```

Zaletą jest przewidywalność ruchu sieciowego. Wadą może być to, że jeśli jedna VM generuje duży ruch a pozostałe maszyny nie wykazują zbyt dużej utylizacji łącza, jedna z kart może byc mocno obciążona podczas gdy druga będzie miała dużo wolnego pasma

* **Dynamic**- Dla ruchu sieciowego dobiera ścieżki dynamicznie. Rozkłada ruch pomiędzy dostępne fizyczne NIC. Przy środowisku gdzie jest dużo maszyn wirtualnych i są różne poziomy ruchu, Dynamic jest sensownym rozwiązaniem

Przy **HyperVPort** możesz mieć sytuację:

NIC 1 -> VM01 + VM03 = 15Gb/s

NIC 2 -> VM02 +VM04 = 3Gb/s

## Test redundancji
Samo utworzenie SET nie oznacza, że konfiguracja została prawidłowo przetestowana.

Należy sprawdzić zachowanie podczas awarii jednej z kart

**Przykładowy scenariusz:**

1. Sprawdź stan interfejsów
2. Uruchom VM korzystającą z vSwitch
3. Wyłącz jeden z interfejsów sieciowych
4. Zweryfikuj dostępność VM
5. Przywróć interfejs sieciowy
6. Wyłącz drugi z interfejsów
7. Ponownie zweryfikuj dostępność VM
8. Przywróć interfejs sieciowy
9. Sprawdź stan zespołu

**Przykładowe polecenia:**
Wyłączenie karty sieciowej
```powershell
Disable-NetAdapter -Name "NIC 1" -Confirm:$false
```
Właczenie karty sieciowej
```powershell
Enable-NetAdapter -Name "NIC 1" -Confirm:$false
```

## Dodatkowe polecenia powershell
**Wyświetlenie konfiguracji**
```powershell
Get-VMSwitchTeam -Name "vSwitch" | FL
```

**Dodanie kolejnej karty sieciowej**
Dodanie karty sieciowej odbywa się poprzez przypisanie obecnych + dodatkowej karty sieciowej. Pominięcie kart, które aktualnie są dodane do konfiguracji spowoduje usunięcie ich z vSwitch
```powershell
Set-VMSwitchTeam -Name "vSwitch" -NetAdapterName "NIC 1","NIC 2","NIC 3"
```

**Usunięcie karty sieciowej**
```powershell
Set-VMSwitchTeam -Name "vSwitch" -NetAdapterName "NIC 1","NIC 2"
```

**Usunięcie vSwtich**
```powershell
Remove-VMSwitch -Name "vSwitch"
```

**Sprawdzenie właściwości fizycznych NIC**
```powershell
Get-NetAdapterAdvancedProperty -Name "NIC 1","NIC 2"
```

**Sprawdzenie VM podłączonych do vSwtich**
```powershell
Get-VMNetworkAdapter -All | Format-Table VMName, Name, SwitchName, Status
```
