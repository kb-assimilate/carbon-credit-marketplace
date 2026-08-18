# **Regulatory and Compliance Landscape for Cross-Border Voluntary Carbon Credit Marketplaces in India**

## **Legal Context and Methodological Disclaimer**

This research report provides a regulatory landscape mapping for an India-based voluntary carbon credit marketplace facilitating cross-border transactions. Operating an environmental commodity marketplace at the intersection of international trade, financial technology, and domestic tax law requires careful analysis across foreign exchange rules, payment aggregator licensing, indirect and direct tax obligations, payment processor acceptable-use policies, registry integrity standards, and institutional escrow arrangements.  
This document constitutes landscape research compiled solely to inform specific questions for qualified legal and tax counsel. It does not constitute formal legal advice, a tax opinion, or an individualized professional consultation. Where regulatory interpretations turn on specific corporate structuring, contractual arrangements, or operational payment flows, those matters depend on the specific business structure and require formal advice from certified Chartered Accountants (CA), Company Secretaries (CS), or specialized fintech and trade lawyers.

## **Foreign Exchange Management Act (FEMA) and Cross-Border Capital Flows**

Cross-border financial transactions involving Indian entities are governed by the Foreign Exchange Management Act, 1999 (FEMA) and its statutory rules1. FEMA categorizes foreign exchange transactions into Current Account Transactions and Capital Account Transactions1. Current account transactions, defined as payments due in connection with foreign trade, other current business, services, and short-term banking facilities, are generally permitted unless explicitly restricted under Schedules I, II, or III of the Foreign Exchange Management (Current Account Transactions) Rules1.  
When an India-based marketplace facilitates the sale of voluntary carbon credits—such as Verified Carbon Units (VCUs) certified by Verra, Gold Standard VERs, or Carbon Removal Certificates (CORCs) issued by Puro.earth—from an Indian project developer to an overseas buyer, the transaction represents an export of an intangible asset or service, resulting in an inward remittance3. Conversely, when the Indian marketplace facilitates purchases by domestic buyers from foreign project developers, an outward remittance is generated8. Under current FEMA guidelines, export proceeds for goods and services must be realized and repatriated to India through an Authorized Dealer Category-I (AD Cat-I) bank within 15 months from the date of export2.  
The capital flow mechanics vary based on the residency of the transacting parties. In an inward export flow, the international buyer initiates a cross-border wire transfer via SWIFT or an authorized payment gateway, which is processed through an AD Cat-I bank in India to credit the Indian marketplace or developer account2. For outward developer payouts, the Indian marketplace remits funds to the foreign developer through an AD Cat-I bank under general current account trade rules, accompanied by Form 15CA/15CB declarations under Section 195 of the Income-tax Act9.  
Where domestic Indian retail buyers purchase carbon credits originating from international project developers, the transaction involves an outward remittance governed by the Liberalised Remittance Scheme (LRS)8. Under the LRS, resident individuals are permitted to remit up to USD 250,000 per financial year (April to March) across all permitted current and capital account transactions combined8.  
The LRS framework applies strictly to resident individuals, including minors represented by legal guardians8. Corporate entities, partnership firms, Hindu Undivided Families (HUFs), and Trusts are legally excluded from utilizing the LRS8. Corporate purchases of foreign carbon credits must proceed under general commercial trade remittance rules backed by commercial invoices and contracts2. Outward remittances under LRS attract Tax Collected at Source (TCS) under Section 206C(1G) of the Income-tax Act, where a 20% TCS rate applies to general remittance amounts exceeding INR 10 lakh in a financial year2. While international credit card spending abroad remains deferred from the LRS aggregate limit, international debit cards, forex cards, and direct bank wires execute under strict LRS tracking via Form A2 and purpose code declarations8.

### **Disambiguation of FEMA Provisions**

| Regulatory Dimension | (a) Documented Public Fact | (b) Business-Specific Legal Determination (Requires Counsel) |
| :---- | :---- | :---- |
| **Current vs. Capital Account** | Current account transactions are permitted unless restricted under statutory schedules1. The standard export realization timeline is 15 months2. | Determination of the precise FEMA Purpose Code (e.g., P0103 for software/data services vs. P1007/P1302 for trade in intangible property) to be assigned by the AD Cat-I bank based on invoice structuring. |
| **LRS Scope and Limits** | The LRS limit is USD 250,000 per financial year for resident individuals and does not apply to corporate entities8. TCS of 20% applies above INR 10 lakh for general category remittances2. | Determination of whether a retail purchase of an international carbon credit qualifies as an intangible good purchase (Current Account) or an overseas asset acquisition (Capital Account) under the marketplace's specific contract terms. |

