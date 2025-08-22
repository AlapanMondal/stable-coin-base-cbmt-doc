# Fireblocks-CPN Integration Sequence Diagrams

## 1. Complete Cross-Border Payment Flow

This diagram shows the end-to-end process from payment initiation to final settlement, highlighting the interaction between Fireblocks custody and CPN network.

```mermaid
sequenceDiagram
  participant S as Sender
  participant UI as Frontend App
  participant API as Remittance API
  participant FB as Fireblocks Service
  participant CPN as CPN Service
  participant DB as Database
  participant FBNet as Fireblocks Network
  participant CPNNet as CPN Network
  participant BFI as Beneficiary FI
  participant R as Receiver

  autonumber
  Note over S, R: Phase 1: Payment Initiation & Quote Generation
  S ->> UI: Enter payment details (amount: $1000, destination: Philippines)
  UI ->> API: POST /api/remittance/quote
  par Liquidity Check
    API ->> FB: getCPNLiquidityBalance(vaultId)
    FB ->> FBNet: GET /v1/vault/accounts/{vaultId}/assets/USDC_ETH_TEST5_AN74
    FBNet -->> FB: { available: "5000.00", total: "5000.00" }
    FB -->> API: Sufficient USDC liquidity confirmed
  and BFI Discovery
    API ->> CPN: discoverMembers({ country: "PH", currency: "PHP" })
    CPN ->> CPNNet: GET /v1/members?country=PH&currency=PHP
    CPNNet -->> CPN: [{ id: "bfi-ph-001", name: "Philippine Bank" }]
    CPN -->> API: Available BFI partners
  end
  API ->> CPN: requestQuote({ amount: "1000", from: "USD", to: "PHP" })
  CPN ->> CPNNet: POST /v1/quotes/request
  Note over CPNNet: Real-time FX rate discovery across BFIs
  CPNNet -->> CPN: [{ rate: 56.80, fees: { total: "15.00" }, expires: "2025-01-15T10:30:00Z" }]
  CPN -->> API: Best quote selected
  API ->> DB: saveQuote(quoteId, terms, expiration)
  API -->> UI: Quote response (₱56,800 PHP, fees: $15)
  UI -->> S: Display quote with 15-min expiration
  Note over S, R: Phase 2: Payment Authorization & CPN Transaction Creation
  S ->> UI: Accept quote & provide recipient details
  UI ->> API: POST /api/remittance/send (quoteId, travelRuleData)
  API ->> DB: validateQuote(quoteId)
  API ->> CPN: automateComplianceScreening(travelRuleData)
  Note over CPN: AML/Sanctions screening
  CPN -->> API: { amlResult: "passed", sanctionsResult: "passed", riskScore: 25 }
  API ->> CPN: acceptQuote(quoteId, travelRuleData)
  CPN ->> CPNNet: POST /v1/transactions/create
  Note over CPNNet: Create payment request with encrypted Travel Rule data
  CPNNet ->> BFI: Payment notification with compliance data
  BFI -->> CPNNet: Payment accepted, awaiting settlement
  CPNNet -->> CPN: { transactionId: "cpn_tx_12345", status: "pending_settlement" }
  CPN -->> API: CPN transaction created
  API ->> DB: createTransaction(cpnTxId, fireblocksData, status: "pending")
  Note over S, R: Phase 3: Blockchain Settlement via Fireblocks
  API ->> FB: executeCPNSettlement(vaultId, destinationAddress, amount, cpnTransactionId)
  FB ->> FBNet: POST /v1/transactions
  Note over FBNet: MPC signing for USDC transfer
  FBNet ->> FBNet: Multi-party computation signature
  FBNet ->> FBNet: Broadcast to Ethereum network
  FBNet -->> FB: { id: "fb_tx_67890", status: "SUBMITTED", txHash: "0xabc123..." }
  FB -->> API: Settlement transaction initiated
  API ->> DB: updateTransaction(txId, fireblocksId, txHash)
  API ->> CPN: confirmSettlement(cpnTxId, txHash)
  CPN ->> CPNNet: POST /v1/transactions/{cpnTxId}/settlement
  CPNNet -->> CPN: Settlement confirmed, monitoring blockchain
  API -->> UI: { status: "settlement_pending", txHash: "0xabc123..." }
  UI -->> S: Payment submitted, blockchain confirmation tracking
  Note over S, R: Phase 4: Confirmation & Final Settlement
  FBNet ->> FBNet: Monitor blockchain confirmation (12 blocks)
  FBNet ->> API: Webhook: TRANSACTION_STATUS_UPDATED (COMPLETED)
  API ->> DB: updateTransaction(txId, status: "blockchain_confirmed")
  API ->> CPN: notifyBlockchainConfirmation(cpnTxId)
  CPN ->> CPNNet: PUT /v1/transactions/{cpnTxId}/confirm
  CPNNet ->> BFI: Settlement confirmed on blockchain
  BFI ->> BFI: Convert USDC to PHP (₱56,800)
  BFI ->> R: Deposit ₱56,800 to recipient account
  BFI -->> CPNNet: Payment delivered to recipient
  CPNNet -->> CPN: { status: "completed", deliveredAmount: "56800.00 PHP" }
  CPN -->> API: Final payment confirmation
  API ->> DB: updateTransaction(txId, status: "completed", deliveredAt: "2025-01-15T10:45:00Z")
  API -->> UI: Payment completed notification
  UI -->> S: ✅ Payment successful - ₱56,800 delivered
```
## 2. Liquidity Management & Vault Operations

