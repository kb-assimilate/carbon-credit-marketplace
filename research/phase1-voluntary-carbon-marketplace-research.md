# **Voluntary Carbon Market Mechanics, Registry Infrastructure, and Regulatory Architecture**

## **1\. Market Structure and Stakeholder Lifecycle**

### **Full Lifecycle of a Voluntary Carbon Credit**

The lifecycle of a voluntary carbon credit spans a multi-stage process from physical activity design to permanent cancellation. This lifecycle transforms an environmental intervention into a standardized, tradable financial asset representing one metric tonne of carbon dioxide equivalent (![][image1]) reduced or removed from the atmosphere1.  
The lifecycle begins with project design and methodology selection. A project developer (proponent) identifies an emission reduction or carbon removal activity, such as afforestation, clean cookstove distribution, biochar production, or direct air capture2. The developer selects an approved quantification methodology established by a standard-setting registry1. This methodology dictates how the baseline scenario (emissions without the project) is established, how additionality is demonstrated, how leakage is accounted for, and how monitoring must be conducted4. The developer compiles these parameters into a Project Design Document (PDD).  
Following project design, the project undergoes ex-ante validation. An independent, accredited Validation and Verification Body (VVB) evaluates the PDD1. The VVB assesses whether the project design satisfies all rules of the chosen registry and methodology, including land eligibility, financial additionality, stakeholder consultations, and safeguard compliance1. This phase includes a mandatory public comment period hosted on the registry portal1.  
Upon receiving a positive validation report from the VVB, the project developer submits the documentation package to the target registry for pipeline listing and registration1. The registry reviews the package for completeness and adherence to program rules8. Once approved, the project is formally listed as "Registered" on the registry database, assigning it a unique project identification number9.  
Once registered, the project proponent initiates project implementation and monitoring. The developer executes the project activity and collects empirical data in accordance with the monitoring plan outlined in the methodology1. Monitoring periods typically range from six months to several years depending on the project category and registry rules2. The developer synthesizes the collected field and remote sensing data into a formal Monitoring Report7.  
The project then enters the verification phase. The developer retains an accredited VVB (often distinct from the validating body) to perform an ex-post audit of the Monitoring Report4. The VVB conducts site visits, audits sensor or survey data, verifies calculation logic, and ensures no double-counting has occurred1. Upon successful audit, the VVB issues a Verification Report and a formal Verification Statement confirming the exact volume of greenhouse gas (GHG) reductions or removals achieved4.  
Next, the project proponent requests credit issuance and serialization. The verification package is submitted to the registry administrator7. Upon payment of applicable issuance fees, the registry mints carbon credits into the developer’s registry account1. Each credit is assigned a unique, immutable serial number encoding the registry code, project ID, country, project type, vintage year, and specific block position9. For land-use or non-permanent project types, a designated percentage of issued credits is automatically deducted and transferred into the registry’s non-permanence buffer pool to mitigate reversal risks4.  
Once issued, credits enter secondary market trading and settlement. Credits can be transferred between registry account holders through over-the-counter (OTC) bilateral transactions, brokered deals, or centralized spot exchanges1. Ownership changes are recorded on the registry's central ledger1.  
The lifecycle concludes with credit retirement and claim attribution. The ultimate end-use of a carbon credit occurs when an account holder permanently cancels (retires) the credit on the registry ledger on behalf of itself or a third-party beneficiary7. Retirement removes the specific serial numbers from active circulation, generating a public Retirement Certificate4. This prevents double-claiming and allows the corporate or retail entity to claim the underlying environmental outcome against its carbon footprint4.

### **Stakeholder Taxonomy and Mapping**

The voluntary carbon market relies on a specialized network of market participants. The following matrix details the functions, revenue models, and governance oversight across all primary stakeholder groups:

| Stakeholder Category | Primary Functions | Operating and Revenue Model | Regulatory and Governance Oversight |
| :---- | :---- | :---- | :---- |
| **Project Developers / Proponents** | Originates projects, secures land/feedstock rights, manages field operations, collects MRV data, and bears development capital expenditure1. | Revenue generated through credit sales on primary markets or long-term off-take contracts1. | Bound by host-country laws, registry rules, host-nation Article 6 requirements, and environmental safeguards1. |
| **Validation & Verification Bodies (VVBs)** | Independent third-party auditors that conduct ex-ante validation of project design and ex-post verification of monitored reductions/removals1. | Fee-for-service model paid by project developers (or by registry under buyer-pays models)10. | Accredited by ISO 14065, ANSI National Accreditation Board (ANAB), or designated registry oversight bodies5. |
| **Standard-Setters / Registries** | Establishes scientific methodologies, maintains central ownership ledgers, supervises VVB accreditation, and issues serialized credits1. | Account registration fees, project listing fees, issuance fees per credit, and transfer/retirement fees1. | Governed by independent boards, scientific advisory panels, ICVCM Core Carbon Principles, and CORSIA framework rules5. |
| **Brokers and Intermediaries** | Facilitates bilateral OTC market transactions, provides liquidity, structures forward purchase agreements, and manages portfolio risk12. | Commission spreads, trading margins, and structured deal advisory fees. | Subject to regional financial conduct authorities, anti-money laundering (AML), and know-your-customer (KYC) frameworks. |
| **Spot Exchanges & Platforms** | Operates central limit order books (CLOB) or execution venues for standardized carbon contracts and spot physical delivery12. | Trading execution fees, clearing fees, and market data licensing fees. | Regulated financial exchanges or alternative trading systems (ATS) governed by market conduct authorities. |
| **Corporate Buyers** | Purchases and retires carbon credits to meet voluntary climate commitments, CSRD BVCM disclosures, or compliance obligations4. | Capital outlay classified as operational expenditure or sustainability investment4. | Regulated by green claims directives (e.g., EU Empowering Consumers Directive), SEC climate disclosure rules, and VCMI Claims Code4. |
| **Retail Buyers / Aggregators** | Individuals or small businesses purchasing fractional or micro-volumes of credits via web platforms or integrated APIs12. | Retail price markups embedded in consumer checkout flows. | Subject to local consumer protection laws, false advertising regulations, and fair trading standards. |

## 

## **2\. Deep Dive Analysis of Carbon Registries**

### **Verra (Verified Carbon Standard \- VCS)**

#### **API Availability and Technical Architecture**

