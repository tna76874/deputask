# Anmeldung über OpenID Connect (Container)

Ohne weitere Einstellung hat der DepuTask-Server **keine Anmeldung**: Jede Person, die ihn im Netz erreicht,
kann alles sehen und ändern. Deshalb lauscht die mitgelieferte `compose.yaml` nur auf `127.0.0.1`. Bevor
DepuTask im Schulnetz erreichbar ist, die Anmeldung über den Identity Provider (IdP) der Schule einschalten,
z. B. Keycloak, Authentik, Microsoft Entra ID, Google Workspace, IServ/Moodle mit OIDC oder Dex.

Die Desktop-App hat nie eine Anmeldung, weil dort nur eine Person arbeitet.

## 1. Client beim Identity Provider anlegen

| Einstellung | Wert |
|---|---|
| Client-Typ | *confidential* (mit Client-Secret) |
| Ablauf | Authorization Code (PKCE wird von DepuTask immer verwendet) |
| Redirect-URI | `https://<adresse-von-deputask>/api/auth/callback` |
| Scopes | `openid profile email` |
| Post-Logout-Redirect (optional) | `https://<adresse-von-deputask>/` |

Der IdP muss die E-Mail-Adresse im ID-Token liefern (Claim `email`, möglichst mit `email_verified`).

## 2. `./data/config.json` anlegen

Die Datei liegt im eingebundenen Datenverzeichnis (neben `deputask.db`). Der Container liest sie beim Start
(`-config /data/config.json`; ein anderer Pfad lässt sich mit `DEPUTASK_CONFIG` angeben).

```json
{
  "auth": {
    "enabled": true,
    "issuer": "https://idp.schule.example/realms/schule",
    "clientId": "deputask",
    "redirectUrl": "https://deputask.schule.example/api/auth/callback",
    "scopes": ["profile", "email"],
    "admins": ["schulleitung@schule.example"],
    "autoCreate": false,
    "autoRedirect": true,
    "prompt": "login",
    "defaultRole": "lesen",
    "sessionHours": 12,
    "allowInsecureHttp": false
  }
}
```

Unbekannte oder falsch geschriebene Felder werden abgelehnt: Der Container startet dann nicht und nennt
das fehlerhafte Feld im Log (`docker compose logs deputask`). Der Abschnitt `auth` wird immer als Ganzes
übernommen.

| Feld | Pflicht | Bedeutung |
|---|:-:|---|
| `enabled` | ✓ | `true` schaltet die Anmeldung ein. Danach braucht jede Anfrage eine Anmeldung, außer Startseite, Versions- und Lizenztexten. |
| `issuer` | ✓ | Adresse des IdP. DepuTask liest `<issuer>/.well-known/openid-configuration`. |
| `clientId` | ✓ | Client-ID aus Schritt 1 |
| `clientSecret` | | Client-Secret. Besser über die Umgebungsvariable `DEPUTASK_OIDC_CLIENT_SECRET` übergeben (siehe Schritt 3). |
| `redirectUrl` | ✓ | Öffentliche Adresse von DepuTask mit `/api/auth/callback`, genau wie beim IdP eingetragen |
| `scopes` | | Zusätzliche Scopes neben `openid`. Standard: `profile`, `email` |
| `admins` | ✓ ¹ | E-Mail-Adressen, die immer die Rolle *Administration* erhalten. So kann sich die erste Administration anmelden, und niemand kann sich aussperren. |
| `autoCreate` | | `false` (empfohlen): Anmelden dürfen nur Personen, die in der Benutzerverwaltung angelegt sind, und `admins`. `true`: Jede Person, die der IdP anmeldet, wird mit `defaultRole` übernommen. |
| `autoRedirect` | | `true`: Wer nicht angemeldet ist, kommt ohne Zwischenseite direkt zum Anmeldedienst. Die Anmeldeseite von DepuTask erscheint dann nur noch nach einem Fehler, nach dem Abmelden oder wenn der Anmeldedienst ohne Anmeldung zurückleitet, damit keine Endlosschleife entsteht. `false` (Standard): Anmeldeseite mit Knopf „Anmelden“. |
| `prompt` | | Wird bei jeder Anmeldung an den Anmeldedienst geschickt. `login` (Standard): Der Anmeldedienst fragt **immer erneut** nach den Zugangsdaten, auch wenn dort noch eine Sitzung besteht – wichtig an gemeinsam genutzten Rechnern. Weitere Werte: `select_account`, `consent`, mehrere mit Leerzeichen. `auto`: kein Wert, der Anmeldedienst entscheidet (eine laufende Sitzung meldet ohne Rückfrage an). |
| `defaultRole` | | Rolle neuer Benutzer bei `autoCreate`: `admin`, `planung`, `sekretariat` oder `lesen` (Standard `lesen`) |
| `sessionHours` | | Wie lange eine Anmeldung gilt, in Stunden (Standard 12) |
| `allowInsecureHttp` | | `issuer` und `redirectUrl` müssen `https` sein. `true` erlaubt `http`, nur für Tests oder interne Netze hinter einem TLS-Proxy. `127.0.0.1` ist immer erlaubt. |