## **RBI Payment Aggregator and Payment Gateway Framework**

Intermediaries facilitating online payment flows in India are regulated by the Reserve Bank of India (RBI) under the Payment and Settlement Systems Act, 2007 (PSSA)12. The operative regulatory framework is set forth in the RBI circular titled *"Guidelines on Regulation of Payment Aggregators and Payment Gateways"* dated March 17, 2020 (updated November 17, 2020\)12.  
The RBI guidelines establish a formal distinction between technology infrastructure providers and financial intermediaries based on whether the entity takes custody of customer funds12:

> * **Payment Gateways (PGs)**: PGs are defined as entities that provide technology infrastructure to route and facilitate the processing of an online payment transaction without taking custody of funds13. PGs are classified as technology providers and do not require formal RBI authorization, although they must adhere to baseline technology and security standards13.  
> * **Payment Aggregators (PAs)**: PAs are defined as entities that facilitate e-commerce sites and merchants to accept various payment instruments from customers to settle payment obligations13. PAs receive payments from customers, pool them, and transfer them to merchants after a designated settlement period13. Non-bank PAs require mandatory authorization from the RBI under Section 7 of the PSSA12.

The primary trigger for mandatory PA authorization is the **handling or pooling of customer funds**12. If a voluntary carbon credit marketplace receives funds from buyers, holds those funds in its own bank account or temporary escrow, and subsequently releases payment to credit sellers or project developers after trade confirmation, the marketplace is performing fund aggregation under the March 17, 2020 guidelines12.  
Non-bank PAs authorized by the RBI must maintain a dedicated escrow account with a Scheduled Commercial Bank to route customer settlements13. The RBI strictly enumerates permissible debits and credits to this escrow account to ensure funds are not diverted, used for working capital, or held beyond permitted settlement timelines (T+1 or T+2 based on settlement model)13. Operating a platform that holds buyer funds prior to seller disbursement without obtaining a PA license or without utilizing an authorized payment aggregator or bank escrow violates the PSSA12.  
In a direct payment gateway integration model, the customer payment flows directly through a licensed bank or authorized payment aggregator to the merchant account, placing the software platform outside the scope of PA licensing. Conversely, when a marketplace intercepts customer funds by holding them in an escrow or node account before releasing them to third-party developers, the platform takes custody of funds, triggering the requirement for mandatory RBI PA authorization12.

### **Disambiguation of RBI PA-PG Provisions**

| Regulatory Dimension | (a) Documented Public Fact | (b) Business-Specific Legal Determination (Requires Counsel) |
| :---- | :---- | :---- |
| **Licensing Triggers** | Handling and pooling customer funds before settling to third-party merchants triggers mandatory RBI Payment Aggregator licensing under the March 17, 2020 circular12. | Assessment of whether the marketplace software flow can be structured as a pure marketplace software routing mechanism to avoid handling funds directly. |
| **Escrow Account Rules** | Non-bank PAs must maintain a dedicated escrow account with a Scheduled Commercial Bank with strictly limited permissible debits and credits13. | Structuring payment split logic at the payment gateway API level to separate platform marketplace commissions from seller principal payouts automatically. |

## **Indirect Taxation: Goods and Services Tax (GST) Treatment**

The classification and taxation of carbon credits under India's Goods and Services Tax (GST) framework remain complex, with historical precedents across direct tax, value-added tax (VAT), and regulatory circulars7. Carbon credits—whether issued as Certified Emission Reductions (CERs), Verified Carbon Units (VCUs), or Carbon Credit Certificates (CCCs)—represent tradable intangible environmental instruments7.  
The tax characterization of carbon credits has been evaluated across multiple legal contexts:

> * **Income Tax Jurisprudence**: The Madras High Court in *CIT v. Wescare (India) Ltd.* held that income from the sale of carbon credits (CERs) generated from renewable energy operations constitutes a capital receipt rather than taxable operational business income, as carbon credits are generated due to environmental concerns rather than commercial production16.  
> * **State VAT Regimes**: Under pre-GST state VAT laws, carbon credits were consistently classified as "goods" and subjected to local VAT17.  
> * **GST Council Circulars on Similar Instruments**: Following GST implementation, the GST Council addressed similar tradable environmental and financial certificates7. Circular No. 34/8/2018-GST clarified that Priority Sector Lending Certificates (PSLCs) are taxable as goods at 18% under the residuary entry of Notification No. 1/2017-Central Tax (Rate)7. Circular No. 46/20/2018-GST subsequently clarified that Renewable Energy Certificates (RECs) and PSLCs fall under Chapter Heading 4907 of the Customs Tariff Act, 1975, attracting a 12% GST rate7. Notification No. 8/2021-Central Tax (Rate) later increased the GST rate on RECs, PSLCs, and similar certificates to 18%7.  
> * **Customs Advance Rulings**: The Customs Authority for Advance Rulings (CAAR), Mumbai (*Re: United Breweries Ltd.*), ruled that International Renewable Energy Certificates downloaded in electronic form qualify as "intangible goods"7. However, CAAR noted that in the absence of physical import machinery under customs law, purely electronic downloads of intangible instruments do not attract basic customs duty or import GST at the customs frontier7.