Verra maintains a centralized registry infrastructure tracking over 2,300 projects4. Programmatic integration options have historically been fragmented. Verra provides access through the "Verra Project Hub" and its associated "Digital Gateway API"8. These APIs allow registered project proponents and partners to exchange project documentation, workflow statuses, and submission metadata8.  
For external market data ingestion, market participants often rely on parsing or utilizing third-party unauthenticated REST endpoints exposed by its underlying web platform9. There is no fully open, unauthenticated public REST API that permits direct automated credit transfer or execution of retirement operations without authenticated account credentials and manual registry portal authorization6.

#### **Authentication Model, Rate Limits, and Data Formats**

Authentication for the official Verra Digital Gateway API utilizes OAuth 2.0 bearer tokens combined with API subscription keys issued via the Verra Developer Portal8. Rate limits are governed by API tier allocations and are managed dynamically; explicit public documentation on exact call-per-minute limits is restricted to account-holders. Data payloads are formatted in standard JSON for workflow metadata, while project verification documents are delivered as PDF assets8.

#### **Methodology Approval Process**

Methodology development under VCS requires a multi-step process including public consultation, dual VVB assessment, and internal Verra review1. The approval timeline historically ranges from 18 to 36 months. To streamline this, Verra has introduced "Digitalized Methodologies" directly within the Project Hub, embedding calculation engines to compute GHG emission reductions or removals (![][image2]) automatically8.

#### **Anti-Double-Counting and Serialization**

Verra assigns every Verified Carbon Unit (VCU) a unique 18-digit numeric serial number4. The serial number specifies the vintage year, project ID, project type, host country, and batch positioning9. Verra enforces strict rules preventing double-issuance across overlapping project boundaries and mandates formal cancellation certificates when converting credits to other programs1.

#### **Market Share and Issuance Volume**

Verra remains the largest voluntary carbon standard globally4. As of 2024, Verra accounted for approximately 63% of total cumulative voluntary carbon market retirements, with over 2,300 registered projects and more than 1.3 billion cumulative VCUs issued4.

#### **Credibility Controversies and Risk Factors**

Verra has faced scrutiny regarding historical REDD+ (Reducing Emissions from Deforestation and Forest Degradation) avoidance methodologies (such as VM0006 and VM0007), where independent academic studies alleged that baseline deforestation risks were over-estimated, leading to over-crediting4. In response, Verra launched a consolidated REDD+ methodology (VM0048) and transitioned baseline mapping responsibilities to direct jurisdictional data providers using standard spatial risk maps (VMD0055)8. Platforms building on VCS data must differentiate between legacy REDD+ vintages and new VM0048 credits8.

### **Gold Standard (GS)**

#### **API Availability and Technical Architecture**

Gold Standard operates a public registry covering approximately 4,000 listed projects9. Gold Standard provides public REST API endpoints allowing users to query project details, credit issuance records, and retirement events without requiring authentication for basic GET queries9. The API exposes structured entities including project IDs, country codes, Sustainable Development Goal (SDG) impacts, methodology versions, and credit statuses9.

#### **Authentication Model, Rate Limits, and Data Formats**

Public GET endpoints require standard HTTP request headers, while write endpoints and account management operations require OAuth 2.0 or bearer token authentication issued to registered registry users9. API responses are delivered in JSON format. Rate limits are throttled at the infrastructure layer to prevent automated scraping overloads, though specific numeric limits are not publicly published.

#### **Methodology Approval Process**

Gold Standard methodologies emphasize sustainable development and community co-benefits9. Methodologies undergo scientific review by the Gold Standard Technical Advisory Committee (TAC) alongside mandatory public stakeholder consultations. The approval timeline typically takes between 12 and 24 months.

#### **Anti-Double-Counting and Serialization**

Gold Standard issues Gold Standard Verified Emission Reductions (GS-VERs). Each credit receives a unique serial number string encoding the project identifier, country, vintage, and credit type. Gold Standard maintains direct registry connections with national registries to avoid double-claiming under Article 6 and compliance regimes.

#### **Market Share and Issuance Volume**

Gold Standard holds the second-largest share of listed projects in the traditional voluntary market, with over 4,000 registered activities9. It accounts for a substantial volume of credits in the clean cooking, household energy efficiency, and renewable energy sectors9.

#### **Credibility Controversies and Risk Factors**

Gold Standard faced scrutiny surrounding historic clean cookstove methodologies (e.g., TPDDTEC and AMS-II.G) regarding the "fraction of non-renewable biomass" (![][image3]) assumptions21. Studies indicated that default ![][image3] factors overstated actual deforestation impact. Gold Standard subsequently updated its cookstove methodologies (requiring metered usage or revised baseline calculations) to align with ICVCM requirements21.

### **American Carbon Registry (ACR)**

#### **API Availability and Technical Architecture**

ACR (formerly American Carbon Registry, an enterprise of Winrock International) operates a centralized registry primarily serving North American voluntary and compliance markets23. Programmatic API access for public automated data extraction is limited. Data access is primarily delivered via a web portal interface and downloadable periodic database exports23. Public programmatic access requires direct enterprise data-sharing agreements or custom technical integrations.

#### **Authentication Model, Rate Limits, and Data Formats**

Public web-portal interactions use standard session-based web security. Private programmatic connections utilize API key authentication over HTTPS. Returned data payloads follow standard JSON/CSV schema conventions for registry exports. Public documentation on rate limits is unavailable.

#### **Methodology Approval Process**

ACR methodologies undergo scientific peer-review processes administered by internal scientific committees and external domain experts. The approval process generally requires 12 to 18 months.

#### **Anti-Double-Counting and Serialization**

ACR issues serialized emission reduction units, tagging each credit with vintage, project origin, sectoral scope, and state/region tracking identifiers11. ACR maintains strict rules prohibiting double-registration with compliance programs like the California Air Resources Board (CARB) cap-and-trade system23.

#### **Market Share and Issuance Volume**

ACR is a major registry in North America, maintaining high volume in Improved Forest Management (IFM), Ozone Depleting Substances (ODS) destruction, and industrial gas capture21.

#### **Credibility Controversies and Risk Factors**

ACR’s Improved Forest Management (IFM) methodologies have been analyzed by academic institutions regarding baseline setting and harvest delay assumptions21. Platforms using ACR data must monitor methodology versions (e.g., IFM v2.0 vs legacy versions) to verify alignment with evolving ICVCM additionality criteria21.

### **Climate Action Reserve (CAR)**

#### **API Availability and Technical Architecture**

Climate Action Reserve focuses on carbon offset projects across North America and Latin America22. Programmatic API availability for public automated ingestion is not published as an open self-service API. External market participants obtain data via periodic system exports, Berkeley Carbon Trading Project database compilations, or custom account portal integrations23.