This diagram shows how the system manages USDC liquidity across multiple Fireblocks vaults for optimal CPN operations.

```mermaid
sequenceDiagram
    participant Admin as Admin Dashboard
    participant LM as Liquidity Manager
    participant API as Management API
    participant FB as Fireblocks Service
    participant FBNet as Fireblocks Network
    participant CPN as CPN Service
    participant Alert as Alert System

    autonumber
    Note over Admin,Alert: Automated Liquidity Monitoring

    loop Every 5 minutes
        LM->>FB: getAllCPNVaultBalances()
        FB->>FBNet: GET /v1/vault/accounts (CPN vaults)
        FBNet-->>FB: [{ id: "vault_main", balance: "50000" }, { id: "vault_settlement", balance: "5000" }]
        FB-->>LM: Aggregate liquidity data
        LM->>LM: Calculate total available USDC
        LM->>CPN: updateAvailableLiquidity(totalUSDC)
        CPN->>CPN: Adjust quote generation limits

        alt Low Liquidity Warning (< $10,000)
            LM->>Alert: triggerLowLiquidityAlert()
            Alert->>Admin: 🚨 Low USDC liquidity warning
        else Critical Liquidity (< $5,000)
            LM->>Alert: triggerCriticalAlert()
            Alert->>Admin: 🔴 CRITICAL: Suspend new payments
            LM->>CPN: suspendQuoteGeneration()
        end
    end

    Note over Admin,Alert: Manual Liquidity Rebalancing

    Admin->>API: Request vault rebalancing
    API->>FB: getVaultBalances([vault_ids])
    FB-->>API: Current distribution
    API->>Admin: Show options
    Admin->>API: Approve transfer (vault_main to vault_settlement, $20,000)
    API->>FB: transferUSDC(fromVaultId, toVaultId, amount)
    FB->>FBNet: POST /v1/transactions (internal transfer)
    FBNet->>FBNet: Internal vault transfer (instant)
    FBNet-->>FB: Transfer completed
    FB-->>API: Rebalancing successful
    API->>LM: triggerBalanceRefresh()
    LM->>CPN: updateAvailableLiquidity(newTotal)
    API-->>Admin: ✅ Liquidity rebalanced

    Note over Admin,Alert: Automated Replenishment

    alt Integration with Circle Mint API
        LM->>API: requestUSDCMinting($100,000)
        API->>API: Initiate wire transfer to Circle
        API->>API: Await mint confirmation
        API->>FB: createVaultWallet(mainVault, "USDC_ETH_TEST5_AN74")
        FB-->>API: New vault address
        API-->>LM: Replenishment initiated
    end
```
## 3. Webhook Coordination & Real-time Updates

