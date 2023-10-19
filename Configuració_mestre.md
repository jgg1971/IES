![Generalitat Valenciana - CEICE / IES Poeta Paco Mollá (Alicante)](http://julio.iespacomolla.es/Comunes/Cabecera_CEICE_IESPPM_Transparente.png)
# Servidor mestre amb BIND

## Hardware virtual
![Pràctica de DNS - Esquema de xarxa](http://julio.iespacomolla.es/Comunes/SERR/DNS_Lab_2.2_1.svg)

Per a aquesta configuració i totes les altres d'aquest tema, es treballarà en una xarxa virtualitzada. El **servidor** tindrà 2 interfícies de xarxa:
* La primera (normalment **enp0s3**) connectada com a "Adaptador puente", adreça IP 192.168.2.X, on X és 200 més el teu número de PC.
* La segona (normalment **enp0s8**) connectada com a "Red interna", adreça IP 10.0.0.1/24.

També tindrem una màquina **client**, amb adreça IP 10.0.0.100 (o una adreça DHCP concedida pel servidor, si ho prefereixes).

El servidor prestarà el **servei DNS** i funcionarà com a **encaminador**, perquè la màquina client puga accedir a Internet.


## Configuració d’un servidor mestre DNS
Una zona màster té les següents característiques:
* Aporta dades autoritàries de la zona o domini.
* El fitxer de zona s'allotja en el disc local del servidor.
* El servidor de la zona **_master_** respon a les sol·licituds de transferència de zona dels servidors de zones **_slave_**.

Qualsevol zona o domini pot tindre una o més zones _master_ i zero o més zones _slave._

Configurarem un servidor mestre per a nostre domini (en aquest exemple usarem "asir.iespacomolla.es"), i una vegada fet tindrem el següent:
* Un servidor **mestre**, i per tant, autoritari, per al domini asir.iespacomolla.es, incloent la seua zona inversa. S’aconseguix creant una zona tipus **master**.
* Un servidor **caché** per a la resta de dominis diferents a asir.iespacomolla.es. S’aconseguix gràcies a l’estament ```recursion yes;``` i a la zona '.' (zona arrel).
* Un servidor que aporta el servei de consultes recursives: el servidor acceptarà consultes i s’encarregarà d’aconseguir les respostes; al final li les entregarà als sol·licitants. S’aconseguix amb l’estament ```recursion yes;```.

![Pràctica de DNS - Esquema de xarxa](http://julio.iespacomolla.es/Comunes/SERR/DNS_archivos_conf.svg)

Sabem que només hi ha un fitxer de configuració per a BIND, que en Debian és ```/etc/bind/named.conf```. La forma actual de treballar amb els fitxers de configuració dels diferents serveis, és fragmentant-los en fitxers més xicotets i temàtics des d’algun punt de vista. En conseqüència, el fitxer named.conf quedarà només amb les directives ```include```.

```c
// This is the primary configuration file for the BIND DNS server named.
//
// Please read /usr/share/doc/bind9/README.Debian.gz for information on the 
// structure of BIND configuration files in Debian, *BEFORE* you customize 
// this configuration file.
//
// If you are just adding zones, please do that in /etc/bind/named.conf.local

include "/etc/bind/named.conf.acl";
include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.logging";
include "/etc/bind/named.conf.local";
include "/etc/bind/named.conf.default-zones";
```

És important recordar que per a tots aquests fitxers, l’usuari **bind** del grup **bind**, ha de tindre permís de lectura.

En primer lloc crearem una llista *acl* per a referir-nos a dos servidors DNS secundaris que tenim **hipotèticament** en la nostra xarxa (no, no cal crear els servidors). Aquesta llista, que limita els servidors als quals s'enviaria actualitzacions,  la crearem en `/etc/bind/named.conf.acl`.

```bash
acl dns-secundaris {
  10.0.0.2;
  10.0.0.3;
};
```

En les opcions que tenim dins del fitxer `/etc/bind/named.conf.options` establim les següents configuracions globals:

* Establirem la opció **_version_** al valor "No disponible".
* Especificarem que el servidor puga escoltar per la interfície 10.0.0.1.
* El servidor DNS aportarà servei de *caché* i permetrà les consultes recursives, però només als equips de les xarxes a les quals està connectat el servidor. 
* Les transferències de zones **es prohibiran a tots els _hosts_ a nivell global**, i serà des de cada zona, des d'on es permetran als equips que interesse.

```bash
options {
    directory "/var/cache/bind";

    version "No disponible";

    listen-on { 10.0.0.1; };

    recursion yes; 
    allow-recursion { localnets; }; 

    allow-transfer { none; };
};
```

Definirem ara la zona per al domini (p. ex. asir.iespacomolla.es) amb la clàusula **_zone_** que escriurem en el fitxer `/etc/bind/named.conf.local`. Les propietats d'aquesta zona són:

##### Zona mestra

* El fitxer de zona serà `/var/cache/bind/db.master.asir.iespacomolla.es`.
* S'atendran les peticions de transferència de la zona sol si provenen dels nostres servidors DNS secundaris.

##### Zona inversa

* El fitxer de zona serà `/var/cache/bind/db.master.0.0.10.IN-ADDR.ARPA.rev`.
* S'atendran les peticions de transferència de la zona sol si provenen dels nostres servidors DNS secundaris.

```bash
zone "asir.iespacomolla.es" in {
   type master;
   file "db.master.asir.iespacomolla.es";
   allow-transfer { dns-secundaris; };
};

zone "0.0.10.IN-ADDR.ARPA" in {
   type master;
   file "db.master.0.0.10.IN-ADDR.ARPA.rev";
   allow-transfer { dns-secundaris; };
};
```

Ara crearem els dos fitxers que contrindran les definicions dels Registres de Recursos (RR) de les zones directa i inversa que acabem de definir.

```bash
$TTL 2d
$ORIGIN asir.iespacomolla.es.

@         IN    SOA   dns1.asir.iespacomolla.es. migueltomas.informatica.iespacomolla.es. (
                      2016040800  ; se = serial number
                      12h         ; ref = refresh
                      15m         ; ret = refresh retry
                      3w          ; ex = expiry
                      2h          ; nx = nxdomain ttl
                      )
          IN    NS    dns1
          IN    NS    dns2
          IN    NS    dns3

dns1       IN    A         10.0.0.1
dns2       IN    A         10.0.0.2
dns3       IN    A         10.0.0.3
ftp        IN    A         10.0.0.10
mail       IN    A         10.0.0.15
correu     IN    CNAME     mail
```

Quant a la zona de resolució inversa podrem configurar-la de la següent forma:

```bash
$TTL 2d
$ORIGIN 0.0.10.IN-ADDR.ARPA.

@         IN    SOA   dns1.asir.iespacomolla.es. migueltomas.informatica.iespacomolla.es. (
                      2016040800  ; se = serial number
                      12h         ; ref = refresh
                      15m         ; ret = refresh retry
                      3w          ; ex = expiry
                      2h          ; nx = nxdomain ttl
                      )
          IN    NS    dns1.asir.iespacomolla.es.
          IN    NS    dns2.asir.iespacomolla.es.
          IN    NS    dns3.asir.iespacomolla.es.

1       IN    PTR         dns1.asir.iespacomolla.es.
2       IN    PTR         dns2.asir.iespacomolla.es.
3       IN    PTR         dns3.asir.iespacomolla.es.
10      IN    PTR         ftp.asir.iespacomolla.es.
15      IN    PTR         mail.asir.iespacomolla.es.
```

Com sempre per a que la configuració prenga efecte haurem de reiniciar el servei

```bash
systemctl restart named.service
```

I podrem comprovar el funcionament llançant una consulta amb *dig*

```bash
dig ftp.asir.iespacomolla.es
```

I la resolució inversa amb la instrucció

```bash
dig -x 10.0.0.15
```

## Adicional

1. Fes una comprovació amb un client de la configuració del servei DNS. Per a això hauràs de configurar el servidor DHCP per a que  done una configuració correcta dels paràmetres de xarxa als clients. 
2. Intenta resoldre des del client l'adreça www.google.es que ocorrix? Modifica la configuració global del servidor DNS per a que les consultes que no pugen ser resoltes s'envien a un servidor DNS com 1.1.1.1 o 8.8.8.8


## Referències
F. Periáñez Gómez (2020) [Configuración de un servidor DNS maestro](https://www.fpgenred.es/DNS/configuracin_de_un_servidor_dns_maestro.html). fpgenred.



## Colofó
Treball original:  
[![GPL](http://julio.iespacomolla.es/Comunes/Iconos/GPLv3_Logo.png)](https://www.gnu.org/licenses/gpl-3.0.html#license-text)  2017 F. Periáñez Gómez, [IES Mar de Cádiz](https://www.ies-mardecadiz.com/), Cadis (Espanya)

Traducció:  
[![CC-BY-SA](https://upload.wikimedia.org/wikipedia/commons/1/1c/Cc_by-sa_%281%29.svg)](https://creativecommons.org/licenses/by-nc-sa/4.0/deed.es) 2021 M. A. Tomás Amat, [IES Poeta Paco Mollà](https://iespacomolla.es/), Alacant (Espanya)

Adaptació:  
[![CC-BY-NC-SA](https://upload.wikimedia.org/wikipedia/commons/5/55/Cc_by-nc-sa_euro_icon.svg)](https://creativecommons.org/licenses/by-nc-sa/4.0/deed.es) 2023 J. Garay, [IES Poeta Paco Mollà](https://iespacomolla.es/), Alacant (Espanya)
