# 🇲🇦 Validateur DGI — Conformité Fiscale CGI Maroc

> **Spring Boot 3.4 + Thymeleaf + MySQL** — Extraction factures (PDF/Excel/Word), parsing hybride Gemini/Regex, moteur d'audit DGI (6 règles), génération PDF Flying Saucer, dashboard admin Chart.js.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Java](https://img.shields.io/badge/Java-17-ED8B00?logo=openjdk)](https://openjdk.org)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.4-6DB33F?logo=springboot)](https://spring.io/projects/spring-boot)
[![Thymeleaf](https://img.shields.io/badge/Thymeleaf-3.1-005F0F?logo=thymeleaf)](https://www.thymeleaf.org)
[![MySQL](https://img.shields.io/badge/MySQL-8.0-4479A1?logo=mysql)](https://mysql.com)
[![Maven](https://img.shields.io/badge/Maven-3.9-C71A36?logo=apachemaven)](https://maven.apache.org)
[![Docker](https://img.shields.io/badge/Docker-Ready-blue?logo=docker)](https://docker.com)

---

## ✨ Fonctionnalités

| Module | Description |
|--------|-------------|
| 📄 **Extraction Multi-format** | Apache POI (DOCX, XLSX) + PDFBox (PDF) → Texte brut nettoyé |
| 🤖 **Parsing Hybride** | Gemini 2.5 Flash (REST) + Fallback Regex déterministe (merge champ par champ) |
| ⚖️ **Moteur Audit DGI** | 5 règles DGI + Règle d'appropriation `DGI-OWN-ICE-001` (ICE émetteur = ICE entreprise) |
| 📊 **Ventilation TVA Auto** | Dérivée des lignes si absente (alimente dashboard + PDF) |
| 📋 **Génération PDF** | Thymeleaf → HTML → Flying Saucer (OpenPDF) → PDF téléchargeable |
| 🔐 **Sécurité** | BCrypt + Spring Security Form + JWT Stateless (HS256 SHA-256) + Multi-tenant |
| 👑 **Super Admin** | Dashboard `/admin` : suppression users/entreprises, garde-fous, messages flash |
| 📈 **Dashboard** | KPIs TVA collectée, CA HT/TTC, Conformité, Timbre, Graphiques Chart.js empilés |

---

## 🏗 Architecture

```
validateur-dgi/
├── src/main/java/ma/fiscalys/
│   ├── domain/              # Company, AppUser, Invoice, InvoiceItem, InvoiceVatSummary, ComplianceAudit
│   ├── dto/                 # InvoiceDTO, IssuerDTO, ClientDTO, InvoiceItemDTO, VatSummaryDTO
│   ├── engine/              # DgiRule, DgiAuditEngine, AuditReport, AuditLabels + 5 règles DGI
│   ├── service/parser/      # PoiExtractionService, SpringAiParsingService, FallbackParsingService
│   ├── service/generator/   # PdfGeneratorService (+ PdfGenerationException)
│   ├── service/             # InvoiceService (upload, save, audit, ICE rule, TVA derivation)
│   ├── security/            # JwtTokenProvider, JwtAuthenticationFilter, TenantContext, SuperAdminService
│   ├── controller/          # AdminDashboardController, DashboardController, WorkspaceController...
│   └── config/              # SecurityConfig, GlobalModelAttributes
├── src/main/resources/
│   ├── templates/           # Thymeleaf (fragments, pdf/modern.html, admin.html, dashboard.html...)
│   └── application.properties
├── pom.xml                  # Maven wrapper (mvnw/mvnw.cmd)
├── .gitignore               # Exclut target/, *.class, .env, IDE
└── LICENSE
```

---

## 🚀 Démarrage Rapide

### Prérequis
- Java 17+
- Maven 3.9+ (ou wrapper `./mvnw`)
- MySQL 8.0 (port **3305**, base `validation`, user `root`, pwd `BD_PASSWORD`)

### 1. Configuration
```properties
# application.properties
spring.datasource.url=jdbc:mysql://localhost:3305/validation
spring.datasource.username=root
spring.datasource.password=BD_PASSWORD
app.gemini.api-key=${GEMINI_API_KEY:}     # Optionnel (fallback regex si absent)
app.jwt.secret=your-secret-key
app.super.admin.emails=oalmajir@gmail.com
```

### 2. Compiler & Lancer
```bash
# Tests
./mvnw.cmd test

# SonarQube (port 9000)
./mvnw.cmd org.sonarsource.scanner.maven:sonar-maven-plugin:5.7.0.6970:sonar '-Dsonar.host.url=http://localhost:9000' '-Dsonar.projectKey=fiscalys-dgi' '-Dsonar.login=admin' '-Dsonar.password=ALMAJIR@'

# Run
./mvnw.cmd spring-boot:run
# App sur http://localhost:8086
```

### 3. Accès Mobile/Distant
```bash
# Cloudflare Tunnel (recommandé)
cloudflared tunnel --url http://localhost:8086
# → URL publique https://xxx.trycloudflare.com (pas d'interstitial)

# Ou LAN (même WiFi)
# http://192.168.x.x:8086
# Pare-feu: netsh advfirewall firewall add rule name="ValidateurDGI" dir=in action=allow protocol=TCP localport=8086
```

---

## 📋 Règles d'Audit DGI

| Code | Règle | Vérification |
|------|-------|--------------|
| `DGI-ISSUER-001` | Identifications Fiscales Émetteur | ICE 15ch, IF 6-8ch, RC, Patente, Adresse |
| `DGI-CLIENT-001` | Validation ICE Client (B2B) | ICE Client 15 chiffres |
| `DGI-MATH-001` | Cohérence Mathématique | HT + TVA = TTC + ventilation par taux |
| `DGI-STAMP-001` | Droit de Timbre | Renseigné quand attendu |
| `DGI-VAT-001` | Clauses Exonération TVA | Mentions CGI cohérentes avec TVA déclarée |
| `DGI-OWN-ICE-001` | **Appropriation ICE** | `issuer.ice == company.ice` (bloquant si différent) |

**Statuts** : `COMPLIANT` (aucune erreur) / `NON_COMPLIANT` (>=1 erreur)

---

## 🔐 Sécurité & Multi-tenant

- **Authentification double** : Formulaire (session) + JWT Stateless
- **JWT** : Clé secrète hachée SHA-256 → HS256, expiration 24h, claims `role` + `companyId`
- **Isolation** : `TenantContext` (ThreadLocal) + `SessionUserService` + Repository filters by `companyId`
- **Super Admin** : Email whitelist (`app.super.admin.emails`), `@PreAuthorize("hasRole('SUPER_ADMIN')")`
- **CORS** : `localhost:8443/3000/5173`, CSRF désactivé sur `/api/**`

---

## 📦 Stack Technique

| Couche | Technologies |
|--------|--------------|
| **Language** | Java 17 |
| **Framework** | Spring Boot 3.4.8, Spring MVC, Spring Security, Spring Data JPA |
| **Template** | Thymeleaf 3.1.5 (SSR) |
| **Database** | MySQL 8.0 (Hibernate, `ddl-auto=update`) |
| **Extraction** | Apache POI 5, PDFBox 3 |
| **IA Parsing** | Gemini 2.5 Flash via REST (`x-goog-api-key`) + Regex Fallback |
| **PDF Gen** | Flying Saucer (OpenPDF/ITextRenderer) |
| **Charts** | Chart.js 4.4.1 (CDN) |
| **Tests** | JUnit 5, Mockito, Surefire, JaCoCo |
| **Quality** | SonarQube (port 9000) |

---

## 🧪 Tests & Qualité

```bash
# Tous les tests
./mvnw.cmd test

# Test unique (log filtré)
./mvnw.cmd -Dtest="AdminDashboardControllerTest" test 2>&1 | Out-File "$env:TEMP\opencode\t.log" -Encoding utf8
Select-String -Path "$env:TEMP\opencode\t.log" -Pattern "Tests run:|BUILD"

# SonarQube Quality Gate
curl.exe -s -u admin:ALMAJIR@ "http://localhost:9000/api/qualitygates/project_status?projectKey=fiscalys-dgi"
```

---

## 🤝 Contribuer

1. Fork → Branche feature (`git checkout -b feat/...`)
2. Conventions : Conventional Commits (`feat:`, `fix:`, `chore:`, `docs:`)
3. Tests obligatoires pour nouvelles règles/audits
4. PR → Review → Merge (squash)

---

## 📄 License

MIT License — Voir `LICENSE`

---

## 👤 Auteur

**MASSALABI OUMAROU Almajir**  
🇲🇦 Maroc | Data Science & AI Engineer  
🔗 [GitHub](https://github.com/amassalabi-spec) | 📧 amassalabi@gmail.com

---

> *"Conformité fiscale automatisée = Sérénité entrepreneuriale"*