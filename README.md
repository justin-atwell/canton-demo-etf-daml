# canton-demo-etf-daml

Production-grade Daml smart contract layer for a Canton Network ETF lifecycle management platform. Demonstrates LDAP-style enterprise identity and access control expressed as first-class ledger objects, governing real ETF operations — collateral, cap tables, rebalancing, market data, and prime brokerage.

Built on Canton DevNet with SV sponsorship from the Canton Foundation.

---

## The Core Thesis

LDAP was built for a world where one institution owns the directory. Tokenized assets blow that perimeter apart. When a fund manager, custodian, compliance officer, prime broker, hedge fund, and market maker are all peers on a shared network, no single party owns identity — and no single party unilaterally controls an asset that others have a legitimate claim on.

This platform solves that by making authorization a property of the asset itself — enforced at the contract layer, not the application layer.

- `RoleMembership` existing on the ledger **is** the role grant
- Archiving it **is** the revocation
- No database flags. No soft deletes. No admin who can quietly flip a row.

The Prime Brokerage module pushes this further: a `LiquidationWaterfall` can only
liquidate a `CollateralPool` if every signatory of that pool — both the custodian
holding the assets and the hedge fund that posted them — co-authorizes the
transaction that touches it. The ledger enforces this even when the business
intent (liquidate a defaulted account) is unilateral from the prime broker's
point of view. Authorization follows contract stakeholdership, not job title.

---

## Contract Inventory

### IAM Layer
| Contract | Purpose |
|---|---|
| `DirectoryEntry` | Maps an LDAP DN to a Canton party. Identity anchor for all downstream contracts. |
| `RoleMembership` | One contract per (party, role) pair. Archiving = revocation. |
| `AccessEvent` | Immutable append-only audit log. Written on every auth decision, granted or denied. |

### ETF Structure
| Contract | Purpose |
|---|---|
| `ETFDefinition` | Master ETF record. Created by FundManager, observable by all parties. |
| `Constituent` | Single basket holding with weight. Supports UpdateWeight for rebalancing. |
| `CapTable` | Share issuance and redemption ledger. Joint signatory between FundManager and Custodian. |

### Pricing
| Contract | Purpose |
|---|---|
| `NAV` | Immutable daily NAV record posted by FundManager. Observable by Auditor. |
| `NBBOQuote` | Oracle-posted NBBO from Polygon.io. Consumed by the FIX execution engine. |

### Collateral Management
| Contract | Purpose |
|---|---|
| `CollateralAccount` | Custodian-held account tracking available collateral balance. |
| `CollateralLock` | Encumbered collateral pending release or liquidation. |
| `CollateralRelease` | Immutable audit record of a successful lock release. |
| `HaircutSchedule` | Compliance-managed table of haircut rates per asset class. |
| `MarginCall` | Issued by Custodian when collateral coverage drops below threshold. |
| `LiquidationOrder` | Terminal action when margin call goes unmet. |

### Rebalancing
| Contract | Purpose |
|---|---|
| `RebalanceProposal` | FundManager proposes new constituent weights. Requires ComplianceOfficer approval. |
| `RebalanceExecution` | Immutable record of an executed rebalance. Created only from an Approved proposal. |

### Prime Brokerage
| Contract | Purpose |
|---|---|
| `CollateralPool` | Source of truth for a hedge fund's posted collateral. Signatories: `hedgeFund, custodian`. Observer: `riskManager`. Position changes are hedge-fund-controlled; revaluation is custodian-controlled. |
| `CollateralEligibility` | Eligibility rules and haircut schedule for a given asset class, scoped to prime brokerage rather than the fund-level `HaircutSchedule`. |
| `SubstitutionRequest` | Three-party workflow letting a hedge fund swap one posted asset for another. State machine: `PENDING → APPROVED_BY_BROKER → COMPLETED`, with `confirmTransfer` atomically archiving the old `CollateralPool` and creating the updated one. |
| `MarginCallV2` | Margin call with a response deadline and automatic default path. Sole signatory: `primeBroker`. Drives `Issued → ResponseReceived → Satisfied \| Defaulted`. |
| `LiquidationWaterfall` | Priority-ordered liquidation (MMF → Treasury → BTC) of a `CollateralPool` against a defaulted `MarginCallV2`. Controller on `ExecuteWaterfall`: `primeBroker, hedgeFund, custodian` — the latter two are required because the choice fetches, archives, and re-creates the `CollateralPool`, and Daml requires every stakeholder of a touched contract to be an authorizer of the action, not merely able to see it. |
| `LiquidationAuditEvent` | Immutable, append-only record of a single liquidated position. One created per liquidation step. Signatory: `primeBroker`. Observers: `hedgeFund, riskManager, custodian`. |

---

## Roles

| Role | Responsibilities |
|---|---|
| `FundManager` | Creates ETFs, posts NAV, proposes and executes rebalances |
| `Custodian` | Issues/redeems shares, manages collateral accounts, issues margin calls, revalues prime brokerage collateral pools, co-authorizes liquidation |
| `ComplianceOfficer` | Approves/rejects rebalances, suspends ETFs, manages haircut schedule |
| `Auditor` | Read-only observer across all contracts. Receives all AccessEvents. |
| `MarketMaker` | Posts NBBO quotes, submits FIX orders |
| `HedgeFund` | Posts and manages prime brokerage collateral, responds to margin calls, co-authorizes liquidation of its own defaulted positions |
| `PrimeBroker` | Issues and manages margin calls, initiates and executes liquidation waterfalls |
| `RiskManager` | Observer across prime brokerage contracts — collateral pools, margin calls, and liquidation activity |