This diagram illustrates how Fireblocks webhooks coordinate with CPN status updates to provide real-time payment tracking.

```mermaid
sequenceDiagram
    participant FBNet as Fireblocks Network
    participant Webhook as Webhook Handler
    participant DB as Database
    participant CPN as CPN Service
    participant CPNNet as CPN Network
    participant SSE as Server-Sent Events
    participant UI as User Interface
    participant Mobile as Mobile App

    autonumber
    Note over FBNet,Mobile: Real-time Transaction Status Updates

    FBNet->>Webhook: POST /api/webhook/fireblocks (CONFIRMING, txHash, blockConfirmations)
    Webhook->>Webhook: Verify webhook signature
    Webhook->>DB: getTransactionByCriteria(fireblocksId)
    DB-->>Webhook: { id, cpnTransactionId, userId }

    par Update Internal State
        Webhook->>DB: updateTransaction(id, status: confirming, blockConfirmations, lastUpdated)
    and Notify CPN Network
        Webhook->>CPN: updateSettlementStatus(cpnTransactionId, status: confirming, txHash, confirmations)
        CPN->>CPNNet: PUT /v1/transactions/cpnTransactionId/status
        CPNNet-->>CPN: Status updated
    and Real-time User Notifications
        Webhook->>SSE: broadcastToUser(userId, type: PAYMENT_UPDATE, confirmations)
        SSE-->>UI: Live status update
        SSE-->>Mobile: Push notification
    end

    Note over FBNet,Mobile: Transaction Completion Flow

    FBNet->>Webhook: POST /api/webhook/fireblocks (COMPLETED, blockConfirmations)
    Webhook->>DB: updateTransaction(id, status: blockchain_confirmed)
    Webhook->>CPN: confirmBlockchainSettlement(cpnTransactionId)
    CPN->>CPNNet: POST /v1/transactions/cpnTransactionId/finalize
    CPNNet->>CPNNet: Trigger final settlement with BFI
    CPNNet-->>CPN: { status: completed, deliveredAt }
    CPN-->>Webhook: Final payment confirmation
    Webhook->>DB: updateTransaction(id, status: completed, deliveredAt, finalAmount)
    Webhook->>SSE: broadcastToUser(userId, type: PAYMENT_COMPLETED, message)
    UI->>UI: Show success animation
    Mobile->>Mobile: Send push notification

    Note over FBNet,Mobile: Error Handling & Recovery

    alt Transaction Failed
        FBNet->>Webhook: status: FAILED, error: Insufficient gas
        Webhook->>DB: updateTransaction(id, status: failed, error)
        Webhook->>CPN: reportSettlementFailure(cpnTransactionId, error)
        CPN->>CPNNet: POST /v1/transactions/cpnTransactionId/fail
        CPNNet-->>CPN: Initiate reversal
        Webhook->>SSE: broadcastToUser(userId, type: PAYMENT_FAILED)
    else Transaction Cancelled
        Webhook->>CPN: cancelCPNTransaction(cpnTransactionId)
        CPN->>CPNNet: DELETE /v1/transactions/cpnTransactionId
        Webhook->>DB: updateTransaction(id, status: cancelled)
    end
```
## 4. Compliance & Travel Rule Integration

This diagram shows how the system handles regulatory compliance requirements through the CPN network while maintaining Fireblocks custody.