#### **Authentication Model, Rate Limits, and Data Formats**

CAR account access relies on standard web portal credentialing (username/password with multi-factor authentication). Automated programmatic data parameters, rate limits, and endpoint documentation are not publicly available without an enterprise account agreement.

#### **Methodology Approval Process**

CAR protocols are developed via multi-stakeholder workgroups and public review cycles. CAR's board formally adopts protocols, a process that typically takes 12 to 18 months.

#### **Anti-Double-Counting and Serialization**

CAR issues Climate Reserve Tonnes (CRTs). Each CRT carries a unique serial number specifying the project type, country, state, vintage, and serial sequence. CAR tracks unit transfers to prevent double-selling across regulatory boundaries11.

#### **Market Share and Issuance Volume**

CAR holds a dominant market position in U.S. livestock methane, landfill gas, ODS destruction, and Mexican forestry protocols (e.g., CAR Mexico Forest Protocol)21.

#### **Credibility Controversies and Risk Factors**

CAR's U.S. forestry protocols faced academic debate concerning non-permanence buffer pool allocations relative to expanding wildfire risks in Western U.S. forests. CAR updated its buffer pool risk calculation tools to adjust for climate-driven disturbance dynamics.

### **Puro.earth**

#### **API Availability and Technical Architecture**

Puro.earth, majority-owned by Nasdaq, is an enterprise registry built specifically for durable Carbon Dioxide Removal (CDR)24. Puro.earth provides dedicated API infrastructure, specifically the "Puro Registry API" and the "MyPuro API"24. These interfaces allow programmatic retrieval of project data, production facility details, Carbon Removal Certificates (CORCs), and real-time retirement statuses24.

#### **Authentication Model, Rate Limits, and Data Formats**

Puro.earth APIs utilize standard API key or OAuth 2.0 authentication models integrated into the Nasdaq market data framework24. Responses are formatted in structured JSON. Rate limits and endpoint call caps follow standard enterprise API tier structures managed via the enterprise developer portal24.

#### **Methodology Approval Process**

Puro.earth develops CDR methodologies (e.g., biochar, enhanced rock weathering, geologically stored carbon, ocean-based CDR) through an independent advisory board and scientific peer-review process21. The approval cycle ranges from 6 to 12 months, benefiting from a narrower focus on engineered removal pathways21.

#### **Anti-Double-Counting and Serialization**

Puro.earth issues serialized Carbon Removal Certificates (CORCs), where one CORC represents one metric tonne of net ![][image4] removed and stored24. The registry assigns individual serial numbers linked directly to verified production batches, facility identifiers, and lifecycle assessment (LCA) data points24.

#### **Market Share and Issuance Volume**

Puro.earth is a prominent certifier in the biochar market and commercial engineered carbon removals, with price indices published in collaboration with Nasdaq21.

#### **Credibility Controversies and Risk Factors**

Mainstream controversies remain low due to its exclusion of avoided-emission projects2. Risks are primarily technical, involving long-term durability verification for terrestrial biomass storage and biomass feedstock sourcing constraints21.

### **Isometric**

#### **API Availability and Technical Architecture**

Isometric is an AI-native, science-led carbon removal registry founded in 2022 and headquartered in London and New York2. Isometric operates dedicated, versioned REST APIs (/v0) designed for programmatic integration12. The architecture consists of two primary endpoints:

> 1. **Registry API** (api.isometric.com/registry/v0): Exposes public records, project data, issuances, credit batches, order fulfillment, transfers, buffer pools, and retirement actions12.  
> 2. **Certify MRV Ingestion API** (api.isometric.com/mrv/v0): Automates the ingestion of continuous MRV sensor data, feedstock logs, LCA calculations, and GHG statements12.

Isometric also deploys a hosted Model Context Protocol (MCP) server (api.isometric.com/mcp), enabling artificial intelligence agents to query protocols, modules, and platform documentation directly12. Data flow transitions seamlessly from on-field IoT sensors into the Certify MRV API, where machine learning algorithms and VVB auditors process removals before issuing credits accessible via the Registry API2.

#### **Authentication Model, Rate Limits, and Data Formats**

Isometric enforces a two-credential authentication model requiring an X-Client-Secret header paired with an organization-scoped Bearer JWT access token12. Data responses use strict ISO 8601 UTC timestamps, ISO 3166-1 Alpha-3 country codes, and Relay cursor-based pagination formats12. OpenAPI specifications and Postman collections are publicly available12.

#### **Methodology Approval Process**

Isometric protocols are drafted by internal scientists in collaboration with project developers and undergo peer-review by the "Science Network"—an independent group of over 200 academic climate scientists—followed by a public consultation period11. The process takes approximately 6 to 9 months11.

#### **Anti-Double-Counting and Serialization**

Isometric issues fully verified, ex-post delivered credits representing 1 metric tonne of net carbon dioxide removed with a minimum required permanence threshold (e.g., 1,000-year permanence for engineered pathways)11. Each credit is serialized and directly linked to all underlying raw verification data, fuel receipts, and sensor data logs publicly viewable on the registry2.

#### **Market Share and Issuance Volume**

As of early 2026, Isometric has issued approximately 157,572 credits and recorded 41,568 retirements14. While issued volumes are lower than legacy registries, Isometric holds a high share of contracted forward volume for high-durability carbon removals12.

#### **Credibility Controversies and Risk Factors**

Isometric addresses industry conflict-of-interest risks by shifting fees away from developers: registry and verification fees are charged directly to buyers rather than project proponents10. The primary risk is structural supply scarcity, as stringently verified durable CDR credits remain supply-constrained14.

### **Registry Comparison Matrix**

