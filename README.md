# f5-user-alert-via-smtp
# F5 BIG-IP — SNMP Trap omzetten naar E-mailnotificatie

> Handleiding voor het toevoegen van een e-mailactie aan een bestaande SNMP trap alert, gebaseerd op F5 Knowledge Article K3667.

---

## Inhoudsopgave

1. [Vereisten](#vereisten)
2. [Stap 1 — SMTP instellen](#stap-1--smtp-instellen)
3. [Stap 2 — Alert aanpassen in user_alert.conf](#stap-2--alert-aanpassen-in-user_alertconf)
4. [Stap 3 — alertd herstarten](#stap-3--alertd-herstarten)
5. [Voorbeeld: Licentie bandbreedte alert](#voorbeeld-licentie-bandbreedte-alert)
6. [Aandachtspunten](#aandachtspunten)
7. [Testen](#testen)

---

## Vereisten

- Toegang tot de BIG-IP via SSH (root of admin met bash-toegang)
- Een SMTP relay die e-mail van de BIG-IP accepteert
- De naam en OID van de te wijzigen alert (te vinden in `/etc/alertd/alert.conf`)

---

## Stap 1 — SMTP instellen

Configureer de SMTP-instellingen via de GUI:

**System › SMTP › Configuration**

Vul in:
- **SMTP Server**: IP-adres of hostname van je mailrelay
- **From Address**: het afzenderadres (bijv. `bigip@jouwbedrijf.nl`)
- **Port**: standaard `25` (of `587` voor STARTTLS)

Verifieer de werking met de volgende CLI-opdracht:

```bash
echo "Dit is een testmail" | mail -vs "BIG-IP SMTP Test" jouw@emailadres.nl
```

---

## Stap 2 — Alert aanpassen in user_alert.conf

> ⚠️ Bewerk **nooit** `/etc/alertd/alert.conf`. Dit bestand wordt door F5 beheerd en overschreven bij upgrades.

Werk uitsluitend in:

```
/config/user_alert.conf
```

### Syntaxis

Voeg een `email`-actie toe aan de bestaande alert. Meerdere acties worden gescheiden door een puntkomma (`;`):

```
alert <ALERTNAAM> "<patroon>" {
    snmptrap OID="<OID>";
    email toaddress="<ontvanger>"
          fromaddress="<afzender>"
          body="<berichttekst>"
}
```

---

## Stap 3 — alertd herstarten

Na het opslaan van de wijzigingen in `user_alert.conf`:

```bash
bigstart restart alertd
```

Controleer of de service correct is opgestart:

```bash
bigstart status alertd
```

---

## Voorbeeld: Licentie bandbreedte alert

### Originele alert (alleen SNMP trap)

```
alert BIGIP_TMM_TMMERR_LICENSE_LIMIT "Bandwidth utilization is %d Mbps, exceeded %d%% of Licensed %d Mbps" {
    snmptrap OID=".1.3.6.1.4.1.3375.2.4.0.300"
}
```

### Aangepaste alert (SNMP trap + e-mail)

```
alert BIGIP_TMM_TMMERR_LICENSE_LIMIT "Bandwidth utilization is %d Mbps, exceeded %d%% of Licensed %d Mbps" {
    snmptrap OID=".1.3.6.1.4.1.3375.2.4.0.300";
    email toaddress="netwerk@jouwbedrijf.nl"
          fromaddress="bigip@jouwbedrijf.nl"
          body="WAARSCHUWING: BIG-IP bandbreedte licentielimiet overschreden! Controleer het systeem."
}
```

---

## Aandachtspunten

| Punt | Toelichting |
|---|---|
| **Alertnaam** | Moet exact overeenkomen met de naam in `/etc/alertd/alert.conf` |
| **OID** | Ongewijzigd laten als je de SNMP trap wilt behouden |
| **`fromaddress`** | Sommige mailrelays accepteren alleen geldige/whitelisted afzenders |
| **Body** | Is statische tekst — de `%d`-waarden uit het originele bericht komen **niet** automatisch mee |
| **Upgrades** | `user_alert.conf` blijft bewaard bij BIG-IP upgrades; `alert.conf` niet |
| **Meerdere ontvangers** | Gebruik meerdere `email`-blokken achter elkaar, elk afgesloten met `;` |

---

## Testen

Simuleer een alert handmatig via de CLI om te controleren of de e-mail aankomt:

```bash
logger -p local0.notice "Bandwidth utilization is 950 Mbps, exceeded 95% of Licensed 1000 Mbps"
```

Controleer de alertd-logs bij problemen:

```bash
tail -f /var/log/alertd
```

---

## Referenties

- [F5 K3667 — Configuring alerts to send email notifications](https://support.f5.com/csp/article/K3667)
- [F5 K3727 — Configuring custom SNMP traps](https://support.f5.com/csp/article/K3727)
- [F5 K11127 — Overview of the user_alert.conf file](https://support.f5.com/csp/article/K11127)
