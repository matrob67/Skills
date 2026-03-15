---
name: inpi-rne
description: "How to connect to and query the INPI RNE API (Registre National des Entreprises) for French company and business owner data. Use this skill whenever the user mentions INPI, RNE, Registre National des Entreprises, French company search, SIREN lookup, searching for a French business by name, finding company directors or owners in France, auto-entrepreneur lookup, or wants to query the French business registry. Also trigger when the user wants to find who owns a company in France, look up a person's businesses, search by company name or SIREN number, or retrieve official French company registration data including addresses, creation dates, and legal representatives."
---

# INPI RNE API (Registre National des Entreprises)

The INPI RNE is the official French national business registry. It provides a REST API to search for companies, retrieve company details, and access formality data (creation, modification, dissolution). Authentication is required via login/password to obtain a Bearer token.

## 1. Authentication

The API requires a login step to obtain a JWT token. The token is then used as a Bearer token for all subsequent requests.

**Production server:** `registre-national-entreprises.inpi.fr`
**Test/fallback server:** `registre-national-entreprises-pprod.inpi.fr`

The production server can sometimes be unreachable. Always try production first, then fall back to the pprod server if you get a timeout or connection error.

### Login

```
POST https://registre-national-entreprises.inpi.fr/api/sso/login
Content-Type: application/json

{
  "username": "your_email@example.com",
  "password": "your_password"
}
```

**Response (200):**
```json
{
  "token": "eyJhbGciOi...",
  "user": {
    "roles": ["ROLE_FO_USER"],
    "id": 111111,
    "email": "your_email@example.com",
    "firstname": "Prénom",
    "lastname": "Nom"
  }
}
```

Use the returned `token` value in all subsequent requests:
```
Authorization: Bearer <token>
```

### Implementation pattern with fallback

```javascript
const HOSTS = [
  'registre-national-entreprises.inpi.fr',
  'registre-national-entreprises-pprod.inpi.fr'
];

async function login(username, password) {
  for (const host of HOSTS) {
    try {
      const response = await fetch(`https://${host}/api/sso/login`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ username, password }),
        signal: AbortSignal.timeout(10000)
      });
      if (response.ok) {
        const data = await response.json();
        return { token: data.token, host }; // Remember which host worked
      }
    } catch { continue; }
  }
  throw new Error('Cannot connect to INPI API');
}
```

## 2. Search Companies

Search by company name, SIREN number, or with filters.

```
GET https://{host}/api/companies?companyName=Dupont&page=1&pageSize=20
Authorization: Bearer <token>
```

### Available filters

| Parameter | Type | Description |
|-----------|------|-------------|
| `companyName` | string | Company name (contains search) |
| `siren[]` | array | Exact SIREN number (9 digits). Can pass multiple. |
| `zipCodes[]` | string | Postal code or department (e.g., `69` for Rhône, `75001` for Paris 1er) |
| `activitySectors` | string | One of: `COMMERCIALE`, `ARTISANALE`, `LIBERALE_REGLEMENTEE`, `LIBERALE_NON_REGLEMENTEE`, `AGRICOLE_NON_ACTIF`, `ACTIF_AGRICOLE`, `ARTISANALE_REGLEMENTEE`, `AGENT_COMMERCIAL`, `GESTION_DE_BIENS`, `SANS_ACTIVITE` |
| `submitDateFrom` | date | Registration date lower bound (YYYY-MM-DD) |
| `submitDateTo` | date | Registration date upper bound (YYYY-MM-DD) |
| `page` | number | Page number (default 1) |
| `pageSize` | number | Results per page, 1-100 (default 20) |

### Examples

```
# Search by company name
GET /api/companies?companyName=Dupont

# Search by SIREN
GET /api/companies?siren[]=123456789

# Search in Lyon area
GET /api/companies?companyName=Martin&zipCodes[]=69

# Search by activity sector
GET /api/companies?activitySectors=COMMERCIALE&zipCodes[]=75
```

## 3. Get Company Detail

Retrieve full details for a specific company by its SIREN number.

```
GET https://{host}/api/companies/{siren}
Authorization: Bearer <token>
```

**Response:** A JSON object containing the full company data including formalities, addresses, directors, and activity details. The structure varies between personne physique (sole proprietor) and personne morale (legal entity).

### Key data paths in the response

**For Personne Physique (sole proprietor):**
```
formality.content.personnePhysique.identite.nom          → Last name
formality.content.personnePhysique.identite.prenoms       → First name(s)
formality.content.personnePhysique.identite.dateDeNaissance → Birth date
```

**For Personne Morale (legal entity):**
```
formality.content.personneMorale.identite.denomination    → Company name
formality.content.personneMorale.identite.formeJuridique  → Legal form (SARL, SAS, etc.)
```

**Address (both types):**
```
formality.content.etablissementPrincipal.adresse.numVoie      → Street number
formality.content.etablissementPrincipal.adresse.typeVoie     → Street type (RUE, AVENUE...)
formality.content.etablissementPrincipal.adresse.voie         → Street name
formality.content.etablissementPrincipal.adresse.codePostal   → Postal code
formality.content.etablissementPrincipal.adresse.commune      → City
```

**Other useful fields:**
```
formality.content.dateCreation                             → Creation date
formality.content.nature                                   → Formality type
siren                                                      → SIREN number
```

## 4. Differential Search (New/Updated Companies)

Get companies created or updated since a specific date.

```
GET https://{host}/api/companies/diff?from=2025-01-01&to=2025-01-31&pageSize=100
Authorization: Bearer <token>
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `from` | date | Start date, exclusive (YYYY-MM-DD). Default: yesterday |
| `to` | date | End date, inclusive (YYYY-MM-DD). Default: today |
| `searchAfter` | string | SIREN to paginate from (don't fill on first call) |
| `companyName` | string | Filter by name |
| `siren[]` | array | Filter by SIREN |

## 5. Historical API

Get a company's state at a specific past date.

```
GET https://{host}/api/companies/{siren}?date=2023-06-15
Authorization: Bearer <token>
```

## 6. Error Codes

| HTTP Code | Meaning |
|-----------|---------|
| 200 | Success |
| 401 | Invalid credentials or expired token |
| 404 | Company not found |
| 429 | Rate limit exceeded |
| 500 | Server error |

If you get a 401 on a data request, the token may have expired. Re-login to get a fresh token.

## 7. Important Legal Notes

- Data marked `diffusionCommerciale: false` must NOT be used for commercial prospecting
- Data marked `diffusionINSEE: "N"` can only be redistributed by INPI itself
- All usage must comply with RGPD and the INPI data license terms
- Opposition to data reuse for prospection must be respected (art. R. 123-320 Code de commerce)

## 8. Practical Tips

- The production server (`registre-national-entreprises.inpi.fr`) can be slow or unreachable at times. The pprod server is more reliable but may have slightly different data.
- Tokens expire after some time (typically a few hours). If requests start returning 401, re-authenticate.
- When searching by name, the API does a "contains" search, so searching for "Martin" will return companies with "Martin" anywhere in the name.
- For paginating through large result sets, increment the `page` parameter. The response includes total count information to know how many pages exist.
- Rate limiting exists but is generous for normal usage. Avoid hammering the API with hundreds of requests per second.