| Registry | Public API Availability | Authentication Model | Primary Serialization / Unit Code | Market Share / Issuance Volume | Core Focus & Protocols |
| :---- | :---- | :---- | :---- | :---- | :---- |
| **Verra (VCS)** | Partial / Digital Gateway API & Portal Scrapes8 | OAuth 2.0 / API Keys8 | Serialized VCU (18-Digit Numeric String)4 | \~63% of retirements; \>1.3B VCUs issued4 | AFOLU, REDD+, Renewable Energy, Cookstoves, Industrial4 |
| **Gold Standard** | Public REST API Available9 | Unauthenticated GET / OAuth 2.0 for write operations9 | Serialized GS-VER | \~4,000 registered projects9 | Clean Cooking, Household Energy, Community SDGs9 |
| **ACR** | Web Portal Export / Restrictive Enterprise API | API Key / Session-based | Serialized ACR Credit Unit11 | Major North American Market Share23 | Improved Forest Management, ODS Destruction, Reclaimed HFCs21 |
| **CAR** | Web Portal Export / Custom Account Integration | Session Auth / Enterprise Credentials | Serialized CRT (Climate Reserve Tonne)11 | Leading U.S. & Mexico Forestry / Methane Share22 | Livestock Methane, Landfill Gas, CAR Mexico Forestry21 |
| **Puro.earth** | Public Registry API & MyPuro API24 | API Key / OAuth 2.0 (Nasdaq Infra)24 | Serialized CORC (Carbon Removal Certificate)24 | Dominant in Biochar & Physical CDR21 | Biochar, Enhanced Rock Weathering, Geologically Stored Carbon21 |
| **Isometric** | Fully Open REST APIs (/v0 Registry & MRV) \+ MCP Server12 | X-Client-Secret \+ Bearer JWT12 | Serialized High-Durability Removal Certificate12 | \~157.5k Issued / \~41.5k Retired; High Forward Contracting14 | 1,000-Yr Durability CDR: Biochar, DAC, Bio-oil, Subsurface Storage11 |

## **3\. Digital Measurement, Reporting, and Verification (D-MRV) Landscape**

### **Geospatial and Satellite Infrastructure**

D-MRV systems replace periodic manual sampling with continuous automated data collection pipelines. Satellite remote sensing serves as the primary macro-layer for spatial baseline setting, canopy disturbance tracking, and biomass change modeling.  
Data ingestion flows sequentially from raw orbital acquisition through automated transformation layers. Optical satellite feeds (such as Sentinel-2 and PlanetScope), Synthetic Aperture Radar (SAR), and LiDAR data are ingested via API connectors. These raw feeds pass into cloud processing engines for atmospheric correction, cloud masking, and spectral unmixing. Machine learning algorithms process these spatial layers to output structural biomass estimations, canopy water metrics, and land-use change maps, which feed directly into automated D-MRV quantification protocols.

#### **Satellite Constellations and Sensors**

##### **Copernicus Sentinel-2 (ESA)**

Provides multispectral optical imaging at 10-meter, 20-meter, and 60-meter spatial resolutions with a 5-day revisit cycle. Sentinel-2 exposes surface reflectance bands crucial for computing Vegetation Indices, including normalized difference vegetation index (NDVI), enhanced vegetation index (EVI), and canopy water content.

##### **Planet Labs (PlanetScope & SkySat)**

PlanetScope delivers daily global 3-meter spatial resolution optical imagery, enabling near-real-time detection of forest degradation and illegal logging. SkySat offers sub-meter targeted tasking for high-resolution audit verification.

##### **Copernicus Sentinel-1 (ESA)**

Synthetic Aperture Radar (SAR) operating in the C-band. Unlike optical sensors, SAR penetrates cloud cover and forest canopies, providing backscatter intensity data directly correlated with structural above-ground biomass (![][image5]) and surface soil moisture levels.

##### **NASA ICESat-2 (ATLAS) and GEDI (ISS)**

Spaceborne LiDAR instruments providing high-precision vertical canopy profiles and tree height measurements, enabling calibration of regional biomass quantification models.

### **Ground Sensor and IoT Integration**

Ground-based sensing infrastructure bridges macro-level satellite observations with physical micro-data:

> * **Forestry & Agroforestry**: Automated tree-mounted dendrometers transmit sub-millimeter trunk growth data via LoRaWAN or cellular IoT, providing continuous growth rates that calibrate satellite biomass models.  
> * **Soil Carbon & Agriculture**: In-situ soil probe networks measure moisture, temperature, electrical conductivity, and field-level soil organic carbon (![][image6]) dynamics. Eddy covariance flux towers measure micro-meteorological fluxes of ![][image4] and methane (![][image7]) exchange directly above the canopy or soil surface.  
> * **Clean Cookstoves**: Thermal logger IoT devices attached to cookstoves record stove operational hours, firing temperatures, and usage frequencies. Data is periodically uploaded via mobile networks to verify usage claims without relying solely on manual survey questionnaires.  
> * **Engineered CDR (Biochar, DAC, Mineralization)**: Automated industrial sensors (flow meters, mass spectrometers, thermocouple chains, scale weights) feed continuous operational metrics into APIs like Isometric Certify, recording energy consumption, mass balance, and carbon concentration12.

### **D-MRV Maturity Matrix across Project Types**

| Project Category | D-MRV Maturity Level | Dominant Sensing Technologies | Residual Manual Dependencies | Key Technical & Integrity Risks |
| :---- | :---- | :---- | :---- | :---- |
| **Forestry (ARR & REDD+)** | **High** | Sentinel-2, PlanetScope, GEDI LiDAR, Synthetic Aperture Radar (SAR). | Physical ground-truthing plot reviews, legal title verification. | Optical cloud cover interference, saturation of radar signal in dense multi-canopy forests22. |
| **Biochar & Industrial CDR** | **Very High** | Industrial IoT sensors, automated mass balances, thermal sensors, digital telemetry12. | Periodic physical sampling for chemical carbon fraction (![][image8] ratio) analysis2. | Calibration drift of physical telemetry devices, feedstock tracking chain-of-custody gaps12. |
| **Clean Cookstoves** | **Medium-High** | Thermal IoT sensors, mobile survey apps, cellular telemetry loggers. | Physical stove inspection, replacement tracking, local fuel gathering surveys. | Sensor disconnection, thermal logger tampering, non-representative sampling deployments. |
| **Soil Organic Carbon (SOC)** | **Medium-Low** | Satellite spectroradiometry, eddy flux towers, soil IoT probe arrays, biogeochemical models (e.g., DayCent). | Physical soil core extraction, wet chemistry laboratory testing (dry combustion). | High spatial variability of SOC, high noise-to-signal ratios in satellite spectral soil reflections. |
| **Ocean Alkalinity / Marine CDR** | **Low (Emerging)** | Autonomous ocean gliders, biogeochemical Argo floats, inline ![][image9]/p$\\text{CO}\_2$ sensors. | Manual vessel-based water sampling, physical laboratory titration. | Complex oceanic dilution modeling, difficulty establishing precise non-project boundary control baselines21. |

## 

## **4\. Regulatory, Governance, and Compliance Environment**

### **ICVCM Core Carbon Principles (CCPs)**

