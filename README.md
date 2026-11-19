# PubMed Intelligent Article Filtration & Research Document Generation System

An enterprise-grade biomedical literature search, intelligent categorization, and regulatory document generation platform. Built on the National Center for Biotechnology Information (NCBI) PubMed E-utilities, the National Library of Medicine (NLM) RxNorm API, and OpenAI's GPT models, this application streamlines nonclinical and clinical research workflows for pharmaceutical scientists, regulatory affairs specialists, and medical researchers.

---

## 📑 Table of Contents

1. [Executive Summary & Purpose](#executive-summary--purpose)
2. [Key Capabilities & Differentiators](#key-capabilities--differentiators)
3. [System Architecture](#system-architecture)
4. [Backend Deep Dive](#backend-deep-dive)
   - [Services Layer](#services-layer)
   - [Hierarchical Category Taxonomies](#hierarchical-category-taxonomies)
   - [Scoring & Ranking Engine](#scoring--ranking-engine)
   - [Document Generation & Template Engines](#document-generation--template-engines)
   - [Reference Document & Prevalence Analysis (EMA ERA)](#reference-document--prevalence-analysis-ema-era)
5. [Frontend Deep Dive](#frontend-deep-dive)
   - [Component Hierarchy](#component-hierarchy)
   - [Cart Management & Storage Migration](#cart-management--storage-migration)
   - [Dual-Column Prevalence & Similarity Viewer](#dual-column-prevalence--similarity-viewer)
   - [Visual Match Verification Badges](#visual-match-verification-badges)
6. [API Reference](#api-reference)
7. [Data Stores & Dictionaries](#data-stores--dictionaries)
8. [Installation & Local Setup](#installation--local-setup)
9. [Deployment Guide](#deployment-guide)
10. [Testing & Quality Assurance](#testing--quality-assurance)
11. [Troubleshooting & FAQ](#troubleshooting--faq)
12. [License](#license)

---

## 1. Executive Summary & Purpose

Regulatory dossier preparation (such as **ICH Common Technical Document (CTD) Module 2.4 Nonclinical Overview** and **Module 2.5 Clinical Overview**), safety pharmacology reviews, and **European Medicines Agency (EMA) Environmental Risk Assessment (ERA)** require precise biomedical literature synthesis. 

Manual PubMed querying is notoriously prone to false positives, homonym confusion, missing brand-generic name links, and labor-intensive manual formatting of citations, abbreviation tables, and study design categorizations.

**PubMed Intelligent Article Filtration System** solves these challenges by combining:
- **Two-Tier Categorization**: Distinct domain hierarchies for **Animal Studies** (Pharmacodynamics, Safety Pharmacology, Pharmacokinetics, Toxicology) and **Human Studies** (Pharmacokinetics, Pharmacodynamics, Efficacy, Safety).
- **RxNorm & MeSH-Powered Query Expansion**: Dynamic resolution of drug synonyms, salt forms, spelling variations, and official MeSH descriptors.
- **Ultra-Strict Context Matching**: Title-first and abstract scoring that demotes or excludes irrelevant studies (e.g., preventing clinical trials in animal searches and vice versa).
- **Automated Document Synthesis**: Native Microsoft Word (`.docx`) generation including full structured reports, short tabular summaries, XML-injected custom templates, abbreviation tables, and bibliographic citation engines.
- **EMA ERA Disease Prevalence Search**: Specialized epidemiological search across European populations to determine Fpen (Market Penetration Factor) with evidence authority tier ranking (WHO/GBD/EMA vs. registries vs. peer-reviewed literature).

---

## 2. Key Capabilities & Differentiators

| Capability | Technical Implementation | Benefit |
|------------|--------------------------|---------|
| **Multi-Tier Drug Synonym Matching** | Curated JSON database (13 therapeutic classes) + NLM RxNorm REST API with 24-hour caching | Never misses literature published under brand names, international nonproprietary names (INN), or combination formulations (e.g. Augmentin ↔ Co-amoxiclav ↔ Amoxicillin/clavulanic acid). |
| **Strict Study-Type Verification** | MeSH validation (`Animals[MeSH Terms]` vs `Humans[MeSH Terms]`) + regex title/abstract scans | Prevents species contamination (e.g., excludes rodent studies from human efficacy searches). |
| **Granular Subheading Filtering** | Path-based taxonomy routing (`pharmacokinetics.absorption`, etc.) with parent-keyword rejection | Prevents broad parent-level terms (e.g., generic "Pharmacokinetics") from polluting specialized subheading results (e.g., "Absorption / Cmax / Tmax"). |
| **Transparency & Audit Trail** | Granular match tracking metadata on every returned article | Displays exact verification chips for Drug Name, Category, Inner Keywords, and Study Design (e.g., RCT, Double-Blind, Cohort, Phase I–IV). |
| **Document Synthesis** | `docx`, `docxtemplater`, `pizzip`, and `mammoth` | Generates fully styled Word documents with auto-extracted abbreviation tables, customizable APA/MLA/Chicago/Harvard/Vancouver references, and biological term italicization. |
| **AI Abstract Summaries** | OpenAI GPT integration (`gpt-3.5-turbo`) with structured fallback rule engine | Formats abstracts into standardized regulatory paragraphs: *"A study was conducted to evaluate [topic]... (Author et al, Year)"*. |
| **Reference Document Upload** | `pdf-parse`, `mammoth`, and TF-IDF / N-gram term extraction | Upload existing dossiers or publications to find kindred literature indexed by similarity and category. |
| **EMA ERA Prevalence Engine** | Country-specific and multi-year Boolean query builder with authority grading | Fetches prevalence rates with quality tiers (Tier 1: WHO/GBD to Tier 4: single studies) and compliance statements. |

---

## 3. System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              REACT CLIENT (SPA)                                 │
│                                                                                 │
│  ┌──────────────┐  ┌──────────────────┐  ┌───────────────┐  ┌────────────────┐  │
│  │  SearchBar   │  │ StudyTypeSelector│  │  CategoryTree │  │ SearchFilters  │  │
│  └──────┬───────┘  └────────┬─────────┘  └───────┬───────┘  └───────┬────────┘  │
│         │                   │                    │                  │           │
│  ┌──────▼───────────────────▼────────────────────▼──────────────────▼────────┐  │
│  │         ResultsDisplay / ArticleCard (Visual Match Summary Chips)         │  │
│  └──────────────────────────────────┬────────────────────────────────────────┘  │
│                                     │                                           │
│  ┌───────────────────────┐   ┌──────▼──────┐   ┌─────────────────────────────┐  │
│  │  PrevalenceSearch /   │   │  SelectCart │   │  ExportModal /              │  │
│  │  ReferenceDocUpload   │   │  (Context)  │   │  UnifiedExportModal /       │  │
│  └───────────────────────┘   └─────────────┘   │  DetailDocumentModal        │  │
│                                                └─────────────────────────────┘  │
└─────────────────────────────────────┬───────────────────────────────────────────┘
                                      │ HTTP / REST (Axios)
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          NODE.JS / EXPRESS BACKEND                              │
│                                                                                 │
│  Middleware: CORS, Express Rate Limit (100 req/min), Winston Logger, Static SPA │
│                                                                                 │
│  API Routes:                                                                    │
│  ├── /api/search           -> Article discovery & multi-category ranking        │
│  ├── /api/categories       -> Hierarchical taxonomy tree provider               │
│  ├── /api/reference-doc    -> File parsing, similarity, & prevalence search     │
│  ├── /api/export           -> Word, CSV, BibTeX, RIS, JSON generation           │
│  ├── /api/generate-detail-document -> AI-assisted regulatory study summaries   │
│  ├── /api/generate-short-summary   -> Tabular nonclinical overview with italics │
│  ├── /api/template*        -> Docx template parsing & XML content injection     │
│  └── /api/drug-stats       -> RxNorm cache statistics and cache management      │
│                                                                                 │
│  Internal Services & Core Engines:                                              │
│  ├── PubMedService        -> ESearch & EFetch with exponential backoff & cache  │
│  ├── FilterService        -> Relevance scoring, species exclusion, match chips │
│  ├── DrugSynonymService   -> Curated database + RxNorm API fallback             │
│  ├── RxNormService        -> NLM REST API integration with 24h caching         │
│  └── MeshService          -> MeSH descriptor & qualifier querying               │
└─────────────────────────────────────┬───────────────────────────────────────────┘
                                      │ External HTTPS APIs
                                      ▼
┌─────────────────────────────────┐   ┌──────────────────────────────────────────┐
│      NCBI PUBMED E-UTILITIES     │   │         NATIONAL LIBRARY OF MEDICINE     │
│   (esearch.fcgi, efetch.fcgi)   │   │             RxNorm REST API              │
└─────────────────────────────────┘   └──────────────────────────────────────────┘
```

---

## 4. Backend Deep Dive

### Services Layer

#### 1. `PubMedService` (`server/services/pubmedService.js`)
- Interfaces directly with NCBI E-utilities (`https://eutils.ncbi.nlm.nih.gov/entrez/eutils`).
- **Resilient Network Architecture**: Custom `retryWithBackoff` with exponential backoff (initial delay 1,000ms, doubling on retries; ignores 4xx client errors).
- **Query Optimization**: Constructs composite queries combining search terms, MeSH terms, category-specific keywords, study type MeSH constraints (`Animals[MeSH Terms]` vs `Humans[MeSH Terms]`), publication type exclusions (e.g. excluding reviews or meta-analyses from clinical trial searches), date ranges (`[dp]`), abstract filters (`hasabstract`), and full-text filters (`[sb]`).
- **Caching**: Local memory caching with `NodeCache` with configurable standard TTL (default: 3600s).
- **XML Parsing**: Utilizes `xml2js` with `explicitArray: false` to normalize complex Medline citations into uniform JavaScript article models.

#### 2. `RxNormService` (`server/services/rxNormService.js`)
- Integrates with the official NLM RxNav REST API (`https://rxnav.nlm.nih.gov/REST`).
- **Three-Stage Lookup**:
  1. Requests RxCUI Concept Unique Identifier via `/rxcui.json?name={drugName}`.
  2. Resolves related drugs and brand names via `/rxcui/{rxcui}/related.json?tty=SCD+SBD+GPCK+BPCK+IN+MIN+PIN+BN`.
  3. Fetches spelling suggestions via `/spellingsuggestions.json?name={drugName}`.
- **Cache Strategy**: In-memory `Map` with 24-hour expiration window.

#### 3. `DrugSynonymService` (`server/services/drugSynonymService.js`)
- Hybrid resolver providing sub-millisecond local lookups with asynchronous RxNorm fallback.
- Preloaded with 13 drug classes: *antibiotics, analgesics, cardiovascular, diabetes, antidepressants, respiratory, gastrointestinal, anticoagulants, antifungals, antivirals, immunosuppressants, hormones, and chemotherapy*.
- **Combination Drug Splitting**: Handles multi-ingredient drugs like Augmentin / Co-amoxiclav by detecting whether distinct components (e.g. amoxicillin and clavulanic acid) appear across different sentences of the abstract.
- Generates bidirectional synonym maps (e.g. searching generic maps to brand; brand maps to generic).

#### 4. `MeshService` (`server/services/meshService.js`)
- Queries NCBI MeSH database via `esearch.fcgi?db=mesh` and `esummary.fcgi?db=mesh`.
- Extracts primary MeSH descriptors and narrower qualifier terms.
- Fallback heuristic queries PubMed for articles matching the phrase with `[Title/Abstract]` and parses resulting `<MeshHeading>` tags directly.

#### 5. `FilterService` (`server/services/filterService.js`)
- Implements the complete mathematical scoring model, study species verification, and match tracking.
- Traverses the 82KB hierarchical taxonomy in `server/data/keywordMappings.json`.

---

### Hierarchical Category Taxonomies

#### Animal Studies Hierarchy
```
Animal Studies
├── Pharmacodynamics
│   ├── Primary (in vitro, in vivo, organ systems, binding affinity, ED50, EC50)
│   └── Secondary (off-target activity, selectivity ratio, secondary receptors)
├── Safety Pharmacology
│   ├── Central Nervous System (CNS) (locomotor, Irwin test, EEG, behavioral)
│   ├── Cardiovascular System (CVS) (hERG, QT prolongation, blood pressure, ECG)
│   └── Other Safety Pharmacology (respiratory rate, tidal volume, gastrointestinal transit)
├── Pharmacokinetics
│   ├── Method of Analysis (LC-MS/MS, HPLC, assay validation, lower limit of quantitation)
│   ├── Absorption (bioavailability, Cmax, Tmax, AUC, intestinal permeability)
│   ├── Distribution (plasma protein binding, volume of distribution, tissue distribution, BBB)
│   ├── Metabolism (CYP450 isozymes, metabolic stability, metabolite identification)
│   ├── Excretion (urinary excretion, biliary elimination, fecal clearance)
│   ├── Special Populations / Disease Models (renal impairment model, hepatic failure model)
│   ├── Drug Interaction (CYP induction, CYP inhibition, P-gp substrate)
│   └── Other Pharmacokinetic Studies
└── Toxicology
    ├── Single-Dose Toxicity (acute toxicity, LD50, maximum tolerated dose - MTD)
    ├── Repeat-Dose Toxicity (subacute, subchronic, chronic, NOAEL, LOAEL)
    ├── Genotoxicity (Ames test, micronucleus assay, chromosomal aberration)
    ├── Carcinogenicity (2-year bioassay, transgenic rodent model)
    ├── Reproductive & Developmental Toxicity (fertility, teratogenicity, embryo-fetal)
    ├── Local Tolerance (dermal irritation, ocular tolerance, injection site reaction)
    └── Other Toxicity Studies (immunotoxicity, phototoxicity, dependence liability)
```

#### Human Studies Hierarchy
```
Human Studies
├── Pharmacokinetics
│   ├── Method of Analysis (validated bioanalytical methods, GLP compliance)
│   ├── Absorption (oral bioavailability, food effect, Cmax, Tmax, AUC0-inf)
│   ├── Distribution (volume of distribution Vd/F, free fraction, blood-plasma ratio)
│   ├── Metabolism (in vivo metabolites, major circulating metabolites, CYP phenotyping)
│   ├── Excretion (renal clearance, terminal elimination half-life t1/2, cumulative excretion)
│   ├── Special Populations (elderly, pediatric, renal impairment, hepatic cirrhosis)
│   └── Drug Interaction (clinical DDI, transporter-mediated interactions, AUC ratio)
├── Pharmacodynamics
│   ├── Proposed Indications (clinical endpoints, biomarker response, target engagement)
│   └── Drug Interaction (pharmacodynamic synergy, antagonism, additive toxicity)
├── Efficacy
│   ├── Placebo-Controlled Trials (blinded RCTs, superior efficacy vs placebo)
│   ├── Active-Controlled Trials (non-inferiority, equivalence vs standard of care)
│   ├── Uncontrolled Trials (open-label extensions, single-arm studies)
│   ├── Meta-Analyses & Systematic Reviews (PRISMA guidelines, pooled odds ratio)
│   ├── Paediatric Studies (pediatric investigation plan - PIP, dosing, safety)
│   └── Dosage & Administration (dose-ranging, regimen optimization, titration)
└── Safety
    ├── Adverse Reactions (treatment-emergent adverse events - TEAEs, incidence)
    ├── Warnings and Precautions (black box warnings, contraindications)
    ├── Pregnancy (teratogenic risk, registry data, trimester safety)
    ├── Breastfeeding / Lactation (milk-to-plasma ratio, infant exposure)
    ├── Fertility (effects on male/female reproductive capacity)
    ├── Overdose (toxic manifestations, reversal agents, hemodialysis efficacy)
    └── Post-Marketing Experience (pharmacovigilance, real-world evidence, PSUR)
```

---

### Scoring & Ranking Engine

The ranking algorithm in `filterService.js` balances precision, species validity, and technical depth:

$$\text{Final Score} = \text{Base Drug Score} + \text{Filter Match Score} + \text{MeSH Weight} + \text{Title Hierarchy Boost} + \text{Multi-Match Bonus}$$

#### 1. Base Drug Score
- Drug identified in **Title**: `+150 points`
- Drug identified in **Abstract**: `+80 points`
- Frequency multiplier: 
  - $\ge 5$ mentions: `+40 points`
  - $3-4$ mentions: `+25 points`
  - $2$ mentions: `+15 points`

#### 2. Title & Hierarchy Boosts
- **Drug + Full Hierarchy Path in Title** (e.g. *Drug* AND *Pharmacokinetics > Distribution*): `+500 points`
- **Drug in Title + Specific Subheading Keywords in Title**: `+800 points`
- **Specific Subheading Keywords in Title** (drug in abstract): `+500 points`
- **Drug + Parent Keyword Match**: `+250 points`

#### 3. Strict Rejection & Demotion Logic
- **Parent Keyword Polluting Subheading Search**: If a user selects a specific subheading (e.g. *Absorption*), articles matching only the parent heading (e.g. *Pharmacokinetics*) without subheading keywords are **strictly rejected (Score = 0)**.
- **Abstract-Only Demotion**: If technical keywords appear solely in the abstract without presence in the title, score is cut to $15\%$ or $5\%$ to keep title-level verified articles on top.
- **Study Species Exclusion**:
  - In *Human* mode: rejects articles with `"in rats"`, `"in mice"`, `"in rabbits"`, `"murine"`, or animal-only MeSH headings.
  - In *Animal* mode: rejects articles stating `"clinical trial"`, `"patients"`, or `"randomized controlled trial"`.

#### 4. Automated Study Design Detection
The engine inspects titles and MeSH classifications to tag articles with high or medium confidence study design types:
- *Randomized Controlled Trial (RCT)*
- *Double-Blind / Single-Blind / Placebo-Controlled Study*
- *Active-Controlled Study*
- *Cohort / Case-Control / Cross-Sectional Study*
- *Phase I, Phase II, Phase III, Phase IV Trials*
- *Systematic Review / Meta-Analysis*
- *Bioequivalence / Pharmacokinetic Study*

---

### Document Generation & Template Engines

The platform includes multiple document generation systems tailored for regulatory reporting:

```
Document Generation Systems
├── 1. Unified Word Export (/api/export/unified-word)
│   ├── Groups articles by Category & Study Type
│   ├── Auto-replaces terms with standard pharmaceutical abbreviations
│   ├── Prepends full Abbreviations Table
│   └── Renders bibliographies in APA, MLA, Chicago, Harvard, or Vancouver
│
├── 2. AI-Powered Detail Document (/api/generate-detail-document)
│   ├── Uses OpenAI GPT-3.5-turbo (with deterministic fallback)
│   ├── Formats each abstract into formal ICH Module 2.4/2.5 narrative
│   └── Structures headings (Heading 1 for main category, Heading 2 for subcategory)
│
├── 3. Tabular Short Summary Document (/api/generate-short-summary)
│   ├── Nonclinical overview summary tables
│   ├── Automatic italicization of species & Latin terms (e.g., C. elegans, in vitro)
│   └── Summary paragraphs linked directly to numbered reference tables
│
└── 4. Template Engines (v2, v3, v4, and Template Final)
    ├── Template Final (/api/template-final/generate): Raw Word XML parser
    │   ├── Dissects document.xml structure using PizZip
    │   ├── Identifies w:pStyle heading nodes and w:numPr list items
    │   └── Injects syntheses directly beneath matching headings
    └── Template V2/V4: Placeholder replacement ({pharmacology}, {toxicology}, etc.)
```

#### Standard Abbreviations Engine
Contains a built-in dictionary of over 40 pharmaceutical acronyms (*AUC, Cmax, Tmax, CLcr, LOAEL, NOAEL, MTD, ADME, CYP, hERG, GLP, etc.*). The export engine scans all abstracts, expands the first occurrence to `Full Term (ABBR)`, converts subsequent instances to `ABBR`, and generates a sorted 2-column table at the beginning of the Word document.

---

### Reference Document & Prevalence Analysis (EMA ERA)

Located in `server/routes/referenceDoc.js` (~2,000 lines):

#### Reference Document Upload
1. Accepts PDF (`pdf-parse`), DOCX (`mammoth`), or TXT file uploads up to 10MB.
2. Extracts key medical concepts, filtering standard stop words while applying a 5x boost to pharmacologic/toxicologic terms.
3. Computes bigrams and trigrams for technical phrase recognition.
4. Generates similarity scores against candidate PubMed articles.

#### EMA ERA Disease Prevalence Engine
Regulatory environmental risk assessments under the European Medicines Agency require calculating the market penetration factor ($F_{\text{pen}}$) based on European disease prevalence:
- **Geographic Filtering**: Allows queries constrained to All Europe, UK, England, Germany, France, Spain, Italy, or India.
- **Multi-Year Search**: Analyzes publication windows up to 5 concurrent years.
- **Dual-Column Presentation**:
  - **Column 1 (PREVALENCE)**: Epidemiological prevalence rates, patient registries, and cohort studies.
  - **Column 2 (ANOTHER)**: Supporting clinical data, dosing regimens, and drug pharmacology.
- **Authority Tiering**:
  - **Tier 1**: Global Burden of Disease (GBD), WHO, EMA official statistics.
  - **Tier 2**: Systematic reviews and Cochrane meta-analyses.
  - **Tier 3**: National health registries and government surveillance databases.
  - **Tier 4**: Peer-reviewed cohort or observational studies.

---

## 5. Frontend Deep Dive

### Component Hierarchy

```
<App>
├── <ToastContainer /> (Notifications)
├── <Header /> (Navigation & branding)
├── <main className="container">
│   ├── <SearchBar /> (Query input, live search trigger)
│   ├── <StudyTypeSelector /> (Animal vs Human switch & Strict Verification toggle)
│   ├── <SearchFilters /> (Year range, Abstract, Free Full Text, Full Text)
│   ├── <CategoryTree /> (Collapsible hierarchical tree with custom keyword inputs)
│   │
│   ├── {Results Mode}
│   │   ├── <ResultsDisplay />
│   │   │   └── <ArticleCard /> (Rank, relevance score, PMID, abstract expander,
│   │   │                       match validation chips, "Add to Cart" button)
│   │   ├── <ExportOptions /> (Instant CSV, JSON, BibTeX, RIS export)
│   │   └── <ExportModal /> (Single search Word export modal)
│   │
│   ├── {Reference & Prevalence Mode}
│   │   ├── <PrevalenceSearch /> (Disease prevalence search for EMA ERA)
│   │   ├── <ReferenceDocUpload /> (Drag-and-drop file upload for kindred article discovery)
│   │   └── {Dual Column Results Display} (PREVALENCE column vs ANOTHER column)
│   │
│   └── <SelectCart /> (Slide-over drawer managing saved articles)
│       ├── <UnifiedExportModal /> (Export grouped articles across categories to Word)
│       ├── <DetailDocumentModal /> (Generate AI Module 2.4/2.5 Word documents)
│       └── <TemplateDocModal /> (Upload custom template and inject findings)
```

### Cart Management & Storage Migration
- Implemented in `client/src/context/CartContext.js`.
- Synchronized to browser `localStorage` under `pubmedCart`.
- **Automatic Data Sanitization**: Includes `clearBadCartData.js` to inspect and migrate legacy nested XML-object PMIDs (`pmid: { _: '12345' }`) into sanitized string primitives (`"12345"`).

### Visual Match Verification Badges
Every `ArticleCard` renders an audit strip demonstrating why the article was ranked:
- 🐾 **Verified Animal Study** or 👨‍⚕️ **Verified Human Study** badge.
- ✅ **Drug Match**: Location (`TITLE` vs `ABSTRACT`), frequency count, and synonym matched.
- ✅ **Category / Heading Match**: Exact matched nomenclature and location.
- ✅ **Specific Terms**: Subheading inner keywords (e.g. *Cmax, bioavailability*).
- ✅ **Study Design**: Detected protocol classification (e.g. *Randomized Controlled Trial, High confidence*).

---

## 6. API Reference

### Search & Taxonomy Endpoints

#### `POST /api/search`
Execute intelligent search and ranking.
```json
{
  "query": "amoxicillin",
  "studyType": "animal",
  "categoryPath": "pharmacokinetics.absorption",
  "customKeywords": "Cmax, bioavailability",
  "maxResults": 200,
  "topN": 30,
  "yearFrom": 2018,
  "yearTo": 2026,
  "hasAbstract": true,
  "freeFullText": false,
  "fullText": false,
  "verifyAnimalStudy": true,
  "verifyHumanStudy": false
}
```

#### `POST /api/search/batch`
Batch execute search across multiple taxonomy paths.

#### `GET /api/categories`
Retrieve full category trees for both Animal and Human domains.

#### `GET /api/categories/:studyType`
Retrieve taxonomy for `:studyType` (`animal` or `human`).

#### `GET /api/categories/:studyType/:categoryPath/keywords`
Retrieve associated MeSH and text keywords for a category path.

#### `DELETE /api/search/cache`
Flush the PubMed in-memory search cache.

---

### Reference Document & Prevalence Endpoints

#### `POST /api/reference-doc/upload`
Upload a document (multipart/form-data) to extract terms and find similar literature.
- Parameters: `document` (file), `studyType` (`animal`|`human`), `drugName`, `doseForm`, `indication`, `includeSubheadings`.

#### `POST /api/reference-doc/prevalence-search`
Epidemiological search without file upload.
```json
{
  "studyType": "human",
  "prevalenceCountry": "Germany",
  "prevalenceYears": "[\"2024\", \"2023\", \"2022\"]",
  "prevalenceDiseaseName": "Type 2 Diabetes",
  "drugName": "Metformin",
  "includeSubheadings": true
}
```

#### `POST /api/reference-doc/fetch-abstracts`
Batch fetch abstracts for an array of PMIDs: `{"pmids": ["12345678", "87654321"]}`.

---

### Export & Document Synthesis Endpoints

#### `POST /api/export/unified-word`
Export cart contents grouped by category into a formatted `.docx` file with an abbreviations table and formatted references.

#### `POST /api/export/word`
Export a flat list of articles into a customized Word document.

#### `POST /api/export/:format`
Export results as `:format` (`csv`, `bibtex`, `ris`, or `json`).

#### `POST /api/generate-detail-document`
Generate an AI-narrated regulatory document from cart items.

#### `POST /api/generate-short-summary`
Generate a tabular nonclinical overview document with Latin/species term italicization.

#### `POST /api/template-final/upload` & `POST /api/template-final/generate`
Upload an existing `.docx` template and generate an XML-injected document maintaining template styling.

---

### Diagnostic Endpoints

#### `GET /api/health`
Returns system uptime and health status.

#### `GET /api/drug-stats`
Returns RxNorm API hit rate, cache size, and local database statistics.

#### `POST /api/drug-stats/clear-cache`
Clears the RxNorm in-memory cache.

---

## 7. Data Stores & Dictionaries

- `server/data/keywordMappings.json` (82 KB): Contains full hierarchical mapping of animal and human categories, specific MeSH headings (`[MeSH]`), text keywords (`[tiab]`), and publication types (`[Publication Type]`).
- `server/data/drugSynonyms.json` (7 KB): Comprehensive synonym catalog indexing brand and generic equivalents across 13 major drug classes.

---

## 8. Installation & Local Setup

### Prerequisites
- **Node.js**: v18.x or v20.x
- **npm**: v9.x or higher
- **NCBI API Key** (optional, but recommended to raise NCBI rate limits from 3 to 10 req/s)
- **OpenAI API Key** (required for AI-powered narrative document generation)

### Setup Steps

1. **Clone the repository**:
   ```bash
   git clone <repository-url>
   cd pubmed-app
   ```

2. **Install all dependencies (root, server, and client)**:
   ```bash
   npm run install-all
   ```

3. **Configure Environment Variables**:
   Create a `.env` file in the root directory:
   ```env
   PORT=5000
   PUBMED_API_BASE_URL=https://eutils.ncbi.nlm.nih.gov/entrez/eutils
   PUBMED_API_KEY=your_ncbi_api_key_here
   RATE_LIMIT_WINDOW_MS=60000
   RATE_LIMIT_MAX_REQUESTS=100
   CACHE_TTL_SECONDS=3600
   OPENAI_API_KEY=your_openai_api_key_here
   ```

4. **Launch Development Servers**:
   ```bash
   npm run dev
   ```
   - Express API will run at: `http://localhost:5000`
   - React application will run at: `http://localhost:3000`

---

## 9. Deployment Guide

The repository includes pre-configured deployment manifests:

### 1. Render (`render.yaml`)
- Configured as a unified Node.js web service.
- Builds React client into static files and serves them directly through Express.
- Build command:
  ```bash
  npm install && cd server && npm install && cd ../client && npm install && CI=false npm run build
  ```
- Start command:
  ```bash
  NODE_ENV=production node server/index.js
  ```

### 2. Railway (`railway.json` & `nixpacks.toml`)
- Nixpacks configuration pins `nodejs-18_x`.
- Executes automated build and boots production server on port assigned by Railway.

### 3. Netlify (`netlify.toml`)
- Configured for standalone frontend builds with single-page app rewrite redirects (`/*` → `/index.html`).

---

## 10. Testing & Quality Assurance

### Automated API Test Suite
Run the backend test script to verify endpoint connectivity, categories loading, and live PubMed queries:
```bash
node server/test-search-endpoint.js
```

### Manual Testing Scenarios
1. **Drug Synonym Verification**: Search `"Augmentin"` under *Animal Studies > Pharmacokinetics > Absorption*. Confirm results include articles indexed under *"amoxicillin-clavulanate"*.
2. **Species Isolation**: Verify that checking `"Strict Animal Study Only"` eliminates human clinical trial records.
3. **Cart & Word Generation**: Add 3 articles across distinct categories to cart, open Select Cart, click *Export Word Document*, and confirm the generated `.docx` contains the abbreviations table, category sections, and citation index.
4. **Prevalence Verification**: Run a prevalence query for *"Multiple Sclerosis"* in *"France"*. Verify dual-column separation and authority tier ratings.

---

## 11. Troubleshooting & FAQ

- **PubMed API 429 Too Many Requests**: NCBI limits unauthenticated requests to 3 per second. Set a valid `PUBMED_API_KEY` in `.env` to increase limit to 10 req/s.
- **OpenAI Summaries Fallback**: If `OPENAI_API_KEY` is omitted, the detail document generator automatically switches to deterministic rule-based formatting without failing.
- **Cart PMID format errors**: If legacy data exists in browser local storage, clear the browser cache or call `clearBadCartData()` to reset `pubmedCart`.

---

## 12. License

This project is licensed under the MIT License.