Cross-border trading of carbon credits creates operational challenges under Integrated Goods and Services Tax (IGST) rules regarding zero-rated export status7:

> 1. **Export of Goods Classification**: If carbon credits are classified as intangible goods under Chapter Heading 4907, zero-rated export treatment under Section 16 of the IGST Act requires the exporter to file a Shipping Bill or Bill of Export on the ICEGATE portal7. Electronic transfers of carbon credits between registry accounts do not generate physical shipping bills, creating compliance friction under Rule 96 of the Central Goods and Services Tax (CGST) Rules, 2017, which links IGST export refunds directly to shipping bill data7.  
> 2. **Export of Services Classification**: If carbon credits or marketplace listing activities are characterized as environmental or management services (SAC 9983), zero-rating requires satisfying the five statutory conditions of Section 2(6) of the IGST Act, including receipt of payment in convertible foreign exchange and ensuring the supplier and recipient are not merely establishments of a distinct person7.  
> 3. **Reverse Charge Mechanism (RCM)**: Importing carbon credits or registry verification services from overseas entities by an Indian marketplace requires evaluation under the Reverse Charge Mechanism (RCM) for IGST liability7.

### **Disambiguation of GST Provisions**

| Tax Dimension | (a) Documented Public Fact | (b) Business-Specific Legal Determination (Requires Counsel) |
| :---- | :---- | :---- |
| **Applicable Tax Rates** | Tradable certificates (RECs, PSLCs) carry an 18% GST rate under Chapter 4907 / residuary classification pursuant to Notification No. 8/2021-Central Tax (Rate)7. | Determination of whether platform transaction facilitation fees should be billed separately as services (SAC 9983 at 18%) from the underlying transfer of carbon credits. |
| **Zero-Rated Export Compliance** | Rule 96 of the CGST Rules requires shipping bill matching for exports of goods to process IGST refunds7. | Structuring digital registry transfers to secure IGST zero-rated export benefits without generating physical customs shipping bills. |

## **Direct Taxation: Cross-Border Withholding Tax (Section 195\)**

When an India-based marketplace remits funds cross-border to an international project developer (located outside India) for the purchase or facilitation of carbon credits, the transaction is subject to Section 195 of the Income-tax Act, 196110.  
Section 195 mandates that any person responsible for paying a non-resident or foreign company any sum chargeable under the Income-tax Act must deduct tax at source (TDS) at the applicable rates10. Unlike domestic TDS provisions, Section 195 contains no minimum monetary threshold10.  
Tax withholding under Section 195 applies only if the payment contains an element of income chargeable to tax in India10:

> * **Business Profits (Article 7\)**: If payments to a foreign developer represent business profits from the sale of carbon credits, the sum is taxable in India under Section 9(1)(i) only if the non-resident developer has a Business Connection or Permanent Establishment (PE) in India. In the absence of an Indian PE, business profits are taxable solely in the developer’s country of residence under Article 7 of the applicable Double Tax Avoidance Agreement (DTAA).  
> * **Fees for Technical Services / Royalty (Article 12\)**: If tax authorities characterize the payment as consideration for technical services, automated verification software access, or proprietary MRV (Measurement, Reporting, Verification) technology, withholding tax rates of 10% to 20% (plus applicable surcharge and cess) may apply under domestic law or relevant treaty provisions.  
> * **Procedural Documentation**: Prior to executing an outward foreign remittance, the remitting entity must file Form 15CA online9. For payments exceeding INR 5 lakh that are claimed to be non-taxable, the remitter must obtain Form 15CB—a formal certificate signed by an independent Chartered Accountant certifying non-taxability under the Income-tax Act read alongside the relevant DTAA9.

### **Disambiguation of Direct Tax Provisions**