The Integrity Council for the Voluntary Carbon Market (ICVCM) establishes a global quality benchmark for carbon credits via its Core Carbon Principles (CCPs)3. The ICVCM operates a "two-tick" governance framework15. First, the standard-setting registry must achieve program-level eligibility by meeting ICVCM governance, tracking, transparency, and auditing rules5. Second, individual project methodologies must independently earn methodology-level approval by passing technical evaluations for additionality, robust quantification, permanence, and social/environmental safeguards3. Only credits that satisfy both levels receive the official CCP label15.  
As of early 2026, eight major crediting programs have achieved CCP-Eligible status, including Verra, Gold Standard, ACR, CAR, Puro.earth, and Isometric15. Over 100 million issued credits carry or are eligible for the CCP label15.  
Key CCP-Approved methodologies include3:

> * **Afforestation, Reforestation, and Revegetation (ARR)**: Verra VM0047 (v1.0 and v1.1)22.  
> * **Engineered Carbon Dioxide Removal (CDR)**: Isometric Direct Air Capture (v1.1), Biomass Geological Storage (v1.0-1.1), Bio-oil Geological Storage (v1.0-1.1), Subsurface Biomass Storage (v1.0), and Biogenic CCS (v1.1)21; Gold Standard Accelerated Carbonation of Concrete Aggregate (v1.0)22.  
> * **Biochar**: Isometric Biochar Protocol (v1.1-1.2), Puro.earth Biochar Methodologies (2022/2025)2.  
> * **Improved Forest Management (IFM)**: CAR Mexico Forest Protocol (v3.0)22.  
> * **Energy & Industrial**: Selected clean cookstove protocols (e.g., Gold Standard metered/TPDDTEC and VCS VM0050 under revised parameters), ozone-depleting substance (ODS) destruction, and landfill gas capture3.

Methodologies undergoing active assessment or revision include legacy REDD+ avoidance protocols and unmetered soil carbon pathways21.

### **VCMI Claims Code of Practice**

The Voluntary Carbon Markets Integrity Initiative (VCMI) regulates corporate demand-side claims15. VCMI provides a framework to ensure corporate climate claims are credible and transparent:

#### **Carbon Integrity Claims**

Allows companies to make public claims ("Silver", "Gold", or "Platinum") based on retiring high-quality credits (specifically requiring CCP-approved credits) to cover residual emissions after demonstrating progress toward internal scope 1, 2, and 3 reduction targets15.

#### **Scope 3 Action Code / Scope 3 Flexibility Claim**

Addresses corporate challenges in meeting value-chain emissions targets16. It permits companies to temporarily bridge a documented Scope 3 target gap using high-quality carbon credits16. The flexibility mechanism caps the volume of credits usable for bridging at a maximum of 25% of the company's total Scope 3 trajectory, requiring simultaneous internal decarbonization investments26.  
Marketplaces must support automated reporting structures that link credit retirements directly to corporate VCMI claim declarations and audit trails4.

### **CORSIA Eligibility Framework**

The Carbon Offsetting and Reduction Scheme for International Aviation (CORSIA), established by the International Civil Aviation Organization (ICAO), mandates that global airlines offset emissions exceeding designated baseline thresholds4.  
During the CORSIA First Phase (2024–2026), eligible credits must fulfill strict criteria4:

> * **Host Nation Authorization (Article 6.2)**: The host country where the carbon reduction project is located must issue a formal Letter of Authorization (LOA)7.  
> * **Corresponding Adjustments (CA)**: The host nation commits to making a corresponding adjustment in its national greenhouse gas inventory under the Paris Agreement, preventing the emission reduction from being counted toward its national target (![][image10]) while simultaneously sold to an airline7.  
> * **Program Accreditation**: Registries including Isometric, ACR, CAR, Gold Standard, and Verra maintain conditional or full CORSIA approval statuses4.

Marketplace platforms must store Article 6 authorization documentation and expose metadata flags identifying whether a credit carries a CORSIA-eligible CA-authorized label7.

### **Data Protection and Compliance Framework**

A carbon marketplace platform handling farmer data, landholding records, and corporate KYC information must comply with international data privacy legal regimes.

#### **EU General Data Protection Regulation (GDPR)**

Applies to any platform processing Personal Identifiable Information (PII) of EU citizens (e.g., corporate buyers, European field developers, individual investors). Key considerations:

> * **Lawful Basis and Consent**: Consent managers must record explicit consent for processing location data, bank accounts, and personal contact info.  
> * **Right to Erasure (Article 17\)**: Conflicts with immutable ledger storage. System architecture must isolate PII in off-chain relational databases while storing non-identifiable cryptographic hashes or synthetic keys on permanent immutability layers.

#### **India Digital Personal Data Protection (DPDP) Act 2023**

Applies directly to platform operations based in India or processing Indian citizens' data (e.g., smallholder farmers participating in biochar or agroforestry projects)2:

> * **Data Fiduciary Obligations**: The marketplace acts as a Data Fiduciary and must process personal data solely for specified, lawful development purpose consents.  
> * **Consent Manager Integration**: Smallholder farmers must be provided notice and consent options in plain language (supporting regional Indian languages).  
> * **Cross-Border Transfers**: Compliance with host-nation cross-border data transfer rules. PII relating to farmer land records and identity documents must adhere to local storage regulations where specified by government notifications.

### **KYC/AML and Tokenization Governance**

#### **Standard Carbon Marketplace KYC/AML Requirements**

Marketplace operations require standard financial compliance frameworks:

> * **Entity Verification**: Identity verification of corporate buyers, project developers, and brokers via corporate registry filings, Ultimate Beneficial Ownership (UBO) disclosures, and Director screening.  
> * **Sanctions & PEP Screening**: Real-time screening against global sanction lists (OFAC, EU, UN, UKCA) and Politically Exposed Person (PEP) databases.  
> * **Transaction Monitoring**: Algorithmic monitoring of high-volume cross-border fund flows and credit batch movements to prevent trade-based money laundering.

#### **Tokenization and On-Chain Digital Assets**

If carbon credits are tokenized, fractionalized, or traded on distributed ledgers, additional regulatory layers apply:

> * **FATF Travel Rule**: Financial Action Task Force recommendations mandate that Virtual Asset Service Providers (VASPs) collect and transmit originator and beneficiary information for digital asset transfers exceeding threshold amounts (![][image11]).  
> * **Regulatory Classifications (EU MiCA)**: Under the European Union’s Markets in Crypto-Assets (MiCA) regulation, tokenized carbon credits may be categorized as utility tokens or asset-referenced tokens depending on redemption rights, requiring VASP registration and whitepaper publications.  
> * **Registry Restrictions**: Registries like Verra explicitly prohibit unauthorized third-party tokenization of active credits, requiring formal platform agreements to operationalize digital credit bridges4.

