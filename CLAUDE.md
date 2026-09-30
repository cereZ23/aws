# Corso: Infrastruttura AWS con Terraform

Serie di dispense in italiano per spiegare a chi parte da zero come costruire con Terraform
un'infrastruttura AWS completa: VPC, firewall, VPN, EC2, RDS PostgreSQL in HA, S3 per il deploy.

## Indice e stato

| # | Dispensa | Stato |
|---|---|---|
| 0 | Terraform in 20 minuti | Fatta – `dispense/dispensa-0-terraform.md` |
| 1 | IAM: chi può fare cosa | Fatta – `dispense/dispensa-1-iam.md` |
| 2 | VPC: la rete privata | Fatta – `dispense/dispensa-2-vpc.md` |
| 3 | Firewall: Security Group e NACL | Da scrivere |
| 4 | Accesso remoto: Client VPN con MFA | Da scrivere |
| 5 | EC2: il server | Da scrivere |
| 6 | RDS PostgreSQL in alta affidabilità | Da scrivere |
| 7 | S3: il repository degli artefatti | Da scrivere |
| 8 | Deploy end-to-end | Da scrivere |
| A | Appendice: IAM avanzato e audit | Da scrivere |

### Contenuti previsti per le dispense da scrivere

- **3 Firewall** – SG stateful/allow-only a livello risorsa vs NACL stateless/allow+deny ordinato a livello subnet;
  porte effimere 1024-65535; SG che referenziano altri SG (DB accetta 5432 solo dal SG app e dal SG VPN admin);
  **Security Group di default da svuotare** (`aws_default_security_group` senza regole, controllo CIS) e NACL di default
  (spostati qui dalla dispensa 2 per evitare sovrapposizioni); laboratorio "rompi e ripara" (togliere porte effimere
  dalla NACL, bloccare un IP con deny sulla NACL). File: `security.tf`.
- **4 Client VPN con MFA** – AWS Client VPN; certificato server in ACM; client CIDR `10.100.0.0/22` fuori dal VPC;
  associazione a una subnet per AZ; SG dell'endpoint; autenticazione SAML federata (MFA nell'IdP) vs AD/RADIUS vs mutual TLS;
  authorization rules per gruppo (admin anche subnet DB, dev solo applicative); split vs full tunnel; connection log su
  CloudWatch; VPN vs SSM (shell via SSM, client DB/console via VPN); costi. File: `vpn.tf`.
- **5 EC2** – AMI, instance type, `user_data`, SSM Session Manager senza SSH, instance profile (collega dispensa 1).
- **6 RDS** – DB subnet group su 2 AZ; Multi-AZ istanza vs Multi-AZ DB cluster vs read replica; failover e riconnessione;
  backup, PITR, KMS; password in Secrets Manager (`manage_master_user_password`); 5432 solo da SG app e SG VPN admin;
  `deletion_protection`, `final_snapshot_identifier`; laboratorio failover forzato. File: `rds.tf`, `secrets.tf`.
- **7 S3** – Block Public Access, versioning, cifratura, bucket policy, VPC Gateway Endpoint per S3, lettura via role.
- **8 Deploy** – CI → S3 → EC2 (user_data, SSM Run Command o CodeDeploy); credenziali DB da Secrets Manager;
  migrazioni schema; diagramma complessivo; destroy/ricostruzione con DB protetto.
- **Appendice** – condition key, permission boundary, SCP, AccessDenied via CloudTrail, IAM DB auth per RDS,
  correlazione log VPN + CloudTrail.

## Convenzioni (da rispettare in tutte le dispense)

- Lingua: italiano. Formato: Markdown, un file per dispensa in `dispense/`.
- Struttura fissa: **Obiettivo → concetti con analogia → codice Terraform → Esercizio (passi + domande di verifica) → Riepilogo**.
- Schemi in **Mermaid** (```mermaid): architettura, flussi, alberi di decisione. Almeno 2-3 per dispensa.
- **Nessuna sovrapposizione** tra dispense: un concetto si spiega una volta sola, altrove si rimanda ("lo vediamo nella dispensa N").
- Ogni dispensa costruisce sul progetto delle precedenti: **unica configurazione Terraform**, un file `.tf` per dominio.
- **Il lettore non conosce Terraform**: ogni blocco di codice è seguito da una spiegazione "Cosa fa questo codice"
  risorsa per risorsa (argomenti, riferimenti, perché serve). Un costrutto del linguaggio nuovo si spiega la prima volta
  che compare (sintassi base in dispensa 0; `for_each`, espressioni `for`, `toset`, `slice` in dispensa 2), poi si rimanda.

## Parametri tecnici fissati

- Terraform `>= 1.10`, provider `hashicorp/aws ~> 6.0`, `hashicorp/random ~> 3.6`.
- Backend S3 con `use_lockfile = true` (niente DynamoDB).
- Regione `eu-south-1` (Milano). Variabile `project` (default esercizi: `corso-aws`).
- Autenticazione: SSO / IAM Identity Center, profilo CLI `corso`. Mai access key statiche.
- Policy IAM con `aws_iam_policy_document`; attach con `aws_iam_role_policy_attachment`, mai `aws_iam_policy_attachment`.
- Già definiti nel progetto: `data.aws_caller_identity.current` (dispensa 1), `aws_s3_bucket.demo` (dispensa 0).
- Rete (dispensa 2): VPC `10.20.0.0/16`; subnet via `cidrsubnet(var.vpc_cidr, 8, n)` con `for_each` sulle AZ
  → pubbliche `10.20.0-1.0/24`, private `10.20.10-11.0/24`, database `10.20.20-21.0/24`.
  Variabili `vpc_cidr`, `az_count` (2-3), `single_nat_gateway` (true in lab).
  Risorse: `aws_vpc.main`, `aws_subnet.public|private|db` (mappe per AZ), `aws_nat_gateway.main`,
  route table public (IGW), private una per AZ (NAT), db senza uscita. DNS del VPC attivo. Flow logs facoltativi su S3.
