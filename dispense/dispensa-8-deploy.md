# Dispensa 8 – Deploy end-to-end

*Corso: Infrastruttura AWS con Terraform*

## Obiettivo

Abbiamo tutti i pezzi: la rete, i firewall, il server, Tailscale, il database, i magazzini ECR e S3. In questa dispensa li colleghiamo. Alla fine, una modifica al codice dell'applicazione arriva in produzione così: **`git push`** → GitHub costruisce l'immagine → la carica su ECR → **`terraform apply`** con la nuova versione → il server viene **sostituito** da uno nuovo che parte già con la versione giusta.

Alla fine di questa dispensa:

- sai far caricare le immagini a **GitHub Actions senza nessuna chiave salvata**, con **OIDC**;
- sai trasformare il server in **"bestiame"**: un **launch template** e un **Auto Scaling group** che lo ricreano da soli se si rompe;
- sai dare al server un **nome stabile** (`app.corso.internal`) che resta lo stesso anche quando il server cambia;
- sai scrivere uno **script di avvio** che installa Docker, scarica l'immagine e i file, legge la password e avvia l'applicazione;
- sai gestire la **rotazione della password** senza fermare l'applicazione;
- sai rilasciare una **nuova versione**, **tornare indietro**, e far sopravvivere i dati a una **ricostruzione** dell'infrastruttura.

---

## Prima di iniziare

- Si lavora nella cartella `infra/` di sempre. In più serve un **secondo repository**, su GitHub, per il codice dell'applicazione: lo chiamiamo `corso-app`. Serve quindi un account GitHub: se hai scelto il "modo 2" della dispensa 5 ce l'hai già.
- Servono tutte le dispense precedenti, dalla 2 alla 7: rete, Security Group, role del server, router Tailscale, database, ECR e bucket degli artefatti.
- Rinnova il login: `aws sso login --profile corso`, `export AWS_PROFILE=corso`, e `export TAILSCALE_API_KEY=…` (dispensa 5).
- Docker sul tuo computer **non** serve più: l'immagine la costruisce GitHub.
- **Costi:** gli stessi delle dispense precedenti (server, router, NAT, database); in più la zona DNS privata, che si paga al mese. GitHub Actions è gratuito entro i minuti mensili inclusi. A fine esercizio si cancella tutto.

---

## 1. Il percorso completo

```mermaid
sequenceDiagram
    participant DEV as Tu
    participant GH as GitHub Actions
    participant AWS as AWS (IAM)
    participant ECR as ECR
    participant TF as terraform apply
    participant ASG as Auto Scaling group
    participant SRV as Server nuovo
    DEV->>GH: git push sul branch main
    GH->>AWS: "sono il workflow di corso-app, branch main" (token OIDC firmato)
    AWS-->>GH: credenziali temporanee, solo per caricare immagini
    GH->>ECR: immagine app:<commit>
    DEV->>TF: image_tag = "<commit>"
    TF->>ASG: nuova versione del launch template
    ASG->>SRV: crea il server nuovo, poi toglie il vecchio
    SRV->>SRV: script di avvio: Docker, immagine, password, app
    Note over DEV,SRV: nessuna chiave salvata da nessuna parte
```

Due metà:

1. **Costruire e caricare** (sezioni 2 e 3): lo fa GitHub, a ogni `git push`.
2. **Mettere in produzione** (sezioni 4-9): lo fai tu, cambiando una variabile e lanciando `terraform apply`.

Le teniamo separate di proposito: GitHub può solo **caricare** immagini, non può toccare l'infrastruttura. Decidere **quale** versione va in produzione resta una scelta tua, scritta nel tfvars e visibile nel `plan`.

---

## 2. GitHub Actions senza chiavi: OIDC

**GitHub Actions** è il servizio di GitHub che esegue dei comandi (un **workflow**) quando succede qualcosa nel repository, per esempio un `git push`. Il nostro workflow costruisce l'immagine e la carica su ECR. Per caricarla, però, deve **entrare in AWS**.

Il modo sbagliato, e ancora molto diffuso: creare in AWS una chiave d'accesso fissa e salvarla nei "segreti" di GitHub. È una chiave che **non scade**, che vale da qualunque posto, e che chiunque la trovi (in un log, in un fork, in un repository violato) può usare per sempre. È esattamente ciò che il corso evita dalla dispensa 0.

Il modo giusto si chiama **OIDC** (*OpenID Connect*): GitHub e AWS si fidano l'uno dell'altro, e **nessuna chiave viene salvata**.

> **Analogia.** Alla reception di un'azienda non tieni una copia delle chiavi di tutti i fornitori. Il fornitore arriva con un **documento d'identità rilasciato da un ente di cui ti fidi**; la reception controlla il documento e **cosa c'è scritto sopra** ("ditta X, tecnico della manutenzione"), e gli dà un **badge visitatore** che vale un'ora e apre solo la sala macchine. Domani il badge non vale più.

Come funziona:

1. Quando il workflow parte, GitHub gli dà un **token** firmato da GitHub: un documento che dice "sono il workflow del repository `mario/corso-app`, sul branch `main`".
2. Il workflow presenta il token ad AWS e chiede di **assumere un role** (dispensa 1).
3. AWS controlla la **firma** (il token viene davvero da GitHub?) e le **condizioni** scritte nella trust policy del role (è proprio quel repository? quel branch?).
4. Se tutto torna, AWS dà al workflow **credenziali temporanee** (un'ora), con i soli permessi del role: caricare immagini nel nostro repository ECR.

```mermaid
sequenceDiagram
    participant WF as Workflow su GitHub
    participant GH as GitHub (emette i token)
    participant STS as AWS (verifica e rilascia)
    WF->>GH: dammi un token
    GH-->>WF: token firmato: repo mario/corso-app, ref refs/heads/main
    WF->>STS: AssumeRoleWithWebIdentity(role, token)
    Note over STS: firma di GitHub? ✓<br/>aud = sts.amazonaws.com? ✓<br/>sub = repo:mario/corso-app:ref:refs/heads/main? ✓
    STS-->>WF: credenziali temporanee (1 ora)
    WF->>WF: docker push su ECR
```

Servono due cose in AWS:

| Cosa | Cos'è |
|---|---|
| **OIDC provider** | dice ad AWS: "i token firmati da GitHub (`token.actions.githubusercontent.com`) sono documenti validi". Uno per account |
| **Role** `github-release` | la **trust policy** accetta solo i token di **quel repository** e di **quel branch**; la **permission policy** permette solo di caricare immagini nel nostro repository ECR |

La parte più importante è la **condizione sul `sub`** (*subject*, "di chi parla il token"): `repo:mario/corso-app:ref:refs/heads/main`. Se fosse più larga, per esempio `repo:mario/*`, **qualunque** repository dell'utente potrebbe caricare immagini in produzione; se mancasse del tutto, **qualunque repository di GitHub al mondo**. È l'errore più grave che si possa fare con OIDC: va sempre ristretta al repository e al branch.

---

## 3. Il workflow: costruire e caricare

Il workflow è un file YAML nel repository dell'applicazione, in `.github/workflows/release.yml`. Fa sei passi:

| Passo | Cosa fa |
|---|---|
| `actions/checkout` | scarica il codice del repository |
| `aws-actions/configure-aws-credentials` | presenta il token OIDC e assume il role (sezione 2) |
| `aws-actions/amazon-ecr-login` | fa il login al registry ECR, con le credenziali appena ottenute |
| `docker/setup-qemu-action` | permette di costruire immagini **ARM** su un computer di GitHub che è **x86** |
| `docker/setup-buildx-action` | prepara lo strumento di costruzione di Docker |
| `docker/build-push-action` | costruisce l'immagine `linux/arm64` e la carica con il tag **uguale al codice del commit** |

Il tag è il **codice del commit** (`github.sha`, per esempio `3f9a2c1…`): ogni versione dell'immagine corrisponde esattamente a un punto della storia del codice. Con i tag immutabili della dispensa 7, "in produzione gira `3f9a2c1`" vuol dire una cosa sola, e si può risalire al codice con `git show 3f9a2c1`.

Due variabili del repository GitHub (non segreti: non c'è niente di segreto) dicono al workflow **quale** role assumere e **dove** caricare: `AWS_RELEASE_ROLE_ARN` e `ECR_REPOSITORY_URL`. Si impostano nelle impostazioni del repository (esercizio, passo 5).

---

## 4. Il server diventa "bestiame"

Nella dispensa 4 il server era una risorsa `aws_instance`: uno, con il suo nome, creato a mano da Terraform. Se si rompe, qualcuno se ne deve accorgere e ricrearlo.

> **Analogia.** Nel mondo dei server si dice: **animali domestici** contro **bestiame**. L'animale domestico ha un nome, lo curi quando sta male, se muore è un dramma. Il bestiame è numerato: se un capo si ammala, lo sostituisci con un altro identico. I server moderni sono bestiame: **nessun server è insostituibile**.

Perché un server sia sostituibile, sul server **non deve esserci niente di unico**:

| Cosa | Dove sta | Se il server sparisce |
|---|---|---|
| i dati | **RDS** (dispensa 6) | restano |
| il programma | **ECR** (dispensa 7) | si riscarica |
| i file di deploy | **S3** (dispensa 7) | si riscaricano |
| la password | **Secrets Manager** (dispensa 6) | si rilegge |
| la configurazione del server | lo **script di avvio** (sezione 6), scritto nel codice Terraform | si riesegue |

Due risorse nuove prendono il posto di `aws_instance.app`:

**Il launch template** è lo **stampo** del server: AMI, tipo, Security Group, instance profile, disco, script di avvio. Tutto quello che prima stava dentro `aws_instance`, ma senza creare niente: descrive **come** fare un server. Ogni modifica crea una **nuova versione** dello stampo.

**L'Auto Scaling group** (ASG) è il **pastore**: tiene d'occhio quanti server ci sono e quanti ce ne dovrebbero essere. Con `min_size = max_size = desired_capacity = 1` il suo compito è semplice: **ci deve essere sempre esattamente un server**. Se il server si ferma o sparisce, l'ASG ne crea un altro dallo stampo. E lo può creare in **qualunque subnet privata**: se un'AZ intera si ferma, il nuovo server nasce nell'altra.

```mermaid
flowchart LR
    LT["Launch template<br/>(lo stampo)<br/>versione 1, 2, 3…"] --> ASG["Auto Scaling group<br/>min = max = 1<br/>(il pastore)"]
    ASG -->|"crea dallo stampo"| S1["Server<br/>AZ-a"]
    S1 -.->|"si rompe / AZ ferma"| X(("✗"))
    ASG -->|"ne crea un altro,<br/>anche nell'altra AZ"| S2["Server<br/>AZ-b"]
```

Come fa l'ASG a capire che il server "si è rotto"? Con `health_check_type = "EC2"` controlla solo che la macchina sia **accesa**. Se la macchina è accesa ma l'applicazione è bloccata, l'ASG non se ne accorge: per quel caso serve un controllo più fine, che vediamo nella dispensa 9.

---

## 5. Un nome che non cambia: `app.corso.internal`

Ogni server nuovo ha un **indirizzo IP nuovo**. Dalla dispensa 5 ci colleghiamo all'applicazione da casa con l'indirizzo del server, `10.20.10.x`: dopo una sostituzione, quell'indirizzo non vale più. Ci serve un **nome** che segua il server, come l'endpoint del database (dispensa 6).

**Route 53** è il servizio DNS di AWS (DNS: dispensa 2). Una **zona privata** è un elenco di nomi che esiste **solo dentro il nostro VPC**: da internet non si vede. Creiamo la zona `corso.internal` (il suffisso `.internal` è riservato proprio alle reti private) con un solo nome:

```
app.corso.internal  →  l'indirizzo del server di adesso
```

Chi aggiorna il nome? **Il server stesso**, all'avvio: legge il proprio indirizzo e scrive in Route 53 "`app.corso.internal` sono io". Si chiama *UPSERT*: crea il nome se non c'è, lo aggiorna se c'è. Il server ha il permesso di cambiare **solo quel nome** in **solo quella zona**.

Resta un problema: la zona è privata, e il tuo computer a casa non è nel VPC. Come fa a sapere cosa vuol dire `app.corso.internal`? Con lo **split DNS** di Tailscale: diciamo al tailnet "per i nomi che finiscono in `corso.internal`, chiedi al DNS del VPC". Il DNS del VPC sta sempre al secondo indirizzo della rete, `10.20.0.2`. Il router lo annuncia come le subnet (dispensa 5), e la policy del tailnet permette a tutti e due i gruppi di interrogarlo sulla porta 53, la porta del DNS.

```mermaid
sequenceDiagram
    participant PC as Il tuo PC (Tailscale)
    participant R as Router Tailscale
    participant DNS as DNS del VPC (10.20.0.2)
    participant APP as Server (10.20.11.37)
    PC->>R: app.corso.internal? (split DNS: chiedi al VPC)
    R->>DNS: app.corso.internal?
    DNS-->>PC: 10.20.11.37 (scritto dal server all'avvio)
    PC->>R: http://10.20.11.37:8080
    R->>APP: richiesta
```

---

## 6. L'avvio del server: lo script

Ogni server nuovo esegue lo **stesso script di avvio** (`user_data`, dispensa 4), che lo porta da "macchina vuota" ad "applicazione in funzione" senza che nessuno ci metta le mani. Questa volta lo script è lungo, quindi non lo scriviamo dentro il blocco Terraform: sta in un file a parte, `user_data.sh.tpl`, e Terraform lo riempie con i valori giusti (sezione 12).

```mermaid
flowchart TD
    A["1. Installa Docker<br/>e docker compose"] --> B["2. Scarica docker-compose.yml da S3<br/>e il certificato della CA di RDS"]
    B --> C["3. Scrive in Route 53:<br/>app.corso.internal = il mio indirizzo"]
    C --> D["4. Crea lo script app-aggiorna:<br/>legge la password, la scrive in app.env,<br/>avvia l'app se è cambiata"]
    D --> E["5. Lo lancia subito,<br/>poi ogni 5 minuti (timer)"]
    E --> F["L'app parte: crea le tabelle se mancano,<br/>risponde sulla 8080"]
```

**docker compose** è lo strumento che avvia uno o più container descritti in un file, `docker-compose.yml`: quale immagine, quali porte, quali variabili, quali file da montare. Il nostro ha un solo container, l'applicazione; un'applicazione vera ne ha spesso di più (l'app, un worker, una cache), e il file resta lo stesso strumento.

Il file `docker-compose.yml` sta nel bucket della dispensa 7, sotto `deploy/`: lo carica **Terraform**, con una risorsa `aws_s3_object`. Se lo modifichi, al prossimo `apply` cambia anche lo stampo del server (sezione 9), e il server viene sostituito.

---

## 7. La password che cambia: il timer `app-aggiorna`

L'applicazione ha bisogno della password del database. Ma il **container non può leggere Secrets Manager** da solo (vedi la sezione 8), e la password **cambia ogni 7 giorni** (dispensa 6).

La soluzione è un piccolo script sul server, `app-aggiorna`, che fa da intermediario:

1. legge la password dal segreto, con il role del server;
2. scrive un file `/etc/app/app.env` con l'indirizzo del database, il nome utente e la password (leggibile solo da root);
3. **se il file è cambiato** rispetto a prima, (ri)avvia l'applicazione, che così riparte con la password nuova.

Lo script gira all'avvio e poi **ogni 5 minuti**, grazie a un **timer di systemd** (systemd è il programma che avvia e controlla i servizi su Linux, dispensa 4; un timer è la sua "sveglia" che lancia un servizio a intervalli regolari). Quando RDS cambia la password, entro 5 minuti l'applicazione riparte con quella nuova: qualche secondo di fermo una volta a settimana. Nei minuti tra la rotazione e il controllo, la nostra app, che apre una connessione nuova a ogni richiesta, risponde 503: al massimo 5 minuti. Se non è accettabile, c'è l'alternativa della dispensa 6 (rotazione decisa da te).

---

## 8. Un'altra protezione: IMDS con hop limit 1

Il server legge le credenziali del suo role dal **servizio dei metadati**, IMDS (dispensa 4). Ma su quel server ora girano dei **container**: se un'applicazione avesse una falla che le fa fare richieste a indirizzi scelti da un attaccante (un attacco molto comune, si chiama *SSRF*), l'attaccante potrebbe chiederle di leggere le credenziali del server da IMDS.

L'**hop limit** (limite di salti) è il numero di passaggi di rete che una risposta di IMDS può fare. Il server stesso è a 1 salto; un container è a **2 salti**, perché tra lui e la scheda di rete c'è la rete interna di Docker. Con `http_put_response_hop_limit = 1`:

- lo **script di avvio** e `app-aggiorna`, che girano sul server, leggono le credenziali normalmente;
- l'**applicazione nel container** non le raggiunge: la risposta muore dopo un salto.

L'immagine Amazon Linux 2023 parte con il limite a 2, proprio per permettere ai container di usare il role del server. Noi lo riportiamo a 1: la nostra applicazione non ha bisogno di credenziali AWS (la password gliela passa `app-aggiorna`), quindi chiudiamo la porta.

---

## 9. Una nuova versione: instance refresh

Come si passa da una versione all'altra? Cambiando la variabile **`image_tag`** nel tfvars e lanciando `terraform apply`.

1. Il tag finisce nello script di avvio, quindi lo **stampo cambia**: Terraform crea una **nuova versione** del launch template.
2. L'ASG ha un blocco **`instance_refresh`**: quando lo stampo cambia, **sostituisce** il server con uno nuovo, fatto dal nuovo stampo.
3. Il server nuovo esegue lo script di avvio, scarica la nuova immagine, scrive il suo indirizzo in Route 53, e parte.

Con le impostazioni `min_healthy_percentage = 0` e `max_healthy_percentage = 100`, l'ASG **prima toglie** il server vecchio e **poi crea** quello nuovo: per qualche minuto l'applicazione non risponde. Per evitarlo bisognerebbe creare prima il nuovo e togliere il vecchio solo quando il nuovo è **davvero pronto**, e per sapere quando è "davvero pronto" serve un controllo dell'applicazione, non solo della macchina: è il tema della dispensa 9.

Lo stesso meccanismo scatta per **qualunque** modifica dello stampo: lo script di avvio, il `docker-compose.yml`, il tipo di server. E anche per una **nuova AMI**: lo stampo legge l'ultima Amazon Linux dal parametro di AWS (dispensa 4), quindi un `apply` fatto dopo un aggiornamento di Amazon Linux sostituisce il server con uno aggiornato. Qui, a differenza della dispensa 4, non usiamo `ignore_changes`: con un server sostituibile, un aggiornamento di sicurezza non è più un rischio, è un vantaggio.

**Tornare indietro** (*rollback*) è la stessa operazione al contrario: rimetti in `image_tag` il commit precedente, `terraform apply`. Grazie ai tag immutabili, l'immagine di allora è ancora lì, identica.

---

## 10. Le migrazioni del database

Una nuova versione dell'applicazione spesso ha bisogno di una **modifica al database**: una tabella nuova, una colonna in più. Queste modifiche si chiamano **migrazioni**. Nel nostro esempio è la cosa più semplice possibile: all'avvio l'applicazione esegue `CREATE TABLE IF NOT EXISTS`, "crea la tabella se non c'è". Le applicazioni vere usano strumenti dedicati (per esempio Alembic per Python, Flyway per Java) che tengono il conto di quali migrazioni sono già state fatte; il principio è lo stesso: **le migrazioni partono all'avvio, prima che l'applicazione inizi a rispondere**.

Una regola da non dimenticare, per via del rollback: **una migrazione deve funzionare anche con la versione precedente dell'applicazione**. Se la versione 2 cancella una colonna che la versione 1 usa, tornare alla versione 1 è impossibile: il codice vecchio cerca una colonna che non c'è più. Il modo sicuro si fa in due rilasci: prima si **aggiunge** il nuovo (la versione 2 usa la colonna nuova, ma la vecchia c'è ancora); solo quando non si torna più indietro, si **toglie** il vecchio (versione 3). In inglese si chiama *expand and contract*.

---

## 11. Il quadro completo

```mermaid
flowchart TB
    DEV["Tu: git push"] --> GH["GitHub Actions<br/>(OIDC, nessuna chiave)"]
    GH -->|"immagine app:commit"| ECR[("ECR")]
    HOME["Il tuo PC<br/>Tailscale"] -->|"http://app.corso.internal:8080"| R

    subgraph VPC["VPC 10.20.0.0/16 · Milano"]
        R["Router Tailscale<br/>SG vpn"]
        DNSV["DNS del VPC<br/>10.20.0.2<br/>zona corso.internal"]
        subgraph PRIV["Subnet private (AZ-a, AZ-b)"]
            ASG["Auto Scaling group min=max=1"] --> APP["Server app<br/>docker compose<br/>SG app · IMDSv2 hop 1"]
        end
        subgraph DBN["Subnet database"]
            DB[("RDS PostgreSQL 18<br/>Multi-AZ · TLS")]
        end
        R --> APP
        R --> DNSV
        APP -->|"5432 verify-full"| DB
        APP -->|"UPSERT app.corso.internal"| DNSV
    end

    APP -->|"pull (pezzi via Gateway Endpoint)"| ECR
    APP -->|"docker-compose.yml"| S3[("S3 artefatti")]
    APP -->|"password, ogni 5 min"| SM[("Secrets Manager")]
```

---

## 12. Il codice Terraform

Questa dispensa tocca più file. Li vediamo uno per uno: prima l'infrastruttura (`infra/`), poi il repository dell'applicazione (`corso-app/`).

### ec2.tf: cosa togliere

Il server non è più una `aws_instance`: in `ec2.tf` **cancella** il blocco `resource "aws_instance" "app"` e, in `outputs.tf`, gli output `app_instance_id` e `app_private_ip`. Tutto il resto di `ec2.tf` **resta**: il parametro dell'AMI, la trust policy `ec2_trust`, il role `app` con le sue policy e l'instance profile, che ora usa lo stampo.

Al prossimo `apply` Terraform cancellerà il vecchio server; l'ASG ne creerà uno nuovo.

### variables.tf (aggiunte)

```hcl
variable "github_repo" {
  description = "Il repository dell'applicazione, nella forma utente/nome"
  type        = string
}

variable "image_tag" {
  description = "La versione dell'applicazione da mettere in produzione (il commit)"
  type        = string
}
```

`image_tag` **non ha un default**, di proposito: la versione in produzione deve essere sempre una scelta scritta, mai un valore implicito.

### github-oidc.tf

```hcl
# ---------------------------------------------------------------
# AWS si fida dei token firmati da GitHub Actions
# ---------------------------------------------------------------

resource "aws_iam_openid_connect_provider" "github" {
  url            = "https://token.actions.githubusercontent.com"
  client_id_list = ["sts.amazonaws.com"]
}

# ---------------------------------------------------------------
# Il role che GitHub assume: solo il nostro repository, solo main
# ---------------------------------------------------------------

data "aws_iam_policy_document" "github_trust" {
  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]

    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.github.arn]
    }

    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"
      values   = ["sts.amazonaws.com"]
    }

    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:sub"
      values   = ["repo:${var.github_repo}:ref:refs/heads/main"]
    }
  }
}

resource "aws_iam_role" "github_release" {
  name               = "${var.project}-github-release"
  assume_role_policy = data.aws_iam_policy_document.github_trust.json
}

# ---------------------------------------------------------------
# Cosa può fare: caricare immagini nel nostro repository, e basta
# ---------------------------------------------------------------

data "aws_iam_policy_document" "github_push" {
  statement {
    actions   = ["ecr:GetAuthorizationToken"]
    resources = ["*"]
  }

  statement {
    actions = [
      "ecr:BatchCheckLayerAvailability",
      "ecr:InitiateLayerUpload",
      "ecr:UploadLayerPart",
      "ecr:CompleteLayerUpload",
      "ecr:PutImage",
      "ecr:BatchGetImage",
    ]
    resources = [aws_ecr_repository.app.arn]
  }
}

resource "aws_iam_policy" "github_push" {
  name   = "${var.project}-github-push"
  policy = data.aws_iam_policy_document.github_push.json
}

resource "aws_iam_role_policy_attachment" "github_push" {
  role       = aws_iam_role.github_release.name
  policy_arn = aws_iam_policy.github_push.arn
}
```

### deploy/docker-compose.yml

Un file nuovo, nella cartella `infra/deploy/`:

```yaml
services:
  app:
    image: ${IMAGE}
    restart: always
    ports:
      - "8080:8080"
    env_file: /etc/app/app.env
    environment:
      APP_VERSION: ${IMAGE}
    volumes:
      - /etc/app/rds-ca.pem:/certs/rds-ca.pem:ro
```

### user_data.sh.tpl

Un file nuovo, nella cartella `infra/`. Le parti `${…}` le riempie Terraform (sezione "Cosa fa questo codice"); le variabili scritte `$NOME`, senza graffe, sono dello script e prendono valore sul server.

```bash
#!/bin/bash
# Avvio del server dell'applicazione.
# Versione del docker-compose: ${compose_etag}
set -euo pipefail
export AWS_DEFAULT_REGION=${region}

# 1. Docker e docker compose
dnf install -y docker
systemctl enable --now docker
mkdir -p /usr/local/lib/docker/cli-plugins
curl -fsSLo /usr/local/lib/docker/cli-plugins/docker-compose \
  https://github.com/docker/compose/releases/latest/download/docker-compose-linux-aarch64
chmod +x /usr/local/lib/docker/cli-plugins/docker-compose

# 2. I file: docker-compose dal bucket, certificato della CA di RDS, immagine da usare
mkdir -p /etc/app
aws s3 cp s3://${bucket}/deploy/docker-compose.yml /etc/app/docker-compose.yml
curl -fsSo /etc/app/rds-ca.pem https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem
echo "IMAGE=${image}" > /etc/app/.env

# 3. Il nome stabile: app.corso.internal punta a questo server
TOKEN=$(curl -fsX PUT http://169.254.169.254/latest/api/token \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 60")
MY_IP=$(curl -fs -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/local-ipv4)
cat > /tmp/dns.json <<DNS
{"Changes": [{"Action": "UPSERT", "ResourceRecordSet": {
  "Name": "${app_name}", "Type": "A", "TTL": 30,
  "ResourceRecords": [{"Value": "$MY_IP"}]}}]}
DNS
aws route53 change-resource-record-sets --hosted-zone-id ${zone_id} \
  --change-batch file:///tmp/dns.json

# 4. app-aggiorna: legge la password e (ri)avvia l'app se qualcosa è cambiato
cat > /usr/local/bin/app-aggiorna <<'SCRIPT'
#!/bin/bash
set -euo pipefail
export AWS_DEFAULT_REGION=${region}
PASSWORD=$(aws secretsmanager get-secret-value --secret-id ${secret_arn} \
  --query SecretString --output text \
  | python3 -c 'import json,sys; print(json.load(sys.stdin)["password"])')
NUOVO=$(mktemp)
trap 'rm -f "$NUOVO"' EXIT
cat > "$NUOVO" <<ENV
DB_HOST=${db_host}
DB_NAME=app
DB_USER=dbadmin
DB_PASSWORD='$PASSWORD'
ENV
if ! cmp -s "$NUOVO" /etc/app/app.env; then
  install -m 600 "$NUOVO" /etc/app/app.env
  if ! { aws ecr get-login-password | docker login --username AWS --password-stdin ${registry} \
         && docker compose --project-directory /etc/app up -d --force-recreate; }; then
    rm -f /etc/app/app.env # non è partita: al prossimo giro riprova
    exit 1
  fi
fi
SCRIPT
chmod 700 /usr/local/bin/app-aggiorna

# 5. Lo lancia adesso, e poi ogni 5 minuti
cat > /etc/systemd/system/app-aggiorna.service <<'UNIT'
[Unit]
Description=Legge la password del database e aggiorna l'applicazione
After=docker.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/app-aggiorna
UNIT

cat > /etc/systemd/system/app-aggiorna.timer <<'UNIT'
[Unit]
Description=Ogni 5 minuti: app-aggiorna

[Timer]
OnBootSec=5min
OnUnitActiveSec=5min

[Install]
WantedBy=timers.target
UNIT

systemctl daemon-reload
systemctl enable --now app-aggiorna.timer
systemctl start app-aggiorna.service || true # se fallisce, riprova il timer
```

### deploy.tf

```hcl
# ---------------------------------------------------------------
# Il nome stabile: la zona DNS privata corso.internal
# ---------------------------------------------------------------

resource "aws_route53_zone" "internal" {
  name          = "corso.internal"
  force_destroy = true # il nome app.* lo scrive il server, non Terraform

  vpc {
    vpc_id = aws_vpc.main.id
  }
}

# ---------------------------------------------------------------
# Il docker-compose.yml, caricato nel bucket degli artefatti
# ---------------------------------------------------------------

resource "aws_s3_object" "compose" {
  bucket = aws_s3_bucket.artifacts.id
  key    = "deploy/docker-compose.yml"
  source = "${path.module}/deploy/docker-compose.yml"
  etag   = filemd5("${path.module}/deploy/docker-compose.yml")
}

# ---------------------------------------------------------------
# Il server può aggiornare SOLO il suo nome, in SOLO questa zona
# ---------------------------------------------------------------

data "aws_iam_policy_document" "app_dns" {
  statement {
    actions   = ["route53:ChangeResourceRecordSets"]
    resources = [aws_route53_zone.internal.arn]

    condition {
      test     = "ForAllValues:StringEquals"
      variable = "route53:ChangeResourceRecordSetsNormalizedRecordNames"
      values   = ["app.corso.internal"]
    }
  }
}

resource "aws_iam_policy" "app_dns" {
  name   = "${var.project}-app-dns"
  policy = data.aws_iam_policy_document.app_dns.json
}

resource "aws_iam_role_policy_attachment" "app_dns" {
  role       = aws_iam_role.app.name
  policy_arn = aws_iam_policy.app_dns.arn
}

# ---------------------------------------------------------------
# Lo stampo del server
# ---------------------------------------------------------------

resource "aws_launch_template" "app" {
  name_prefix            = "${var.project}-app-"
  image_id               = data.aws_ssm_parameter.al2023.insecure_value
  instance_type          = var.instance_type
  vpc_security_group_ids = [aws_security_group.app.id]
  update_default_version = true

  iam_instance_profile {
    name = aws_iam_instance_profile.app.name
  }

  metadata_options {
    http_tokens                 = "required" # IMDSv2
    http_put_response_hop_limit = 1          # i container non arrivano a IMDS
  }

  block_device_mappings {
    device_name = "/dev/xvda"
    ebs {
      volume_type = "gp3"
      volume_size = 20 # spazio per le immagini Docker
      encrypted   = true
    }
  }

  user_data = base64encode(templatefile("${path.module}/user_data.sh.tpl", {
    region       = var.region
    bucket       = aws_s3_bucket.artifacts.id
    compose_etag = aws_s3_object.compose.etag
    image        = "${aws_ecr_repository.app.repository_url}:${var.image_tag}"
    registry     = split("/", aws_ecr_repository.app.repository_url)[0]
    zone_id      = aws_route53_zone.internal.zone_id
    app_name     = "app.${aws_route53_zone.internal.name}"
    db_host      = aws_db_instance.main.address
    secret_arn   = aws_db_instance.main.master_user_secret[0].secret_arn
  }))

  tag_specifications {
    resource_type = "instance"
    tags          = { Name = "${var.project}-app" }
  }
}

# ---------------------------------------------------------------
# Il pastore: sempre esattamente un server, in una delle subnet private
# ---------------------------------------------------------------

resource "aws_autoscaling_group" "app" {
  name                      = "${var.project}-app"
  min_size                  = 1
  max_size                  = 1
  desired_capacity          = 1
  vpc_zone_identifier       = [for s in aws_subnet.private : s.id]
  health_check_type         = "EC2"
  health_check_grace_period = 300

  launch_template {
    id      = aws_launch_template.app.id
    version = aws_launch_template.app.latest_version
  }

  instance_refresh {
    strategy = "Rolling"
    preferences {
      min_healthy_percentage = 0   # prima toglie il vecchio…
      max_healthy_percentage = 100 # …poi crea il nuovo
    }
  }
}
```

### tailscale.tf (modifiche)

Tre modifiche al file della dispensa 5, per lo split DNS (sezione 5). In cima al file, un nuovo `locals`:

```hcl
locals {
  vpc_dns = cidrhost(var.vpc_cidr, 2) # il DNS del VPC: 10.20.0.2
}
```

Nella policy `tailscale_acl.main`, una regola in più in `acls` e la rotta in più in `autoApprovers`:

```hcl
      {
        action = "accept"
        src    = ["group:admin", "group:dev"]
        dst    = ["${local.vpc_dns}:53"]
      }
```

```hcl
    autoApprovers = {
      routes = { for cidr in concat(local.private_cidrs, local.db_cidrs, ["${local.vpc_dns}/32"]) : cidr => ["tag:router-aws"] }
    }
```

Nel `user_data` del router, la rotta in più tra quelle annunciate:

```hcl
      --advertise-routes=${join(",", concat(local.private_cidrs, local.db_cidrs, ["${local.vpc_dns}/32"]))}
```

E in fondo al file, una risorsa nuova:

```hcl
# Per i nomi *.corso.internal, chiedi al DNS del VPC
resource "tailscale_dns_split_nameservers" "internal" {
  domain      = "corso.internal"
  nameservers = [local.vpc_dns]
}
```

### outputs.tf (aggiunte)

```hcl
output "github_release_role_arn" {
  value = aws_iam_role.github_release.arn
}

output "app_asg_name" {
  value = aws_autoscaling_group.app.name
}

output "app_url" {
  value = "http://app.${aws_route53_zone.internal.name}:8080"
}
```

### Il repository dell'applicazione: `corso-app`

Tre file. L'applicazione, `app.py`: una pagina che conta le visite nel database.

```python
"""Applicazione del corso: conta le visite in PostgreSQL."""

import http.server
import os

import psycopg

DSN = (
    f"host={os.environ['DB_HOST']} dbname={os.environ['DB_NAME']} "
    f"user={os.environ['DB_USER']} password={os.environ['DB_PASSWORD']} "
    "sslmode=verify-full sslrootcert=/certs/rds-ca.pem connect_timeout=5"
)
VERSION = os.environ.get("APP_VERSION", "sconosciuta")


def migra() -> None:
    """La migrazione: crea la tabella se non c'è."""
    with psycopg.connect(DSN) as conn:
        conn.execute(
            "CREATE TABLE IF NOT EXISTS visite ("
            "id serial PRIMARY KEY, quando timestamptz NOT NULL DEFAULT now())"
        )


class Pagina(http.server.BaseHTTPRequestHandler):
    def do_GET(self) -> None:
        try:
            # una connessione nuova a ogni richiesta: dopo un failover
            # si rilegge il nome dell'endpoint (dispensa 6)
            with psycopg.connect(DSN) as conn:
                conn.execute("INSERT INTO visite DEFAULT VALUES")
                visite = conn.execute("SELECT count(*) FROM visite").fetchone()[0]
            codice, testo = 200, f"<h1>Versione {VERSION}</h1><p>Visite: {visite}</p>"
        except psycopg.Error:
            codice, testo = 503, "<h1>Database non raggiungibile</h1>"
        self.send_response(codice)
        self.send_header("Content-Type", "text/html; charset=utf-8")
        self.end_headers()
        self.wfile.write(testo.encode())


if __name__ == "__main__":
    migra()
    http.server.ThreadingHTTPServer(("", 8080), Pagina).serve_forever()
```

La ricetta, `Dockerfile`:

```dockerfile
FROM public.ecr.aws/docker/library/python:3.13-slim
RUN pip install --no-cache-dir "psycopg[binary]==3.2.*"
WORKDIR /app
COPY app.py .
CMD ["python", "app.py"]
```

Il workflow, `.github/workflows/release.yml`:

```yaml
name: release

on:
  push:
    branches: [main]
  workflow_dispatch: # si può lanciare anche a mano, dalla pagina Actions

permissions:
  id-token: write # serve per ottenere il token OIDC
  contents: read

jobs:
  immagine:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7

      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: ${{ vars.AWS_RELEASE_ROLE_ARN }}
          aws-region: eu-south-1

      - uses: aws-actions/amazon-ecr-login@v2

      - uses: docker/setup-qemu-action@v4

      - uses: docker/setup-buildx-action@v4

      - uses: docker/build-push-action@v7
        with:
          platforms: linux/arm64
          provenance: false
          push: true
          tags: ${{ vars.ECR_REPOSITORY_URL }}:${{ github.sha }}
```

### Cosa fa questo codice, blocco per blocco

**`aws_iam_openid_connect_provider.github`** registra GitHub come "ente che rilascia documenti validi" (sezione 2). `url` è l'indirizzo da cui GitHub pubblica le chiavi con cui firma i token; `client_id_list` è il destinatario che i token devono dichiarare (`aud`, *audience*): `sts.amazonaws.com`, cioè "questo token è per AWS". Ne esiste **uno per account** con quell'indirizzo: se l'account ne ha già uno (creato da un altro progetto), l'`apply` dà errore e bisogna usare quello esistente.

**`data.aws_iam_policy_document.github_trust`** è la trust policy del role (dispensa 1), con tre differenze rispetto a quelle viste finora:

| Parte | Valore | Significato |
|---|---|---|
| `actions` | `sts:AssumeRoleWithWebIdentity` | assumere il role presentando un token esterno, invece di essere un utente o un servizio AWS |
| `principals` | `type = "Federated"`, l'ARN del provider | "chi presenta un token di GitHub" |
| `condition` su `aud` | `sts.amazonaws.com` | il token è stato emesso per AWS |
| `condition` su `sub` | `repo:mario/corso-app:ref:refs/heads/main` | **solo** quel repository, **solo** il branch `main` |

Le due condizioni sono in due blocchi separati: devono essere vere **tutte e due**. `StringEquals` vuol dire "uguale, lettera per lettera", senza asterischi.

**`aws_iam_role.github_release`** e la sua permission policy `github_push`: il login al registry su `*` (come per il server, dispensa 7), e le sei azioni che servono a **caricare** un'immagine (controllare quali pezzi ci sono già, caricare i pezzi nuovi, registrare l'immagine, rileggerla per verifica) solo sul nostro repository. GitHub non può fare nient'altro: né leggere segreti, né toccare server o database, né caricare in altri repository.

**`deploy/docker-compose.yml`** descrive l'unico container. `image: ${IMAGE}` prende l'immagine dal file `/etc/app/.env`, che docker compose legge da solo (lo scrive lo script, passo 2). `restart: always` riavvia il container se si ferma, e anche dopo un riavvio del server. `ports` pubblica la porta 8080 del container sulla 8080 del server, quella che il SG `app` apre al router (dispensa 3). `env_file` passa al container le variabili del file scritto da `app-aggiorna`. `volumes` rende visibile dentro il container, in sola lettura (`ro`), il certificato della CA di RDS scaricato dallo script: serve per `verify-full` (dispensa 6). Attenzione: questo file **non** passa da `templatefile`, quindi `${IMAGE}` arriva così com'è sul server, ed è docker compose a sostituirlo.

**`user_data.sh.tpl`** è lo script della sezione 6. Le righe da notare:

- `set -euo pipefail`: se un comando fallisce, lo script si ferma subito invece di andare avanti a metà (il suo racconto si legge sul server in `/var/log/cloud-init-output.log`);
- `export AWS_DEFAULT_REGION=${region}`: la regione per i comandi `aws`;
- passo 1: Docker dai pacchetti di Amazon Linux; **docker compose** non c'è, quindi si scarica dalla pagina dei rilasci del progetto (`latest`, l'ultima versione; in produzione si scrive una versione precisa);
- passo 3: il token e l'indirizzo letti da IMDS (dispensa 4), poi il file JSON con l'UPSERT e il comando che lo manda a Route 53. `TTL 30`: chi legge il nome lo ricorda al massimo 30 secondi, così dopo una sostituzione il nome nuovo si vede subito;
- passo 4: lo script `app-aggiorna` viene **scritto** su disco con un heredoc (dispensa 4). Il delimitatore `'SCRIPT'` tra apici dice a bash di non toccare niente mentre lo scrive: `$PASSWORD` e `$NUOVO` restano scritti così e prendono valore solo quando lo script gira. Le parti `${…}` invece le ha già riempite Terraform. `DB_PASSWORD='$PASSWORD'` è tra apici singoli perché docker compose prenda la password lettera per lettera, anche se contiene simboli. `cmp -s` confronta il file nuovo con quello vecchio: se sono uguali non fa niente; se sono diversi (primo avvio, o password cambiata) installa il file con permessi `600` (leggibile solo da root), fa il login a ECR e (ri)crea il container. Se il login o l'avvio falliscono (per esempio l'immagine non esiste ancora), cancella il file: così al giro successivo lo vede "cambiato" e riprova. `trap … EXIT` cancella il file temporaneo comunque vada a finire lo script;
- passo 5: un servizio systemd che esegue `app-aggiorna` una volta (`Type=oneshot`), e un **timer** che lo rilancia 5 minuti dopo l'avvio e poi ogni 5 minuti. Prima si attiva il timer, poi `systemctl start` lo esegue subito la prima volta; `|| true` vuol dire "anche se fallisce, lo script di avvio va avanti": ci riproverà il timer.

**`aws_route53_zone.internal`** crea la zona `corso.internal`. Il blocco `vpc` la rende **privata** e la collega al nostro VPC: solo lì dentro esiste. `force_destroy = true` serve al `destroy`: una zona che contiene nomi non si può cancellare, e il nome `app.corso.internal` non lo conosce Terraform (lo scrive il server), quindi Terraform non saprebbe toglierlo da solo.

**`aws_s3_object.compose`** carica il file `deploy/docker-compose.yml` nel bucket, sotto `deploy/`, dove il server ha il permesso di leggere (dispensa 7). `path.module` è la cartella del progetto. `etag = filemd5(…)` è l'**impronta** del file (`filemd5` calcola un codice che cambia se cambia anche un solo carattere): se modifichi il file, l'impronta cambia, e Terraform sa che va ricaricato.

**`data.aws_iam_policy_document.app_dns`**: il server può fare `ChangeResourceRecordSets` (cambiare i nomi) **solo** sulla nostra zona, e la condizione `route53:ChangeResourceRecordSetsNormalizedRecordNames` restringe ulteriormente a **un solo nome**, `app.corso.internal`. `ForAllValues:StringEquals` vuol dire "**tutti** i nomi della modifica devono essere in questo elenco". Così anche un server compromesso non potrebbe dirottare altri nomi. Il role del server ora ha quattro policy: SSM, password del database, artefatti e DNS.

**`aws_launch_template.app`** è lo stampo (sezione 4):

| Argomento | Significato |
|---|---|
| `name_prefix` | il nome inizia così, AWS aggiunge un suffisso unico |
| `image_id`, `instance_type`, `vpc_security_group_ids`, `iam_instance_profile` | gli stessi valori del server della dispensa 4 |
| `update_default_version = true` | ogni modifica crea una versione nuova e la rende quella "di default" |
| `metadata_options` | IMDSv2 obbligatorio, e **hop limit 1** (sezione 8) |
| `block_device_mappings` | il disco: `/dev/xvda` è il nome del disco principale in Amazon Linux; 20 GB, perché le immagini Docker occupano spazio |
| `user_data` | lo script, riempito da `templatefile` e codificato con `base64encode` |
| `tag_specifications` | le etichette da mettere sui server creati dallo stampo |

La funzione **`templatefile(file, valori)`** legge un file e ci sostituisce le parti `${…}` con i valori della mappa: `${region}` diventa `eu-south-1`, `${image}` diventa l'indirizzo completo dell'immagine, e così via. È lo stesso meccanismo dell'interpolazione (dispensa 0), ma su un file intero. **`base64encode`** trasforma il testo in un formato che il launch template pretende (a differenza di `aws_instance`, che lo fa da solo). `split("/", url)[0]` divide l'indirizzo del repository alle barre e prende il primo pezzo: il registry (dispensa 7). `compose_etag` serve solo come commento nello script: quando il `docker-compose.yml` cambia, cambia lo script, quindi lo stampo, quindi il server viene sostituito e scarica il file nuovo.

**`aws_autoscaling_group.app`** è il pastore:

| Argomento | Significato |
|---|---|
| `min_size`, `max_size`, `desired_capacity` | tutti a 1: sempre esattamente un server |
| `vpc_zone_identifier` | le subnet in cui può crearlo: tutte le private, una per AZ (espressione `for`, dispensa 2) |
| `health_check_type = "EC2"` | controlla che la macchina sia accesa (sezione 4) |
| `health_check_grace_period = 300` | per i primi 5 minuti dopo la creazione non giudica il server: gli lascia il tempo di avviarsi |
| `launch_template` | lo stampo, nella sua **ultima versione** (`latest_version`) |
| `instance_refresh` | quando lo stampo cambia, sostituisci il server (sezione 9). `Rolling` è la strategia standard; le due percentuali dicono "prima togli, poi crea" |

**Le modifiche a `tailscale.tf`.** La funzione **`cidrhost(rete, n)`** dà l'**n-esimo indirizzo** di una rete (la sorella di `cidrsubnet`, dispensa 2): `cidrhost("10.20.0.0/16", 2)` è `10.20.0.2`, il DNS del VPC. La regola in più nella policy permette a tutti e due i gruppi di interrogarlo sulla porta 53. La rotta `10.20.0.2/32` (un indirizzo solo) va annunciata dal router e approvata in `autoApprovers`. **`tailscale_dns_split_nameservers`** è lo split DNS: per il dominio `corso.internal`, il tailnet manda le domande a `10.20.0.2`. Perché funzioni, nella console di Tailscale deve essere attivo **MagicDNS** (sezione DNS; sui tailnet nuovi è attivo di default).

**`app.py`** è un piccolo server web scritto con la libreria standard di Python, più **psycopg**, la libreria per parlare con PostgreSQL. `DSN` (*Data Source Name*) è la stringa di connessione, costruita dalle variabili d'ambiente scritte da `app-aggiorna`, con `sslmode=verify-full` e il certificato montato dal compose (dispensa 6). `migra()` è la migrazione: crea la tabella `visite` se non c'è. A ogni richiesta la pagina apre una connessione **nuova**, inserisce una visita e la conta: aprire una connessione nuova ogni volta è lento per un'applicazione vera (che usa un *pool*, dispensa 6), ma qui ha un pregio didattico, perché ogni richiesta rilegge il nome dell'endpoint. Se il database non risponde, la pagina restituisce il codice **503** ("servizio non disponibile") invece di un errore incomprensibile.

**`Dockerfile`**: parte dall'immagine di Python (dalla copia di AWS, come nella dispensa 7), installa psycopg, copia `app.py`, e dice quale comando lanciare all'avvio del container.

**`release.yml`**: `on` dice quando parte il workflow (a ogni push su `main`, oppure a mano). `permissions` dà al workflow il permesso di chiedere il token OIDC (`id-token: write`) e di leggere il codice. Il resto sono i sei passi della sezione 3. `${{ vars.NOME }}` è una **variabile del repository**, che imposti su GitHub; `${{ github.sha }}` è il codice del commit. Le versioni delle azioni (`@v7`, `@v6`…) cambiano nel tempo: se ce n'è una più recente, si aggiorna il numero. In produzione conviene "inchiodarle" al codice di un commit (`@3d3c42e…`) invece che al numero di versione: un numero si può spostare, un commit no.

---

## 13. Esercizio

Obiettivo: costruire la catena completa, rilasciare due versioni, tornare indietro, e mettere alla prova la sostituzione del server, la rotazione della password e la ricostruzione dell'infrastruttura.

### Passi

1. **Il repository dell'applicazione.** Su GitHub crea un repository **privato** `corso-app`. Clonalo sul tuo computer e aggiungi i tre file: `app.py`, `Dockerfile`, `.github/workflows/release.yml`. Non fare ancora `git push`: il role su AWS non esiste ancora.
2. **I file dell'infrastruttura.** Nella cartella `infra/`: togli `aws_instance.app` da `ec2.tf` e i suoi due output; aggiungi `github-oidc.tf`, `deploy.tf`, `user_data.sh.tpl`, `deploy/docker-compose.yml`; applica le modifiche a `tailscale.tf`, `variables.tf` e `outputs.tf`. In `terraform.tfvars`:

   ```hcl
   github_repo = "il-tuo-utente/corso-app"
   image_tag   = "nessuna"
   ```

   `"nessuna"` è un tag che non esiste: per ora il server partirà, ma senza applicazione. Lo sistemiamo al passo 7.
3. **`terraform apply -replace=tailscale_tailnet_key.router`.** Il `-replace` serve perché il router va ricreato (annuncia una rotta nuova) e gli serve una chiave nuova (dispensa 5). Nel `plan` controlla: il vecchio `aws_instance.app` viene **cancellato**; nascono lo stampo, l'ASG, la zona DNS, l'OIDC provider e il role di GitHub; il router viene ricreato.
4. **Il server che non trova l'immagine.** In console: **EC2 → Auto Scaling groups → corso-aws-app**: c'è un server. Entra con SSM (**EC2 → Instances**, quello chiamato `corso-aws-app`) e guarda come è andato lo script: `sudo tail -20 /var/log/cloud-init-output.log`, poi `sudo journalctl -u app-aggiorna -n 20`. (Atteso: Docker installato, nome scritto in Route 53; `app-aggiorna` fallisce perché l'immagine `:nessuna` non esiste. Il server è sano, manca solo l'applicazione.)
5. **Le variabili su GitHub.** Leggi i valori:

   ```bash
   terraform output -raw github_release_role_arn
   terraform output -raw ecr_repository_url
   ```

   Su GitHub, nel repository `corso-app`: **Settings → Secrets and variables → Actions → scheda Variables → New repository variable**. Crea `AWS_RELEASE_ROLE_ARN` e `ECR_REPOSITORY_URL` con quei due valori.
6. **Il primo `git push`.** Fai commit dei tre file e `git push`. Su GitHub, scheda **Actions**: il workflow `release` parte. Aprilo e segui i passi; la costruzione per ARM impiega qualche minuto. (Atteso: tutto verde. In ECR c'è un'immagine con il tag uguale al codice del commit, che leggi con `git rev-parse HEAD`.)
7. **In produzione.** In `terraform.tfvars` scrivi `image_tag = "<il codice del commit>"` e lancia `terraform apply`. Nel `plan`: cambia lo stampo (una nuova versione) e l'ASG. Segui la sostituzione:

   ```bash
   aws autoscaling describe-instance-refreshes --auto-scaling-group-name "$(terraform output -raw app_asg_name)" \
     --query 'InstanceRefreshes[0].{stato:Status,percentuale:PercentageComplete}'
   ```

   In console, sotto **Instances**, vedi il server vecchio che si spegne e quello nuovo che nasce.
8. **Da casa.** Con Tailscale acceso, dopo qualche minuto:

   ```bash
   curl "$(terraform output -raw app_url)"
   ```

   (Atteso: `Versione …:<commit>` e `Visite: 1`; riprova, le visite salgono.) Apri lo stesso indirizzo nel browser. Stai raggiungendo per nome un server privato, scelto da un Auto Scaling group, che legge un database Multi-AZ in TLS verificato.
9. **Il nome segue il server.** Leggi l'indirizzo a cui punta il nome: `dig +short app.corso.internal` (oppure `nslookup app.corso.internal`). Ora **termina il server** a mano: in console, **Instances** → il server `corso-aws-app` → **Instance state → Terminate**. Entro qualche minuto l'ASG ne crea un altro, forse nell'altra AZ. Ripeti `dig` e `curl`. (Atteso: indirizzo diverso, stessa pagina, e le **visite non sono ripartite da zero**: i dati sono nel database, non sul server.)
10. **Il container e IMDS.** Entra nel nuovo server con SSM e prova a raggiungere IMDS dal server e dal container:

    ```bash
    curl -s -m 3 -X PUT http://169.254.169.254/latest/api/token \
      -H "X-aws-ec2-metadata-token-ttl-seconds: 60" > /dev/null && echo "server: ok"
    sudo docker compose --project-directory /etc/app exec app python -c \
      "import urllib.request as u; u.urlopen(u.Request('http://169.254.169.254/latest/api/token', method='PUT', headers={'X-aws-ec2-metadata-token-ttl-seconds': '60'}), timeout=3)"
    ```

    (Atteso: il server ottiene il token; dal container la richiesta **scade**: è l'hop limit 1.)
11. **La password cambia.** Dal tuo computer forza una rotazione:

    ```bash
    aws secretsmanager rotate-secret --secret-id "$(terraform output -raw db_secret_arn)"
    ```

    Entro 5 minuti, sul server: `sudo journalctl -u app-aggiorna -n 20`. (Atteso: in una delle esecuzioni, il file è cambiato e il container è stato ricreato.) Da casa, `curl` funziona ancora, con le visite al loro posto.
12. **Una seconda versione, e ritorno.** In `app.py` cambia il titolo, per esempio `Versione {VERSION} – nuova!`. Commit e push: GitHub costruisce un'immagine con un nuovo tag. Mettila in produzione come al passo 7 e controlla con `curl`. Poi **torna indietro**: rimetti in `image_tag` il commit di prima e `terraform apply`. (Atteso: torna la pagina di prima. L'immagine vecchia era ancora in ECR, identica: tag immutabili.)
13. **Chi può caricare immagini.** Su GitHub crea un branch `prova` e, dalla scheda **Actions → release → Run workflow**, lancia il workflow **dal branch `prova`**. (Atteso: il passo `configure-aws-credentials` fallisce, *Not authorized to perform sts:AssumeRoleWithWebIdentity*. Il role accetta solo `main`: è la condizione sul `sub`.)
14. **Ricostruire tutto, tranne i dati** (facoltativo, lungo). Lancia `terraform destroy`. (Atteso: Terraform cancella quasi tutto, ma si ferma con un errore sul database: `deletion_protection`, dispensa 6. Restano il database e ciò da cui dipende: VPC, subnet, Security Group.) Ora lancia `terraform apply -replace=tailscale_tailnet_key.router`: tutto il resto rinasce intorno al database. Il repository ECR però è nuovo e vuoto: su GitHub, **Actions → release → Run workflow** dal branch `main`, per ricostruire l'immagine con lo stesso tag (entro 5 minuti `app-aggiorna` la trova da solo). Da casa, `curl`: le visite sono ancora lì. È la prova che l'infrastruttura è davvero "bestiame": la si può buttare e ricostruire, e i dati, protetti, sopravvivono.
15. **Pulizia.** In `terraform.tfvars` scrivi `db_deletion_protection = false`, `terraform apply`, poi `terraform destroy`. Cancella lo snapshot finale del database (dispensa 6). Su GitHub puoi cancellare le due variabili o l'intero repository `corso-app`.

### Domande di verifica

1. Perché con OIDC non serve salvare nessuna chiave su GitHub? Cosa succede se qualcuno copia un token del workflow e lo usa il giorno dopo?
2. Cosa potrebbe succedere se nella trust policy la condizione sul `sub` fosse `repo:il-tuo-utente/*`? E se mancasse?
3. Cosa vuol dire che il server è "bestiame"? Quali cose, sul server, renderebbero impossibile sostituirlo senza perdere niente?
4. Nel passo 9 il server cambia indirizzo: chi aggiorna `app.corso.internal`, e con quale permesso? Cosa gli impedisce di cambiare altri nomi?
5. Perché il container non può leggere la password da Secrets Manager da solo? Chi lo fa al suo posto, e quando?
6. Cosa succede, passo per passo, quando cambi `image_tag` e lanci `terraform apply`? Perché per qualche minuto l'applicazione non risponde?
7. La versione 2 cancella una colonna della tabella. Perché il rollback alla versione 1 sarebbe un problema, e come si fa la stessa modifica in modo sicuro?
8. Nel passo 14, perché il `destroy` si ferma? Cosa sopravvive, e perché basta un `apply` per tornare a funzionare?

---

## Riepilogo

- **GitHub Actions con OIDC**: GitHub presenta un token firmato, AWS controlla firma, repository e branch e dà credenziali **temporanee**. Nessuna chiave salvata. La condizione sul **`sub`** è la protezione più importante.
- Il workflow costruisce l'immagine **ARM** e la carica in ECR con il **tag del commit**. GitHub può solo **caricare**: quale versione va in produzione lo decidi tu, con `image_tag`.
- Il server è **bestiame**: un **launch template** (lo stampo) e un **Auto Scaling group** con min = max = 1 (il pastore) che lo ricrea da solo, anche in un'altra AZ. Sul server non c'è niente di unico: dati in RDS, programma in ECR, file in S3, password in Secrets Manager.
- Il **nome stabile** `app.corso.internal`, in una zona **Route 53 privata**, lo aggiorna il server all'avvio; da casa lo risolve lo **split DNS** di Tailscale.
- Lo **script di avvio** installa Docker, scarica il compose e il certificato, scrive il nome, e crea **`app-aggiorna`**, che ogni 5 minuti rilegge la password e riavvia l'app se è cambiata.
- **IMDS con hop limit 1**: lo script legge le credenziali del server, i container no.
- Nuova versione = nuovo `image_tag` + `apply` → **instance refresh**. **Rollback** = il tag di prima. Le **migrazioni** partono all'avvio e devono restare compatibili con la versione precedente.
- Con il database protetto, l'infrastruttura si può **distruggere e ricostruire**: i dati sopravvivono.
