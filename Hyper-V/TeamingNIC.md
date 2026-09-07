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
                          |
Hyper-V Host ──── SET ────| 
                          │
                          └──── Switch 2
```

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

## [!NOTE]
Używaj **SET** jeśli korzystasz z hyper-V, Failover Cluster lub klastrów S2D (Storage Spaces Direct)

Używaj **LBFO** na serwerach fizycznych (Bare Metal) bez roli Hyper-V jeśli potrzebujesz redundancji sieciowej


```powershell
Get-NetAdapter
```

s
