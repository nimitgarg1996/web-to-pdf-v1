# plan.md
document_type: WILL

---

## Dependencies
- B requires A — document must be retrieved before extraction
- D requires A × A — comparison requires two document instances
- F requires C — lifecycle requires validation first

---

## A — Document Access

### A1 — By Identity
- document_id: unique identifier of the will document
- testator_name: full legal name of the testator

### A2 — By Time
- execution_date: date the will was signed and executed
- latest_version: most recent version of the will on record
- amendment_history: list of codicils or amendments with dates

### A3 — By Version
- version_number: specific version identifier
- version_date: date of this version
- superseded_by: reference to the version that replaced this one

---

## B — Content Extraction

### B1 — People
- testator: person making the will — full legal name, address, declaration
- executor: person appointed to administer the estate — name, conditions of appointment
- co_executor: any co-executor appointed alongside the primary executor
- successor_executor: person named if primary executor cannot or will not serve
- beneficiary_primary: persons who receive assets as primary beneficiaries — names and entitlements
- beneficiary_contingent: persons who receive assets if primary beneficiary predeceases
- guardian: person appointed as guardian of minor children — name and conditions
- trustee: person appointed to manage any testamentary trust — name and conditions
- successor_trustee: person named if primary trustee cannot serve
- first_death: first party to die among those named in survivorship conditions
- second_death: party who survives the first to die — survivorship period required

### B2 — Assets
- residuary_estate: everything remaining after specific bequests — composition and distribution
- specific_bequest_monetary: named monetary amounts bequeathed to named persons
- specific_bequest_property: named real property or titled assets bequeathed to named persons
- specific_bequest_personal: named personal items or tangible property bequeathed
- business_interests: any business ownership interests and their disposition
- digital_assets: digital accounts, cryptocurrency, or online assets and their disposition
- trust_provisions: assets directed into a testamentary trust — terms and beneficiaries
- pour_over_provisions: assets directed into an existing living trust on death

### B3 — Clauses
- survivorship_clause: required survival period for a beneficiary to inherit
- simultaneous_death_clause: provisions for simultaneous or near-simultaneous death
- anti_lapse_clause: provisions preventing lapse of bequest if beneficiary predeceases
- no_contest_clause: forfeiture conditions for any party who contests the will
- spendthrift_clause: restrictions on beneficiary access or assignment of inherited assets
- disinheritance_clause: explicit exclusion of any person from inheritance
- funeral_instructions: burial, cremation, or memorial preferences stated in the will
- organ_donation: organ or body donation instructions if present

### B4 — Attestation
- testator_signature: location and confirmation of testator signature
- witness_one: name, address, and signature location of first witness
- witness_two: name, address, and signature location of second witness
- notarisation: notary name, commission details, and date if notarised
- self_proving_affidavit: presence and details of self-proving affidavit if attached

---

## C — Validation & Compliance

### C1 — Legal Validity
- execution_requirements: whether signing, witnessing, and dating requirements are met
- testamentary_capacity: evidence that testator was of sound mind at execution
- witness_competency: whether witnesses meet legal competency requirements
- formality_compliance: whether document meets formal requirements for jurisdiction

### C2 — Execution Status
- revocation_clause: whether prior wills and codicils are explicitly revoked
- codicils_present: any codicils attached or referenced — dates and scope
- prior_wills_status: status of any prior wills identified

### C3 — Revocation
- revocation_status: whether this will has been revoked by a subsequent document
- revocation_instrument: document or event that revoked this will if applicable

---

## D — Comparison & Change

### D1 — Version Diff
- changed_sections: sections that differ between two versions
- added_provisions: provisions present in new version absent in prior
- removed_provisions: provisions present in prior version absent in new

### D2 — Beneficiary Delta
- added_beneficiaries: persons added as beneficiaries in newer version
- removed_beneficiaries: persons removed as beneficiaries in newer version
- modified_entitlements: changes to what existing beneficiaries receive

### D3 — Clause Diff
- changed_clauses: clauses added, removed, or modified between versions
- changed_executor: changes to executor appointments between versions

---

## E — Summarisation

### E1 — Full Summary
- full_summary: complete structured summary — all parties, assets, conditions, clauses

### E2 — Person-Scoped Summary
- person_role: role of the named person in this will
- person_entitlement: what the named person is entitled to receive
- person_conditions: conditions attached to this person's role or entitlement

### E3 — Estate Value Summary
- estate_composition: types of assets comprising the estate
- estimated_distribution: how the estate is proportionally distributed across beneficiaries

---

## F — Lifecycle & Status

### F1 — Probate
- probate_status: whether this will has entered probate
- probate_jurisdiction: court and jurisdiction handling probate
- probate_filing_date: date probate was filed if available

### F2 — Execution Status
- executor_appointment_status: whether executor has been formally appointed
- letters_testamentary: whether letters testamentary have been issued

### F3 — Contest
- contest_status: whether the will is being or has been contested
- contest_parties: persons contesting the will
- contest_grounds: legal basis for the contest