---

## Regulatory Coverage

| Contract | Regulation | Requirement Satisfied |
|---|---|---|
| `AccessEvent` | FINRA Rule 3110 | Supervision and immutable audit trail |
| `RebalanceProposal` | SEC Rule 38a-1 | Compliance approval workflow |
| `NAV` | SEC Rule 22c-1 | Daily NAV pricing requirement |
| `CollateralLock` / `HaircutSchedule` | Basel III | Collateral management and haircut enforcement |
| `NBBOQuote` / `ExecutionReport` | Reg SHO | Best execution compliance |
| `AccessEvent` / `RebalanceExecution` | SEC Rule 17a-4 | Immutable records retention |
| `LiquidationAuditEvent` | FINRA Rule 4210 / SEC Rule 17a-4 | Immutable margin and liquidation audit trail |
| `MarginCallV2` | FINRA Rule 4210 | Margin call issuance, response window, and default handling |

---

## Prerequisites

```bash
# Install Daml SDK 3.4.11
curl -sSL https://get.daml.com/ | sh -s 3.4.11

# Verify
daml version

# Java 21 required for Daml script service
java -version
```

---

## Build

```bash
daml build
# Output: .daml/dist/canton-demo-etf-0.1.0.dar
```

---

## Test

```bash
# Run all Daml Script tests
daml test --all --no-legacy-assistant-warning
```

> Test count and template count are being updated to reflect the Prime
> Brokerage module (six additional templates: `CollateralPool`,
> `CollateralEligibility`, `SubstitutionRequest`, `MarginCallV2`,
> `LiquidationWaterfall`, `LiquidationAuditEvent`). Prime brokerage logic has
> been verified end-to-end via direct ledger curl testing against a local
> sandbox; Daml Script test coverage for this module is in progress.

---

## Deploy to Canton DevNet

```bash
# Upload DAR to DevNet participant node
daml ledger upload-dar \
  --host <devnet-participant-host> \
  --port 6865 \
  .daml/dist/canton-demo-etf-0.1.0.dar

# Run a script against DevNet
daml script \
  --ledger-host <devnet-participant-host> \
  --ledger-port 6865 \
  --dar .daml/dist/canton-demo-etf-0.1.0.dar \
  --script-name Canton.ETF.Test.IAM.DirectoryEntryTest:testFullLifecycle
```

---

## Java Codegen

Generates typed Java binding classes for use in the Spring Boot API layer:

```bash
daml codegen java \
  --input-file .daml/dist/canton-demo-etf-0.1.0.dar \
  --output-directory ../canton-demo-etf-api/src/main/java/generated
```

Codegen must run with the API stopped — a running Spring Boot process holds
the previously generated classes in memory/classpath, and re-running codegen
without restarting the API afterward will leave it serving stale contract
shapes (most visibly: a constructor that's missing a field you just added).

---

## Project Structure

```
daml/
└── Canton/ETF/
    ├── IAM/
    │   ├── DirectoryEntry.daml
    │   ├── RoleMembership.daml
    │   └── AccessEvent.daml
    ├── Fund/
    │   ├── ETFDefinition.daml
    │   ├── Constituent.daml
    │   └── CapTable.daml
    ├── Pricing/
    │   ├── NAV.daml
    │   └── NBBOQuote.daml
    ├── Collateral/
    │   ├── CollateralAccount.daml
    │   ├── CollateralLock.daml
    │   ├── CollateralRelease.daml
    │   ├── HaircutSchedule.daml
    │   ├── MarginCall.daml
    │   └── LiquidationOrder.daml
    ├── Rebalance/
    │   ├── RebalanceProposal.daml
    │   └── RebalanceExecution.daml
    └── PrimeBrokerage/
        ├── CollateralAsset.daml
        ├── CollateralPool.daml
        ├── CollateralEligibility.daml
        ├── SubstitutionRequest.daml
        ├── MarginCallV2.daml
        └── LiquidationWaterfall.daml
```

---

## Related Repositories

| Repo | Purpose |
|---|---|
| `canton-demo-etf-api` | Java Spring Boot REST API gateway |
| `canton-demo-etf-iam` | LDAP → Auth0 → Canton IAM bridge |
| `canton-demo-etf-fix` | QuickFIX/J execution engine + Polygon.io NBBO oracle |
| `canton-demo-etf-infra` | GCP GKE Terraform + Helm charts |
| `canton-demo-etf-ui` | React frontend with compliance panel |

---

## Built By

Justin Atwell — Director-level solutions engineering and developer relations leader with production DLT experience at Hedera Hashgraph (Avery Dennison atma.io — billions of mainnet transactions) and traditional capital markets experience at Edward Jones (FIX connectivity, NBBO, T+1 settlement).

[LinkedIn](https://www.linkedin.com/in/justin-atwell)