## **5\. Competitive Infrastructure Landscape**

### **Xpansiv (CBL)**

> * **Market Position & Core Functions**: Central market infrastructure provider operating CBL, the world’s largest spot exchange for environmental commodities, alongside direct registry portfolio management tools (EMA) and market data distribution platforms.  
> * **Registry Integration Mechanism**: Direct API connections and secure host-to-host system integrations with major registries (Verra, Gold Standard, ACR, CAR), enabling automated settlement and unit transfers upon trade execution.  
> * **Tokenization Approach**: Operates within traditional centralized financial rails; avoids public blockchain tokenization while delivering digital clearing and registry reconciliation.  
> * **Settlement Infrastructure**: Proprietary central clearing and settlement engines processing physical delivery against cash settlement via corporate banking networks.

### **Carbonplace**

> * **Market Position & Core Functions**: Bank-backed carbon transaction network founded by major financial institutions (BNP Paribas, CIBC, NatWest, Standard Chartered, UBS). Designed as an interbank settlement engine for carbon credits.  
> * **Registry Integration Mechanism**: Direct integration with registries to facilitate secure transfer-versus-payment execution between institutional buyer accounts.  
> * **Tokenization Approach**: Uses private, permissioned ledger technology to manage settlement instructions without deploying volatile public cryptocurrencies or public ERC-20 tokens.  
> * **Settlement Infrastructure**: Utilizes SWIFT-compliant banking infrastructure and institutional cash transfer systems to execute simultaneous delivery-versus-payment (![][image12]).

### **Abatable**

> * **Market Position & Core Functions**: Enterprise carbon procurement and market intelligence platform providing corporate buyers and intermediaries with project diligence tools, pricing benchmarks, and procurement management software.  
> * **Registry Integration Mechanism**: Aggregates registry data via programmatic API connections, scraping layers, and third-party ratings feeds (e.g., BeZero, Sylvera) into a unified interface.  
> * **Tokenization Approach**: Operates strictly on traditional off-chain digital rails; does not perform tokenization.  
> * **Settlement Infrastructure**: Off-market bilateral transaction structuring, software-assisted procurement agreements, and enterprise invoicing workflows.

### **Toucan Protocol**

> * **Market Position & Core Functions**: Web3 infrastructure developer that built digital carbon bridges (Base Carbon Tonne \- BCT, Nature Carbon Tonne \- NCT) to bring voluntary carbon credits onto public blockchains (Polygon, Celo).  
> * **Registry Integration Mechanism**: Initially utilized manual registry retirement workflows where credits were retired on Verra under generic names to mint equivalent on-chain ERC-20 tokens. Following Verra's explicit prohibition on unauthorized tokenization of retired credits in 2022, Toucan paused un-permissioned bridging and pivoted toward building permissioned digital registry frameworks directly partnered with standard-setters.  
> * **Tokenization Approach**: Native ERC-20 tokenization representing fractionalized 1:1 carbon credit backing on-chain.  
> * **Settlement Infrastructure**: Automated decentralized exchange (DEX) liquidity pools (e.g., Uniswap) executing instant on-chain smart contract settlement.

### **Ceezer**

> * **Market Position & Core Functions**: API-first carbon portfolio management marketplace connecting corporate enterprise buyers directly with project developers for screened credit sourcing and risk monitoring.  
> * **Registry Integration Mechanism**: Automated API integrations with registries and D-MRV data providers to aggregate credit availability, project documentation, and vintage status.  
> * **Tokenization Approach**: Off-chain digital platform architecture.  
> * **Settlement Infrastructure**: Automated corporate purchasing workflows, multi-currency cash settlement, and programmatic registry retirement requests.

### **KlimaDAO / Digital Carbon Infrastructure**

> * **Market Position & Core Functions**: Decentralized autonomous organization and liquidity protocol that accumulated significant volumes of tokenized carbon credits into an on-chain treasury, establishing a transparent pricing layer for digital carbon.  
> * **Registry Integration Mechanism**: Ingests tokenized credits minted via digital bridges (Toucan, C3, Moss) into automated market maker (AMM) liquidity pools.  
> * **Tokenization Approach**: On-chain treasury backing utilizing the KLIMA rebase and bonding token model to incentivise digital carbon accumulation.  
> * **Settlement Infrastructure**: Decentralized smart contract execution on public blockchain networks (Polygon), enabling instantaneous programmatic credit purchases and retirements.

### **Competitive Landscape Matrix**

| Platform | Registry Integration Mechanism | Tokenization Model | Settlement Architecture | Primary Target Market | Current Market / Strategic Status |
| :---- | :---- | :---- | :---- | :---- | :---- |
| **Xpansiv (CBL)** | Direct API / Host-to-Host Registry Integrations | Centralized Digital Ledger (No Public Tokens) | Centralized spot exchange clearing & cash network | Institutional traders, energy firms, brokers | Market-leading spot exchange volume provider for voluntary credits. |
| **Carbonplace** | Direct Bank-to-Registry Interfaces | Permissioned Interbank Distributed Ledger | Bank-grade Settlement & Cash DvP Rails | Global enterprise banking clients, corporate treasuries | Active bank-consortium enterprise settlement network. |
| **Abatable** | Aggregated API Ingestion & Diligence Layers | None (Off-Chain Software) | Enterprise Invoicing & Bilateral Procurement Contracts | Corporate buyers, sustainability leads, asset managers | Active enterprise procurement intelligence platform. |
| **Toucan** | Historical Registry Bridging; pivoting to Registry Partnerships | Public ERC-20 Tokens (BCT, NCT) on Polygon/Celo | Automated Smart Contract Decentralized Pools | Web3 applications, digital carbon ecosystem | Pivoted away from un-permissioned bridging after 2022 registry restrictions. |
| **Ceezer** | API-First Multi-Registry Data Ingestion | None (Off-Chain API Platform) | Enterprise Purchasing & Automated Invoicing | Enterprise corporate sustainability teams | Growing market footprint across European corporate procurement. |
| **KlimaDAO** | Aggregates On-Chain Bridges (Toucan, C3, Moss) | Native Protocol Treasury Tokens (KLIMA, carbon-backed pools) | On-Chain Smart Contracts on Polygon Network | Decentralized web3 platforms, programmatic buyers | Operational on-chain carbon liquidity layer and infrastructure platform. |

## **6\. Carbon Credit Pricing Benchmark Infrastructure**

