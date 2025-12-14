# OCP Guestbook ORGI – CI/CD till OpenShift

## Deltagare
- Ayuub Mohammadin

## Kort om projektet
Detta projekt består av minst två containers:
- **Frontend** (Nginx som serverar statiska filer och proxy:ar API-anrop)
- **Backend** (Go API)
- **PostgreSQL** används av backend för lagring.

Målet var att få en pipeline där:
`git push` → GitHub Actions bygger containers → pushar till GHCR → deployar till OpenShift med `oc apply`.

## Länk till applikationen (Route)
Frontend Route:
- http://guestbook-frontend-grupp5.apps.devops24.cloud2.se/

## Struktur i repot
- `openshift/`
  - `postgresdeploy.yaml`
  - `backend-deploy.yaml`
  - `frontend-deploy.yaml`
- `.github/workflows/`
  - workflow för **frontend** (build + push till GHCR)
  - workflow för **backend** (build + push till GHCR)
  - `deploy-openshift.yml` (loggar in och kör `oc apply -f openshift/`)

## Hur jag löste uppgiften (steg för steg)

### 1) Build container 1 (frontend) + publicera till GHCR
Jag utgick från lärarens exempel på GitHub Actions för att bygga och publicera en image till GHCR.
Jag skapade en workflow för frontend som:
- checkar ut repot
- bygger frontend-image
- taggar med `${{ github.sha }}` och `latest`
- loggar in på GHCR och pushar images

### 2) Build container 2 (backend) + publicera till GHCR
Jag gjorde samma sak för backend, men anpassade stegen till backend-mappen och backend-imagen:
- bygger backend-image
- taggar med `${{ github.sha }}` och `latest`
- loggar in på GHCR och pushar images

### 3) PostgreSQL i OpenShift
Backend behöver en databas, så jag deployade Postgres i OpenShift via YAML.
Jag började från en tidigare fungerande Postgres-YAML och anpassade den för vårt kluster.

Under arbetet stötte jag på att namespace hade ResourceQuota som krävde att alla pods måste ha requests/limits, som jag glömmer alltid bort... så jag lade till:
- `requests.cpu`, `requests.memory`
- `limits.cpu`, `limits.memory`

Jag stötte också på problem med Postgres-image och taggar (manifest/tag saknades), och löste det genom att byta till en korrekt tillgänglig image/tag som gick att pull:a i klustret.

### 4) Deploya till OpenShift via GitHub Actions (oc apply)
Jag skapade en separat deploy-workflow som kör på push till `main` och:
- loggar in i OpenShift med `oc login`
- byter till vårt namespace (`oc project grupp5`)
- kör `oc apply -f openshift/`

Jag upptäckte att tokens kan vara kortlivade (t.ex. ~24h) lärde jag från jonas när han skulle kicka martin som oc apply administör :P, så jag valde att logga in med **användarnamn + lösenord** via GitHub Secrets vid varje körning för att undvika att deploy slutar fungera när en token går ut.

För att vara robusta kontrollerar deploy-workflow också om `oc` finns i runnern, och installerar annars `oc` automatiskt.

## GitHub Secrets som krävs
För deploy-workflow använder jag dessa repo-secrets:
- `OCP_USER`
- `OCP_PASS`
- `OCP_SERVER` = `https://api.devops24.cloud2.se:6443`
- `OCP_NAMESPACE` = `grupp5`

## YAML-filer (OpenShift)
Alla resurser appliceras från mappen `openshift/`:
- `postgresdeploy.yaml` (PostgreSQL + secret + PVC + service)
- `backend-deploy.yaml` (backend deployment + service)
- `frontend-deploy.yaml` (frontend deployment + service + route vid behov)

Deploy sker med:

```bash
 oc apply -f openshift/
```

## Bonus 1 – Vilken YML är ändrad? Behöver vi `oc apply`-a alla? Om vi ändrat ConfigMap, behöver vi starta om Deployment?

- **Vilken YAML är ändrad / måste vi apply:a alla?**  
  Man kan köra `oc apply` bara på den fil man har ändrat. 
  Men det är onödigt då oc inte ändrar på resuresr du inte har ändrat på så lika bra köra apply på dir driect


## Bonus 2 – GHCR privata images + autentisering från OpenShift (imagePullSecret)

Jag gjorde våra GHCR-packages (frontend + backend) **privata** och konfigurerade OpenShift så att klustret kan hämta (pull) images från GHCR.

### Så löste jag det
1) Vi skapade en GitHub Personal Access Token (PAT) med minst rättigheten `read:packages`.
2) I OpenShift skapade vi en `docker-registry` secret mot GHCR:
   - server: `ghcr.io`
   - username: `macfatty`
   - password: GitHub PAT
3) Jag länkade pull-secreten till ServiceAccount `default` i vårt namespace, så att pods automatiskt kan använda den vid image pull.
4) Jag startade om deployments för att tvinga en ny pull från GHCR.

### Hur Jag verifierade att det fungerade
- Jag kontrollerade att `ghcr-pull` fanns under **Image pull secrets** för `default` ServiceAccount (dvs att pods i namespace använder den för att hämta images).
- När packages var privata och pull-secreten var länkad kunde pods starta och images kunde hämtas normalt.
- Vid felsökning såg jag att om pull-secreten saknas kan pods annars hamna i `ImagePullBackOff` med “unauthorized” i events.

### Varför Jag inte använde ServiceAccount-token för GitHub Actions deploy
Jag försökte skapa en egen ServiceAccount för deploy (t.ex. `github-deployer`) och ge den rollen `edit` med `oc adm policy ...`,
men jag saknade behörighet att skapa/lista RoleBindings i projektet (RBAC/admin-åtgärd).
Därför valde jag istället att logga in i GitHub Actions med **användarnamn + lösenord** vid varje körning och sedan köra `oc apply -f openshift/`.