```mermaid
sequenceDiagram
    participant User as Sender
    participant API as Compliance API
    participant CPN as CPN Service
    participant CPNNet as CPN Network
    participant BFI as Beneficiary FI
    participant AML as AML Provider
    participant Sanctions as Sanctions DB
    participant TravelRule as Travel Rule Service
  
    autonumber
    Note over User,TravelRule: Enhanced Due Diligence

    User->>API: Initiate $15,000 payment
    API->>API: Detect high-value transaction
    API->>CPN: requestEnhancedCompliance(amount, corridor)

    par AML Screening
        CPN->>AML: screenCustomer(name, address, id, amount)
        AML->>AML: Check watchlists
        AML-->>CPN: { result: CLEAR, riskScore, reportId }
    and Sanctions Screening
        CPN->>Sanctions: checkSanctionsList(fullName, country, beneficiary, beneficiaryCountry)
        Sanctions-->>CPN: { result: NO_MATCH, confidence }
    and Travel Rule Compliance
        CPN->>TravelRule: validateTravelRuleData(originatorInfo, beneficiaryInfo, amount, currency)
        TravelRule->>TravelRule: Validate jurisdiction fields
        TravelRule-->>CPN: { compliant: true, requiredFields }
    end

    CPN-->>API: complianceStatus: ADDITIONAL_INFO_REQUIRED, missingFields, amlCleared, sanctionsCleared
    API-->>User: Request additional information
    User->>API: Provide purpose
    API->>CPN: submitAdditionalComplianceData(transactionPurpose, relationshipToBeneficiary)
    CPN->>TravelRule: finalizeCompliancePackage()
    TravelRule-->>CPN: status: COMPLIANT, package

    Note over User,TravelRule: CPN Transaction with Enhanced Compliance

    CPN->>CPNNet: createHighValueTransaction(amount, compliancePackage, enhancedDueDiligence)
    CPNNet->>BFI: transmitComplianceData(package)
    BFI->>BFI: Decrypt, validate, local compliance
    BFI-->>CPNNet: complianceApproved, readyForSettlement
    CPNNet-->>CPN: transactionId, status: APPROVED
    CPN-->>API: Transaction approved

    Note over User,TravelRule: Ongoing Compliance Monitoring

    loop Transaction Monitoring
        CPN->>AML: reportTransactionProgress(transactionId, status, timestamp)
        alt Suspicious Activity Detected
            AML->>CPN: alert: VELOCITY_CHECK, recommendation: MANUAL_REVIEW
            CPN->>API: suspendTransaction(transactionId)
            API->>API: Queue for team review
        else Normal Processing
            AML-->>CPN: status: MONITORING, noAlerts: true
        end
    end

    Note over User,TravelRule: Post-Settlement Reporting

    CPN->>TravelRule: reportCompletedTransaction(transactionId, completedAt, finalAmount, deliveredAmount)
    TravelRule->>TravelRule: Generate regulatory reports (CTR, SAR if needed)
    TravelRule-->>CPN: Compliance reporting completed
    CPN-->>API: Transaction fully compliant and reported

```
## Key Integration Points Explained

### 1. **Fireblocks as Custody Layer**
- **Vault Management**: Each CPN operation uses dedicated Fireblocks vaults for USDC custody
- **MPC Signing**: All blockchain transactions are signed using Fireblocks' MPC technology
- **Balance Monitoring**: Real-time USDC balance checks before quote generation
- **Settlement Execution**: Fireblocks handles the actual blockchain settlement

### 2. **CPN as Network Orchestration**
- **BFI Discovery**: Finds optimal beneficiary financial institutions
- **Quote Aggregation**: Collects competitive rates from multiple BFIs
- **Compliance Automation**: Handles AML/sanctions screening and Travel Rule compliance
- **Settlement Coordination**: Manages the end-to-end payment lifecycle

### 3. **Webhook Coordination**
- **Status Synchronization**: Fireblocks webhooks update CPN transaction status
- **Real-time Updates**: Users receive live payment progress notifications
- **Error Handling**: Failed transactions trigger appropriate recovery mechanisms
- **Compliance Reporting**: Automated regulatory reporting throughout the process

### 4. **Liquidity Management**
- **Multi-Vault Strategy**: Separate vaults for different operational purposes
- **Automated Monitoring**: Continuous balance checking and rebalancing
- **Alert Systems**: Proactive notifications for low liquidity conditions
- **Integration Ready**: Prepared for Circle Mint API integration for USDC replenishment

This architecture ensures that Fireblocks provides enterprise-grade custody while CPN enables global reach and regulatory compliance for cross-border payments.