Pricing in the voluntary carbon market is highly fragmented, with credits trading at wide spreads based on project category, location, vintage, methodology, CCP status, and co-benefit certifications4.

### **Primary Pricing Benchmark Providers**

#### **S\&P Global Commodity Insights (Platts)**

Publishes daily price assessments for standardized carbon credit categories, including Platts CEC (Corsia-eligible carbon credits), CNC (Nature-based carbon credits), DRC (Durable carbon removal), and methane abatement indices. Methodology relies on daily market surveys of OTC broker quotes, bids, offers, and executed spot trades.

#### **OPIS (A Dow Jones Company)**

Delivers daily pricing reports and full-curve assessments across REDD+, Improved Forest Management, cookstoves, renewable energy, and biochar credits. Data is gathered through direct interaction with traders, project developers, and brokers.

#### **Ecosystem Marketplace (Forest Trends)**

Provides historical and aggregated price data based on voluntary reporting from a network of over 300 market participants1. Data is updated periodically, offering comprehensive baseline tracking for primary market transaction trends1.

#### **Nasdaq Puro.earth Carbon Removal Indexes**

Publishes reference price indices tracking Carbon Removal Certificates (CORCs)24. The index family includes composite pricing for durable carbon removal, alongside dedicated sub-indices for biochar and bio-based construction materials24.

#### **Xpansiv CBL Market Data**

Exposes real-time bid, offer, and execution pricing from active spot exchange order books for standardized contracts (e.g., GEO, N-GEO, C-GEO).

### **Pricing Drivers and Market Differentials**

Market data demonstrates structural pricing splits across project categories:

> * **Avoidance vs. Removal Spread**: Avoidance credits (legacy REDD+, large-scale renewable energy) trade at substantial discounts (![][image13]), driven by oversupply concerns and strict corporate preference shifts15.  
> * **High-Durability CDR Premium**: Engineered removal pathways (Direct Air Capture, Bio-oil, Biomass Geological Storage) command premium valuations (![][image14]) due to verified permanence, low reversal risk, and scientific durability15. Biochar trades in a mid-tier band (![][image15])15.  
> * **The "CCP Premium"**: Credits carrying ICVCM CCP-Approved labels command a price premium over non-labeled credits within the same methodology class, as corporate procurement policies increasingly require CCP compliance to insulate against greenwashing liability4.  
> * 

## **Executive Context Summary**

**Date**: March 2026  
**Subject**: Technical Baseline & Architectural Requirements for Phase 2 Marketplace Build  
This Executive Context Summary establishes the verified domain facts and technical constraints governing the Voluntary Carbon Market (VCM) as of March 2026\. This summary serves as the foundational domain input for the Phase 2 system architecture team.

### **Core Domain Facts Established for Architecture Phase**

#### **1\. Registry API Heterogeneity**

Registry connectivity cannot be designed around a single protocol standard:

> * **Advanced Programmatic Interfaces**: Isometric exposes open REST APIs (/v0) with OAuth/JWT auth models, strict JSON schemas, and hosted Model Context Protocol (MCP) servers12. Gold Standard and Puro.earth provide functional REST APIs for querying project and unit data9.  
> * **Restricted / Portal Architectures**: Verra relies on its Verra Project Hub and Digital Gateway APIs for workflow management, requiring web scraping or third-party wrappers for external registry reconciliation8. ACR and CAR operate closed portals requiring custom enterprise integration pathways23.

#### **2\. Anti-Double-Counting and Unique Serialization**

Every credit lifecycle interaction requires tracking precise, immutable serial numbers7. The platform infrastructure must maintain absolute synchronization with source registry ledgers to track statuses (Listed, Issued, Transferred, Retired, Cancelled) in real time to avoid double-selling or double-claiming4.

#### **3\. Transition to Digital MRV (D-MRV)**

D-MRV is shifting from an optional innovation to a core requirement for high-value credit categories12. The platform's ingestion engine must be architected to handle multi-modal data streams, including 10-meter satellite spectral imaging (Sentinel-2), 3-meter daily optical tasking (Planet), Synthetic Aperture Radar (SAR), and continuous industrial IoT telemetry streams (dendrometers, cookstove loggers, industrial plant mass balances)12.

#### **4\. Regulatory Quality Gateways (ICVCM & VCMI)**

Market demand is heavily bifurcated by regulatory and governance standards15. The ICVCM Core Carbon Principles (CCPs) serve as a quality gatekeeper15. Platform data models must support multi-label tagging (CCP-Approved, CORSIA Eligible, Article 6 Authorized, CCB Labeled) at the individual serial number level4. Enterprise buyers require audit proof connecting credit retirements to VCMI Claims Code guidelines4.

#### **5\. Data Privacy and Regulatory Compliance**

Operating a marketplace based in India serving a global user base introduces multi-jurisdictional compliance obligations:

> * **GDPR & India DPDP Act 2023**: Requires complete isolation of smallholder farmer and corporate user Personally Identifiable Information (PII) from immutable transaction ledgers2.  
> * **KYC/AML & Digital Assets**: Standard fiat spot trading requires entity screening and beneficial ownership verification. Incorporating tokenization or public blockchain rails triggers FATF Travel Rule compliance, VASP registration obligations, and EU MiCA regulatory standards.

### **Critical Parameters Requiring Re-Verification Prior to Phase 2 Build**

Given the rapid evolution of carbon market regulation and registry software systems, the following parameters must be formally re-verified with registry administrators and compliance counsel immediately prior to executing Phase 2 system buildouts:

> 1. **Verra Digital Gateway API Specifications**: Confirm current production release endpoints, official developer sandbox access keys, and write-permission scopes for the Verra Project Hub8.  
> 2. **ICVCM Methodology Decisions**: Audit the ICVCM website for newly approved or rejected methodologies—specifically regarding pending reviews of Improved Forest Management (IFM), REDD+, and soil organic carbon protocols3.  
> 3. **Host-Nation Article 6.2 Frameworks**: Re-verify host-country authorization procedures and registry transfer mechanisms for credits tagged for CORSIA Phase 1 delivery4.  
> 4. **India DPDP Act Implementation Rules**: Review the latest administrative rules published by the Indian Data Protection Board concerning cross-border data transfer whitelists and consent manager technical integrations affecting agricultural project data2.  
> 5. **Registry Tokenization Policies**: Check for updated formal policy positions from Verra, Gold Standard, and ACR regarding approved, permissioned digital bridging and tokenization protocols.

#### **Works cited**