| Tax Dimension | (a) Documented Public Fact | (b) Business-Specific Legal Determination (Requires Counsel) |
| :---- | :---- | :---- |
| **Section 195 Statutory Scope** | Tax withholding applies to all payments to non-residents containing income chargeable to tax in India, with no statutory minimum threshold10. | Determining whether payments to foreign developers represent pure offshore sale of property (exempt from Indian TDS) or taxable technical fees/royalties. |
| **DTAA Relief Requirements** | Treaty benefits require obtaining a Tax Residency Certificate (TRC), Form 10F, and a No-PE declaration from the foreign developer9. | Reviewing developer contract terms to confirm Permanent Establishment risks and securing CA certification under Form 15CB prior to remittance. |

## **Payment Provider Policies and Registry Digital Asset Rules**

Commercial payment processors, payment aggregators, and voluntary carbon registries enforce terms of service that restrict certain environmental commodity transactions and digital asset structures13.  
Payment Aggregators such as Stripe, Razorpay, and merchant acquirers maintain explicit Acceptable Use Policies (AUP) and Prohibited/Restricted Business lists13. Marketplaces dealing in environmental commodities, carbon offsets, and renewable energy certificates are frequently classified as "Restricted Businesses" or "High-Risk Verticals." Merchant onboarding in these categories requires pre-approval and enhanced underwriting due to specific risk factors13:

> 1. *Delivery Risk*: Marketplaces listing unissued or forward-delivery (ex-ante) carbon credits introduce chargeback risk if project validation or credit issuance fails.  
> 2. *Greenwashing and Regulatory Risk*: Inability to verify credit provenance exposes payment processors to reputational and regulatory scrutiny3.  
> 3. *Unauthorized Tokenization*: Intermediaries attempting Web3 tokenization without registry authorization trigger risk freezes under payment processor terms21.

Major voluntary carbon registries maintain specific public rules regarding the tokenization and blockchain representation of their credits21:

> * **Verra (Verified Carbon Standard)**: On May 25, 2022, Verra issued a formal policy prohibition against creating crypto tokens or digital instruments based on *retired* carbon credits, taking the position that retirement permanently consumes the credit's environmental benefit21. Verra's public statement established that tokenizing retired credits undermines environmental integrity21. Verra has evaluated frameworks for immobilizing active credits in registry accounts subject to Know-Your-Customer (KYC) checks, but unapproved third-party tokenization remains prohibited under the Verra Registry Terms of Use21.  
> * **Gold Standard**: Updated its Registry Terms of Use to state that the creation of digital tokens, cryptocurrencies, or digital assets representing Gold Standard credits is **prohibited without express written consent**22. Entities seeking to tokenize Gold Standard credits must execute formal terms of reference and undergo institutional review22.

### **Disambiguation of Payment Provider and Registry Rules**

| Domain | (a) Documented Public Fact | (b) Business-Specific Legal Determination (Requires Counsel) |
| :---- | :---- | :---- |
| **PSP Underwriting Rules** | Payment aggregators classify environmental commodities as restricted verticals requiring underwriting pre-approval13. | Securing written custom approval from payment providers tailored to the marketplace's trade delivery timelines. |
| **Registry Digital Asset Policies** | Verra prohibits tokenizing retired credits21. Gold Standard requires express written consent for tokenization22. | Designing the platform's database and ledger architecture to prevent terms-of-service violations with carbon registries. |

## **VCMI Claims Code of Practice and Required Reporting Metadata**

Corporate climate claims are evaluated under the Voluntary Carbon Markets Integrity Initiative (VCMI) Claims Code of Practice and the Integrity Council for the Voluntary Carbon Market (ICVCM) Core Carbon Principles (CCPs)24.  
To enable corporate buyers to make valid claims (such as VCMI Carbon Integrity Silver, Gold, or Platinum claims) and to satisfy audit standards under the EU Corporate Sustainability Reporting Directive (CSRD) (ESRS E1), carbon credits must be retired transparently with specific metadata recorded on public registries3. Under CSRD, carbon credits are classified under "beyond value chain mitigation" and cannot offset Scope 1–3 operational reduction targets3.  
To support corporate claim verification, an environmental marketplace must capture, document, and display specific metadata fields for every credit retirement transaction3.

### **Standardized Reporting Field Requirements for Credit Retirement Claims**