¹ Nur Pflicht, wenn `autoCreate` `false` ist.

## 3. Client-Secret und Neustart

In `compose.yaml` das Secret als Umgebungsvariable übergeben, am besten über eine Datei `.env` neben
`compose.yaml`, die nicht weitergegeben wird:

```yaml
services:
  deputask:
    environment:
      DEPUTASK_OIDC_CLIENT_SECRET: ${DEPUTASK_OIDC_CLIENT_SECRET}
```

```bash
echo 'DEPUTASK_OIDC_CLIENT_SECRET=geheim-aus-dem-idp' >> .env
docker compose up -d          # Container neu starten, damit config.json gelesen wird
```

Ohne Compose: `docker run … -e DEPUTASK_OIDC_CLIENT_SECRET=… ghcr.io/tna76874/deputask:latest`.

## 4. HTTPS über einen Reverse-Proxy

DepuTask selbst spricht nur HTTP auf Port 8080. Davor gehört ein Reverse-Proxy mit TLS-Zertifikat, z. B.
Caddy, Traefik oder nginx. Er leitet `https://deputask.schule.example` an `127.0.0.1:8080` weiter und
übergibt den Host-Namen (`Host` bzw. `X-Forwarded-Host`). Der Port in `compose.yaml` bleibt auf
`127.0.0.1` gebunden, damit niemand den Proxy umgehen kann.

Beispiel Caddy:

```
deputask.schule.example {
    reverse_proxy 127.0.0.1:8080
}
```

## 5. Erste Anmeldung und Rollen

1. `https://deputask.schule.example` öffnen und mit einer Adresse aus `admins` anmelden.
2. Unter **Lizenz** die Lizenzdatei einspielen. Ohne gültige Lizenz und ohne Daten lässt sich nichts
   anderes öffnen.
3. Unter **Benutzer** die weiteren Personen mit E-Mail-Adresse und Rolle anlegen. Bei ihrer ersten
   Anmeldung wird das Konto zugeordnet.

| Rolle | Stammdaten lesen | Stammdaten ändern / importieren | Planung lesen | Planung ändern | Administration |
|---|:-:|:-:|:-:|:-:|:-:|
| Administration (`admin`) | ✓ | ✓ | ✓ | ✓ | ✓ |
| Planung (`planung`) | ✓ | ✓ | ✓ | ✓ | |
| Sekretariat (`sekretariat`) | ✓ | ✓ | | | |
| Lesen (`lesen`) | ✓ | | ✓ | | |
| Lehrkraft (`lehrkraft`) | | | | | |

**Lehrkräfte und Deputatswünsche:** Mit Anmeldung wird jede aktive Lehrkraft, bei der in den Stammdaten
eine E-Mail-Adresse steht, automatisch Benutzer mit der Rolle *Lehrkraft* (oder einem schon vorhandenen
Benutzer mit dieser Adresse zugeordnet). Lehrkräfte sehen nur ihre eigenen Deputatswünsche. Wer beim Fach
als Fachschaftsleitung eingetragen ist, sieht zusätzlich das Wunschraster des Fachs und reicht es bei der
Schulleitung ein. Lehrkräfte ohne E-Mail-Adresse können sich nicht anmelden.

*Administration* umfasst Benutzerverwaltung, Lizenz, Aktualisierungen, Einstellungen, Beispieldaten und
„Alles löschen“. Rolle ändern und Sperren wirken sofort. Die letzte Administration und die Adressen aus
`admins` lassen sich nicht aussperren.

## Fehlersuche

| Meldung / Verhalten | Ursache |
|---|---|
| Container startet nicht, Log nennt `auth.…` | Feld fehlt, ist falsch geschrieben oder keine `https`-Adresse |
| `auth.redirectUrl muss auf /api/auth/callback enden` | `redirectUrl` unvollständig |
| `auth.admins ist leer` | Ohne `autoCreate` muss mindestens eine Adresse in `admins` stehen |
| IdP meldet „invalid redirect_uri“ | Redirect-URI beim IdP und `redirectUrl` stimmen nicht genau überein |
| „Anfrage von fremder Herkunft abgelehnt“ | Der Proxy gibt den Host-Namen nicht weiter (`Host`/`X-Forwarded-Host`) |
| Mit `autoRedirect` erscheint trotzdem die Anmeldeseite | Gewollt nach Abmelden, Fehlern oder einer zweiten Weiterleitung innerhalb einer Minute. „Anmelden“ führt weiter. |
| Anmeldung klappt, aber „nicht freigeschaltet“ | Die Person ist nicht angelegt und `autoCreate` ist `false` |

Ausschalten: in `config.json` `"enabled": false` setzen oder den Abschnitt `auth` entfernen, dann den
Container neu starten.