> 1. Verified Carbon Standard \- Verra, [https://verra.org/programs/verified-carbon-standard/](https://verra.org/programs/verified-carbon-standard/)  
> 2. The 5 Best Isometric Carbon Credit Projects of 2026 \- Regreener, [https://www.regreener.earth/blog/best-carbon-credit-projects-isometric](https://www.regreener.earth/blog/best-carbon-credit-projects-isometric)  
> 3. CCP-Approved Methodologies \- Integrity Council for the Voluntary Carbon Market, [https://icvcm.org/knowledge-resources/ccp-approved-methodologies/](https://icvcm.org/knowledge-resources/ccp-approved-methodologies/)  
> 4. Verra VCS Registry Intelligence | Carbon Verification \- PramaanOne, [https://www.pramaanone.com/registries/verra-vcs](https://www.pramaanone.com/registries/verra-vcs)  
> 5. CORE CARBON PRINCIPLES, ASSESSMENT FRAMEWORK AND ASSESSMENT PROCEDURE, [https://icvcm.org/wp-content/uploads/2024/02/CCP-Book-V2-FINAL-6Feb24-compressed.pdf](https://icvcm.org/wp-content/uploads/2024/02/CCP-Book-V2-FINAL-6Feb24-compressed.pdf)  
> 6. VERRA REGISTRY USER GUIDE \- Voluntary Carbon Markets Integrity Initiative, [https://vcmintegrity.org/wp-content/uploads/2023/09/Verra-Registry-User-Guide.pdf](https://vcmintegrity.org/wp-content/uploads/2023/09/Verra-Registry-User-Guide.pdf)  
> 7. VCU Labels \- Verified Carbon Standard \- Verra, [https://verra.org/programs/verified-carbon-standard/verified-carbon-units-labels/](https://verra.org/programs/verified-carbon-standard/verified-carbon-units-labels/)  
> 8. Verra Project Hub, [https://verra.org/verra-project-hub/](https://verra.org/verra-project-hub/)  
> 9. Carbon Offset Project Scraper — Verra VCS & Gold Standard \- Apify, [https://apify.com/jungle\_synthesizer/carbon-credit-registry-scraper](https://apify.com/jungle_synthesizer/carbon-credit-registry-scraper)  
> 10. Navigating Carbon Credit Registries, [https://insights.carbon-direct.com/hubfs/gated-assets/navigating-carbon-credit-registries.pdf](https://insights.carbon-direct.com/hubfs/gated-assets/navigating-carbon-credit-registries.pdf)  
> 11. COPY OF APPLICATION \- Integrity Council for the Voluntary Carbon Market, [https://icvcm.org/wp-content/uploads/2024/04/Isometric-Copy-of-Application-R-07032024.pdf](https://icvcm.org/wp-content/uploads/2024/04/Isometric-Copy-of-Application-R-07032024.pdf)  
> 12. GitHub \- api-evangelist/isometric: Isometric is a London- and New York-based certifier of climate solutions building an AI-native, science-led registry and verification platform for the industrial economy. Founded in 2022 by Eamon Jubbawy (Onfido), Isometric certifies durable carbon dioxide removal (CDR), superpollutant abatement, and related environmental attributes against the, [https://github.com/api-evangelist/isometric](https://github.com/api-evangelist/isometric)  
> 13. Isometric Documentation, [https://docs.isometric.com/getting-started](https://docs.isometric.com/getting-started)  
> 14. Isometric carbon removal registry, [https://registry.isometric.com/](https://registry.isometric.com/)  
> 15. Structural Shifts in Carbon Credit Markets (2025–2026): What Every Business Should Understand \- Hestiya, [https://www.hestiya.com/blogs/structural-shifts-carbon-credit-markets-2025-2026](https://www.hestiya.com/blogs/structural-shifts-carbon-credit-markets-2025-2026)  
> 16. Claims Code \- Explanatory notes \- Voluntary Carbon Markets Integrity Initiative, [https://vcmintegrity.org/wp-content/uploads/2023/11/Claims-Code-Explanatory-Notes-November-2023.pdf](https://vcmintegrity.org/wp-content/uploads/2023/11/Claims-Code-Explanatory-Notes-November-2023.pdf)  
> 17. Verra Registry Overview, [https://verra.org/registry/overview/](https://verra.org/registry/overview/)  
> 18. GitHub \- yc-wang00/verra-scaper: This project facilitates the extraction of document data from the Verra Verified Carbon Standard (VCS) Registry, an open database widely utilized by carbon credit traders., [https://github.com/yc-wang00/verra-scaper](https://github.com/yc-wang00/verra-scaper)  
> 19. What Is API Authentication? Benefits, Methods & Best Practices | Postman, [https://www.postman.com/api-platform/api-authentication/](https://www.postman.com/api-platform/api-authentication/)  
> 20. How to choose a carbon registry \- Onnu, [https://www.onnu.com/insights/how-to-choose-a-carbon-registry](https://www.onnu.com/insights/how-to-choose-a-carbon-registry)  
> 21. ICVCM CCP-label tracker \- Calyx Global, [https://calyxglobal.com/research-hub/commentary/icvcm-ccp-label-tracker](https://calyxglobal.com/research-hub/commentary/icvcm-ccp-label-tracker)  
> 22. Integrity Council Approves 6 New Carbon Removal (CDR) Methods, [https://icvcm.org/integrity-council-approves-six-carbon-dioxide-removal-methodologies/](https://icvcm.org/integrity-council-approves-six-carbon-dioxide-removal-methodologies/)  
> 23. Voluntary Registry Offsets Database | Berkeley Carbon Trading Project, [https://gspp.berkeley.edu/berkeley-carbon-trading-project/offsets-database](https://gspp.berkeley.edu/berkeley-carbon-trading-project/offsets-database)  
> 24. Puro.earth Carbon Removal Platform \- Nasdaq, [https://www.nasdaq.com/solutions/carbon-removal-platform](https://www.nasdaq.com/solutions/carbon-removal-platform)  
> 25. The Core Carbon Principles Advancing High-Integrity Climate Action at Scale, [https://icvcm.org/wp-content/uploads/2025/12/IC-Impact-Report-2025-FINAL.pdf](https://icvcm.org/wp-content/uploads/2025/12/IC-Impact-Report-2025-FINAL.pdf)  
> 26. Scope 3 Action Code of Practice \- Voluntary Carbon Markets Integrity Initiative, [https://vcmintegrity.org/wp-content/uploads/2025/04/VCMI-Scope-3-Action-Code-of-Practice.pdf](https://vcmintegrity.org/wp-content/uploads/2025/04/VCMI-Scope-3-Action-Code-of-Practice.pdf)