| Field \# | Data Field Name | Specific Reporting Requirement and Technical Description | Sourced Registry Context |
| :---- | :---- | :---- | :---- |
| **1** | **Unique Serial Identifier** | Complete, unbroken registry serial number string assigned to the credit batch. | Verra, Gold Standard, Isometric, ACR4 |
| **2** | **Standard & Program Name** | Operating registry and crediting program (e.g., Verra VCS Version 5, Gold Standard for the Global Goals, Isometric, Puro.earth)3. | Public Registry Schema4 |
| **3** | **Registry Public Record URL** | Direct, unauthenticated hyperlinked URI pointing to the canonical registry issuance/retirement record4. | Verra Project Hub / Isometric Registry API4 |
| **4** | **Project Identifier & Name** | Official numerical project ID and registered title (e.g., Verra Project ID \#2304)4. | Registry Project Master Database3 |
| **5** | **Host Country & Coordinates** | Geographic location of project implementation, including ISO 3166-1 country code and latitude/longitude4. | Registry Location Metadata4 |
| **6** | **Methodology / Protocol Code** | Standardized protocol code governing quantification (e.g., VM0048 for REDD+, AMS-III.R, Isometric Biochar Protocol)3. | Methodological Framework3 |
| **7** | **Vintage Year** | Calendar year in which the greenhouse gas emission reduction or removal occurred3. | Unit Level Vintage Tag3 |
| **8** | **Volume (Metric Tonnes CO2e)** | Exact quantity of credits retired, where 1 credit \= 1 metric tonne of CO2 equivalent3. | Transaction Volume Field3 |
| **9** | **ICVCM CCP Label Status** | Binary indicator confirming whether the credit batch carries an official Core Carbon Principles (CCP) label3. | ICVCM Quality Tag3 |
| **10** | **Beneficial Owner Name** | Legal entity name of the ultimate corporate buyer making the climate claim23. | Retirement Certificate Field25 |
| **11** | **Retirement Reason / Purpose** | Narrative statement describing the claim purpose (e.g., "VCMI Carbon Integrity Gold Claim FY2025-26")3. | Inscription Metadata3 |
| **12** | **VVB Audit Report Link** | Link to the accredited Validation and Verification Body (VVB) audit statement and monitoring report3. | Registry Audit Archive3 |

### **Disambiguation of VCMI and Registry Data Provisions**

| Governance Dimension | (a) Documented Public Fact | (b) Business-Specific Legal Determination (Requires Counsel) |
| :---- | :---- | :---- |
| **Data Schema Completeness** | VCMI and CSRD standards require serial numbers, beneficial owner names, project IDs, and VVB reports to substantiate corporate claims3. | Structuring legal indemnities in buyer contracts regarding potential credit invalidation or registry reversal events. |
| **CCP Qualification Rules** | ICVCM CCP labels require both program-level eligibility and methodology-level approval24. | Reviewing marketing representations to ensure credit listings comply with Indian consumer protection standards regarding environmental claims. |

## **Institutional Escrow Services Framework in India**

To facilitate high-value commercial transactions without taking direct custody of customer funds, marketplaces can utilize third-party institutional escrow arrangements13. Dedicated third-party escrow providers in India operate under established regulatory frameworks:

> 1. **Scheduled Commercial Banks**: Banks regulated under the Banking Regulation Act, 1949, offer commercial escrow account services13. Escrow accounts held directly with Scheduled Commercial Banks operate under tri-party agreements between the bank, the buyer, and the seller, keeping funds off the marketplace platform’s balance sheet13.  
> 2. **SEBI-Registered Debenture Trustees**: Corporate trustees regulated under the Securities and Exchange Board of India (Debenture Trustees) Regulations, 1993, manage institutional escrow arrangements for commercial transactions15. Trustees act as independent fiduciaries holding funds or title documents in escrow, releasing funds strictly upon fulfillment of verified contractual conditions (such as registry credit retirement confirmation)15.  
> 3. **Cross-Border Escrow via AD Cat-I Banks**: Where transactions involve foreign currency flows from overseas buyers, escrow accounts must be administered through Authorized Dealer Category-I banks compliant with FEMA current account rules, handling foreign exchange conversion and inward remittance certification2.

In a third-party institutional escrow structure, the buyer remits funds directly to an escrow account managed by a SEBI-registered trustee or AD Cat-I bank2. The escrow agent verifies that contractual conditions—such as credit issuance and registry verification—are satisfied before disbursing principal proceeds to the seller and platform commission fees to the marketplace.

### **Disambiguation of Escrow Provisions**

| Escrow Dimension | (a) Documented Public Fact | (b) Business-Specific Legal Determination (Requires Counsel) |
| :---- | :---- | :---- |
| **Regulatory Frameworks** | Third-party escrow providers operate under RBI bank directions or SEBI Debenture Trustee regulations12. | Drafting tri-party escrow agreements that define conditions for fund release, credit verification, and default remedies. |
| **Cross-Border Forex Rules** | Foreign exchange escrow accounts must be managed by AD Cat-I banks under FEMA rules2. | Determining whether marketplace commission fees can be automatically deducted within the bank escrow prior to seller disbursement. |

## **Comprehensive Agenda for Legal and Tax Counsel**

The following specific questions are organized by regulatory topic to guide consultation with qualified professional advisers.

### **Questions for Chartered Accountant (CA) / Company Secretary (CS)**

#### **1\. Foreign Exchange Management Act (FEMA) & LRS**

> * Which specific FEMA Purpose Code should be assigned by our AD Cat-I bank for inward export remittances resulting from carbon credit sales to overseas buyers?  
> * What accounting workflow is required to track export realizations and ensure compliance with the 15-month repatriation rule under FEMA directions2?  
> * How should the marketplace track retail outward remittances under the Liberalised Remittance Scheme (LRS) to ensure compliance with the USD 250,000 annual limit and 20% TCS requirements2?

#### **2\. RBI Payment Aggregator / Gateway Framework**

> * From a financial audit perspective, how must customer funds be reflected if routed through a bank escrow account to ensure they are excluded from the marketplace's operational balance sheet13?

#### **3\. Goods and Services Tax (GST)**

> * Should the cross-border sale of carbon credits be classified as an export of goods under Chapter Heading 4907 or as an export of services under SAC 99837?  
> * In the absence of a physical customs shipping bill for digital registry transfers, what documentation will the GST department accept to process zero-rated export refunds under Rule 96 of the CGST Rules7?  
> * Does Reverse Charge Mechanism (RCM) IGST apply to registry verification and listing fees paid to overseas standard-setting bodies such as Verra or Gold Standard7?

#### **4\. Withholding Tax (Section 195\)**

> * Do cross-border payments made to non-resident project developers constitute business profits exempt from Indian TDS under Article 7 of applicable DTAAs, or could they be characterized as Fees for Technical Services (FTS) under Section 9(1)10?  
> * What documentation (TRC, Form 10F, No-PE declaration) is required to issue Form 15CB CA certificates for foreign outward remittances9?

#### **5\. Payment Provider Policies**

> * How should marketplace fee structures (platform commission vs. principal credit cost) be itemized on customer tax invoices to satisfy payment processor audit requirements13?

#### **6\. VCMI Reporting Metadata**

> * What internal accounting controls are necessary to maintain an unbroken audit trail matching credit retirement metadata with corporate purchase invoices for CSRD audit compliance3?

#### **7\. Institutional Escrow Services**

> * What are the tax withholding and invoicing implications when transaction proceeds are disbursed to international sellers directly from a third-party bank escrow account13?

### **Questions for Fintech, Regulatory & Corporate Lawyer**

#### **1\. Foreign Exchange Management Act (FEMA)**

> * Does the sale of an international carbon credit to a domestic Indian retail buyer qualify as a current account import of intangible goods or an overseas capital account investment under FEMA rules1?

#### **2\. RBI Payment Aggregator / Gateway Framework**

> * Does holding buyer funds in a temporary escrow account prior to registry credit transfer constitute payment aggregation under the RBI’s March 17, 2020 guidelines, requiring a PA license12?  
> * Can the marketplace structure its software integration so that buyer funds flow directly to a licensed payment aggregator or bank, avoiding direct custody of funds12?

#### **3\. Goods and Services Tax (GST)**

> * How can cross-border digital credit transfers be structured in marketplace terms of service to satisfy the legal definition of zero-rated exports under Section 16 of the IGST Act7?

#### **4\. Withholding Tax (Section 195\)**

> * How should developer contracts be drafted to establish that title transfer occurs offshore, minimizing Indian tax chargeability and Permanent Establishment exposure10?

#### **5\. Payment Provider Policies & Registry Tokenization**

> * What specific contractual terms must be included in user agreements to comply with payment processor acceptable-use policies regarding environmental commodities13?  
> * Does the marketplace's ledger architecture comply with Verra’s policy against tokenizing retired credits and Gold Standard’s requirement for express written consent21?

#### **6\. VCMI Claims & Liability**

> * What legal disclosures and disclaimers must be included on the marketplace platform to protect against greenwashing liability under Indian consumer protection laws3?

#### **7\. Institutional Escrow Structuring**

> * Can a SEBI-registered Debenture Trustee act as an escrow agent for high-value B2B carbon credit trades, and what specific triggers should govern fund release and refund mechanisms15?

#### **Works cited**

> 1. Capital & Current Account Transactions, [https://www.bcasonline.org/Referencer2018-19/part1/capital-current-account-transactions.html](https://www.bcasonline.org/Referencer2018-19/part1/capital-current-account-transactions.html)  
> 2. Current Account Transactions Under FEMA: 2026 Rules & Limits \- Razorpay, [https://razorpay.com/blog/current-account-transactions-under-fema/](https://razorpay.com/blog/current-account-transactions-under-fema/)  
> 3. Verra VCS Registry Intelligence | Carbon Verification \- PramaanOne, [https://www.pramaanone.com/registries/verra-vcs](https://www.pramaanone.com/registries/verra-vcs)  
> 4. Carbon Offset Project Scraper — Verra VCS & Gold Standard \- Apify, [https://apify.com/jungle\_synthesizer/carbon-credit-registry-scraper](https://apify.com/jungle_synthesizer/carbon-credit-registry-scraper)  
> 5. Carbon standards: Gold Standard, Verra, Puro, ISO, LBC | Arka, [https://arkaclimate.com/learn/standards/](https://arkaclimate.com/learn/standards/)  
> 6. Carbon Credit Standards: Which to Trust in 2026 \- Regreener, [https://www.regreener.earth/blog/carbon-credit-standards-compared](https://www.regreener.earth/blog/carbon-credit-standards-compared)  
> 7. Buy, Sell, Confuse: The Carbon Credit Game Show across borders \- Bar and Bench, [https://www.barandbench.com/view-point/buy-sell-confuse-the-carbon-credit-game-show-across-borders](https://www.barandbench.com/view-point/buy-sell-confuse-the-carbon-credit-game-show-across-borders)  
> 8. LRS limits 2024–2026: Avoid penalties before you remit \- Karbon Business, [https://www.karboncard.com/blog/lrs-limits-2024-2026-guide](https://www.karboncard.com/blog/lrs-limits-2024-2026-guide)  
> 9. Liberalised Remittance Scheme (LRS): Meaning, Rules, Limits and How It Works \- Groww, [https://groww.in/blog/liberalised-remittance-scheme](https://groww.in/blog/liberalised-remittance-scheme)  
> 10. Section 195 of Income Tax Act \- TDS Applicability for NRI \- Bajaj Finserv, [https://www.bajajfinserv.in/investments/section-195-of-income-tax-act](https://www.bajajfinserv.in/investments/section-195-of-income-tax-act)  
> 11. FAQs about the Liberalised Remittance Scheme (LRS) \- HSBC India, [https://www.hsbc.co.in/help/faqs/lrs/](https://www.hsbc.co.in/help/faqs/lrs/)  
> 12. Regulation of Fintech in Asia-Pacific \- DOKUMEN.PUB, [https://dokumen.pub/regulation-of-fintech-in-asia-pacific.html](https://dokumen.pub/regulation-of-fintech-in-asia-pacific.html)  
> 13. NATIONAL SECURITIES DEPOSITORY LIMITED \- BSE, [https://www.bseindia.com/corporates/download/329165/NSDL%20Addendum%20and%20DRHP\_20250520164131.pdf](https://www.bseindia.com/corporates/download/329165/NSDL%20Addendum%20and%20DRHP_20250520164131.pdf)  
> 14. [https://www.rbi.org.in/Scripts/BS\_ViewMasDirections.aspx?id=12896](https://www.rbi.org.in/Scripts/BS_ViewMasDirections.aspx?id=12896)  
> 15. NATIONAL SECURITIES DEPOSITORY LIMITED ... \- NSE, [https://nsearchives.nseindia.com/content/equities/NSDL\_RHP.pdf](https://nsearchives.nseindia.com/content/equities/NSDL_RHP.pdf)  
> 16. Sale of carbon credits is capital receipt and not taxable \- TaxGuru, [https://taxguru.in/income-tax/sale-carbon-credits-capital-receipt-taxable.html](https://taxguru.in/income-tax/sale-carbon-credits-capital-receipt-taxable.html)  
> 17. taxupindia.com \- WordPress.com, [https://taxupindia.files.wordpress.com/2017/06/gst-ready-reckoner-by-taxmann-42-chapters-updated.pdf](https://taxupindia.files.wordpress.com/2017/06/gst-ready-reckoner-by-taxmann-42-chapters-updated.pdf)  
> 18. Overview of GST Implementation in India | PDF | Value Added Tax \- Scribd, [https://www.scribd.com/document/797042206/Gst-Master-File](https://www.scribd.com/document/797042206/Gst-Master-File)  
> 19. Hostel accommodation to attract 12% GST: AAR \- The Hindu, [https://www.thehindu.com/business/Economy/hostel-accommodation-to-attract-12-gst-aar/article67135249.ece](https://www.thehindu.com/business/Economy/hostel-accommodation-to-attract-12-gst-aar/article67135249.ece)  
> 20. Section 195 of the Income-tax Act, 1961, [https://incometaxindia.gov.in/\_layouts/15/dit/pages/viewer.aspx?grp=act\&cname=cmsid\&cval=102120000000041704\&k=section+195\&isdlg=0](https://incometaxindia.gov.in/_layouts/15/dit/pages/viewer.aspx?grp=act&cname=cmsid&cval=102120000000041704&k=section+195&isdlg=0)  
> 21. Verra Addresses Crypto Instruments and Tokens, [https://verra.org/verra-addresses-crypto-instruments-and-tokens/](https://verra.org/verra-addresses-crypto-instruments-and-tokens/)  
> 22. Conditions for consenting to tokenisation of Gold Standard-issued credits | GS, [https://www.goldstandard.org/consultations/conditions-for-consenting-to-tokenisation-of-gold-standard-issued-credits](https://www.goldstandard.org/consultations/conditions-for-consenting-to-tokenisation-of-gold-standard-issued-credits)  
> 23. Verras consultation on carbon tokens \- Toucan Protocol, [https://blog.toucan.earth/verra-consultation-summary/](https://blog.toucan.earth/verra-consultation-summary/)  
> 24. Structural Shifts in Carbon Credit Markets (2025–2026): What Every Business Should Understand \- Hestiya, [https://www.hestiya.com/blogs/structural-shifts-carbon-credit-markets-2025-2026](https://www.hestiya.com/blogs/structural-shifts-carbon-credit-markets-2025-2026)  
> 25. Isometric Documentation, [https://docs.isometric.com/getting-started](https://docs.isometric.com/getting-started)  
> 26. GitHub \- api-evangelist/isometric: Isometric is a London- and New York-based certifier of climate solutions building an AI-native, science-led registry and verification platform for the industrial economy. Founded in 2022 by Eamon Jubbawy (Onfido), Isometric certifies durable carbon dioxide removal (CDR), superpollutant abatement, and related environmental attributes against the, [https://github.com/api-evangelist/isometric](https://github.com/api-evangelist/isometric)  
> 27. COPY OF APPLICATION \- Integrity Council for the Voluntary Carbon Market, [https://icvcm.org/wp-content/uploads/2024/04/Isometric-Copy-of-Application-R-07032024.pdf](https://icvcm.org/wp-content/uploads/2024/04/Isometric-Copy-of-Application-R-07032024.pdf)  
> 28. Top Carbon Credit Registries Compared — Verra, Gold Standard & More \- ESGweise, [https://www.esgweise.com/insights/top-carbon-credit-registries-compared/](https://www.esgweise.com/insights/top-carbon-credit-registries-compared/)  
> 29. Carbon Credit Tokenization Development Company | Clarisco, [https://www.clarisco.com/carbon-credits-tokenization-development](https://www.clarisco.com/carbon-credits-tokenization-development)  
> 30. Verra vs Gold Standard 2026: Carbon Credit Standards Compared \- FG Capital Advisors, [https://www.fgcapitaladvisors.com/verra-vs-gold-standard-2026-carbon-credit-standards-compared](https://www.fgcapitaladvisors.com/verra-vs-gold-standard-2026-carbon-credit-standards-compared)  
> 31. Voluntary Registry Offsets Database | Berkeley Carbon Trading Project, [https://gspp.berkeley.edu/berkeley-carbon-trading-project/offsets-database](https://gspp.berkeley.edu/berkeley-carbon-trading-project/offsets-database)  
> 32. Voluntary carbon markets: overview of frameworks \- DLA Piper, [https://www.dlapiper.com/en/insights/topics/carbon-markets-hub/voluntary-carbon-markets-frameworks](https://www.dlapiper.com/en/insights/topics/carbon-markets-hub/voluntary-carbon-markets-frameworks)  
> 33. Verra Project Hub, [https://verra.org/verra-project-hub/](https://verra.org/verra-project-hub/)  
> 34. The 5 Best Isometric Carbon Credit Projects of 2026 \- Regreener, [https://www.regreener.earth/blog/best-carbon-credit-projects-isometric](https://www.regreener.earth/blog/best-carbon-credit-projects-isometric)  
> 35. Isometric carbon removal registry, [https://registry.isometric.com/](https://registry.isometric.com/)  
> 36. Verified Carbon Standard \- Verra, [https://verra.org/programs/verified-carbon-standard/](https://verra.org/programs/verified-carbon-standard/)  
> 37. VCU Labels \- Verified Carbon Standard \- Verra, [https://verra.org/programs/verified-carbon-standard/verified-carbon-units-labels/](https://verra.org/programs/verified-carbon-standard/verified-carbon-units-labels/)