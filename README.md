# Corso: Infrastruttura AWS con Terraform

Un corso in italiano per chi parte **da zero**: come costruire con Terraform un'infrastruttura AWS completa, un pezzo alla volta.

Alla fine del corso avrai costruito, con un unico progetto Terraform:

- una **rete privata** (VPC) distribuita su due data center;
- i **firewall** che decidono cosa può passare;
- un **accesso remoto** sicuro (Client VPN con MFA);
- un **server** (EC2) gestito senza SSH;
- un **database PostgreSQL** in alta affidabilità (RDS Multi-AZ);
- un **archivio** per gli artefatti di rilascio (S3);
- il **deploy** completo, dal codice al server.

Non serve conoscere né Terraform né AWS: ogni concetto è spiegato la prima volta che compare, ogni blocco di codice è commentato riga per riga.

```mermaid
flowchart LR
    L0["0 · Terraform<br/>le basi"] --> L1["1 · IAM<br/>chi può fare cosa"]
    L1 --> L2["2 · VPC<br/>la rete"]
    L2 --> L3["3 · Firewall"]
    L3 --> L4["4 · VPN"]
    L4 --> L5["5 · EC2"]
    L5 --> L6["6 · RDS"]
    L6 --> L7["7 · S3"]
    L7 --> L8["8 · Deploy"]
```

---

## Come è organizzato

Ogni lezione ha due materiali:

| Materiale | Cartella | A cosa serve |
|---|---|---|
| **Slide** (PowerPoint) | [`slide/`](slide/) | Per presentare la lezione in aula. Ogni slide ha le **note del relatore** con la spiegazione da dire a voce |
| **Dispensa** (Markdown) | [`dispense/`](dispense/) | Il **materiale di studio**: spiega tutto in dettaglio, con il codice completo, l'esercizio passo passo e i risultati attesi |

Il modo consigliato: si segue la lezione con le slide, poi si studia la dispensa e si fa l'esercizio.

Ogni dispensa ha sempre la stessa struttura: **obiettivo → concetti con un'analogia → codice Terraform spiegato → esercizio con domande di verifica → riepilogo**.

---

## Indice delle lezioni

| # | Lezione | Dispensa | Slide | Stato |
|---|---|---|---|---|
| 0 | **Terraform: le basi**: cos'è, come si scrive, come si usa, lo state | [dispensa-0-terraform.md](dispense/dispensa-0-terraform.md) | [slide-0-terraform.pptx](slide/slide-0-terraform.pptx) | ✅ Pronta |
| 1 | **IAM: chi può fare cosa**: identità, policy, role, privilegi minimi | [dispensa-1-iam.md](dispense/dispensa-1-iam.md) | [slide-1-iam.pptx](slide/slide-1-iam.pptx) | ✅ Pronta |
| 2 | **VPC: la rete privata**: indirizzi, subnet, route table, NAT | [dispensa-2-vpc.md](dispense/dispensa-2-vpc.md) | [slide-2-vpc.pptx](slide/slide-2-vpc.pptx) | ✅ Pronta |
| 3 | **Firewall**: Security Group e NACL, porte, stateful e stateless | [dispensa-3-firewall.md](dispense/dispensa-3-firewall.md) | [slide-3-firewall.pptx](slide/slide-3-firewall.pptx) | ✅ Pronta |
| 4 | **Accesso remoto**: Client VPN con MFA | – | – | In preparazione |
| 5 | **EC2**: il server | – | – | In preparazione |
| 6 | **RDS PostgreSQL** in alta affidabilità | – | – | In preparazione |
| 7 | **S3**: il repository degli artefatti | – | – | In preparazione |
| 8 | **Deploy end-to-end** | – | – | In preparazione |
| A | **Appendice**: IAM avanzato e audit | – | – | In preparazione |

---

## Prima di iniziare

Ti servono:

- un **account AWS**;
- **Terraform** 1.10 o successivo (`terraform version`);
- **AWS CLI** versione 2 (`aws --version`);
- un editor di testo (es. VS Code) e un terminale.

La regione del corso è **Milano (`eu-south-1`)**, che sugli account nuovi va attivata una volta dalla console. Il login si fa con **IAM Identity Center (SSO)** e credenziali temporanee: mai chiavi fisse. La procedura completa, passo per passo, è nella sezione "Prima di iniziare" e nella sezione 6 della [dispensa 0](dispense/dispensa-0-terraform.md).

> **Attenzione ai costi.** Alcune risorse del corso (per esempio il NAT Gateway, la VPN e il database) si pagano a ore anche quando non si usano. Ogni esercizio si chiude con `terraform destroy`: prendi l'abitudine fin dalla prima lezione e imposta un budget con avviso nella console AWS.

---

## Il progetto Terraform

Tutte le lezioni costruiscono **un unico progetto**, nella stessa cartella: ogni lezione aggiunge i suoi file accanto a quelli delle precedenti.

```
infra/
├── providers.tf       # lezione 0: AWS, regione Milano
├── variables.tf       # le variabili, arricchite lezione dopo lezione
├── terraform.tfvars   # i valori scelti
├── main.tf            # lezione 0: il primo bucket
├── backend.tf         # lezione 0: lo state in un bucket S3
├── iam.tf             # lezione 1
├── vpc.tf             # lezione 2
├── security.tf        # lezione 3
├── vpn.tf             # lezione 4
├── ec2.tf             # lezione 5
├── rds.tf             # lezione 6
├── s3.tf              # lezione 7
└── outputs.tf         # cosa stampare alla fine
```

Parametri tecnici usati in tutto il corso:

| Cosa | Valore |
|---|---|
| Terraform | `>= 1.10` |
| Provider | `hashicorp/aws ~> 6.0`, `hashicorp/random ~> 3.6` |
| Regione | `eu-south-1` (Milano) |
| State | bucket S3 con `use_lockfile = true` (senza DynamoDB) |
| Autenticazione | SSO (IAM Identity Center), profilo CLI `corso` |
| Rete | VPC `10.20.0.0/16`, subnet `/24` su due Availability Zone |

---

## Autore

[cereZ23](https://github.com/cereZ23)
