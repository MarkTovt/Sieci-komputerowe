## Wymagania

W sieci pracują komputery biurowe oraz urządzenia siecowe współdzielące zasoby. Do tej pory organizacja borykała się z ręczna konfiguracją urządzeń oraz adresami IP które dla ludzi z poza kadry technicznej były niezrozumiałe. Postanowiono:

* Wykorzystać usługę DHCP do nadawania adresów w sposób automatyczny dla wszystkich stacji roboczych
* Serwer oraz durządzenia IP tj: drukarka muszą posiadać stałe adresy celem zminimalizowanai potrzeby rekonfiguracji ustawiań klientów
* Wprowadzić translację pomiędzy Adresami IP oraz nazwami domenowymi dla kluczowych zasobów
   - erp.mojaorganizacja.pl
   - drukarka.mojaorganizacja.pl
   - router.mojaorganizacja.pl
* Wszystkie urządzenia łączą się z siecią internet z wykorzystaniem bramy NAT
* Wykorzystać podsieć rozmiaru /22 pozwalającej zaadresować co najmniej 600 urządzeń

## Rozwiązanie

Dla osiągnięcia celu było wykorzystane oprogramowanie `VirtualBox`.
Na początek przyjmijmy że nasza podsieć posiada adres `192.168.100.0` i maskę `255.255.252.0` co nam daje 1022 adresów IP do zaadresowania. 
Nasz router na interfejsie sieciowym `eth0` będzie się łączył z siecią Internet, a na interfejsie `eth1` - z naszą siecią firmową.
Wszystkie urządzenia firmowe działają na interfejsię `eth0`.

Kolejnym krokiem w konfiguracji naszej sieci będzie skonfigurowanie routera który będzie spełniać rolę bramy NAT oraz służyć serwerem DHCP. Dla wykonania powyższych zadań powinniśmy wpisać poniższy kod:
`apk add dhcp`   - instalujemy usługę DHCP na nasz router

`vi /etc/dhcp/dhcpd.conf`  - zaczynamy redagować plik konfiguracyjny usługi DHCP

``` 
    subnet 192.168.100.0 netmask 255.255.252.0 {
    range 192.168.100.1 192.168.103.254;
    option routers 192.168.100.1;
    option domain-name-servers 192.168.100.1,8.8.8.8,8.8.4.4,1.1.1.1;
    }
    
    host router{
    hardware ethernet 08:00:27:d1:5c:dc;
    fixed-address router.mojaorganizacja.pl;
    }
    
    host erp{
    hardware ethernet 08:00:27:cd:1e:57;
    fixed-address erp.mojaorganizacja.pl;
    }
    
    host drukarka{
    hardware ethernet 08:00:27:69:4c:f5;
    fixed-address drukarka.mojaorganizacja.pl;
    }
```

Zgodnie z zamieszczonym kodem nasz router, który teraz pełni rołę serwera DHCP, będzie automatycznie przypisywał adresy IP wszystkim urządzeniom podłączonym do niego.
Adresy IP będą z zakresu `192.168.100.1 - 192.168.103.254` z naszej podsieci `192.168.100.0` o masce `255.255.252.0`. Ten router jest tak samo domyślną bramą dla urządzeń firmowych.

Następnie rozwiązujemy problem łączenia się siecią Internet urządzeń naszej sieci firmowej. Zamieszczony poniżej kod rozwiązuje ten problem:
``` 
    sysctl net.ipv4.ip_forward=1
    apk add iptables
    iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```
Nadajemy możliwość naszemu PC'owi przesyłania pakietów , czyli wykonywania funkcji routera. Instalujemy iptables i nadajemy możliwość zmiany adresu prywatnego na publiczny w celu łaczenia naszych urządzeń firmowych z siecią Internet na interfejsie sieciowym `eth0`.


Ostatnim krokiem będzie edytowanie pliku pod nazwą hosts, w celu prypisania adresów IP urządzeniom sieci firmowej:

```
    apk add dnsmasq
    vi /etc/hosts/
       192.168.100.1 router.mojaorganizacja.pl
       192.168.100.2 erp.mojaorganizacja.pl
       192.168.100.3 drukarka.mojaorganizacja.pl
    service dnsmasq restart
```

Zgodnie z powyższą instrukcją jesteśmy w stanie skonfigurować nasz router, który będzie pełnił rołę serwera DHCP, DNS i bramy NAT.
