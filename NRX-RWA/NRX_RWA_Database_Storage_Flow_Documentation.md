# NRX RWA Database Storage Flow Documentation

This document provides a comprehensive, end-to-end technical reference for the data lifecycles, cross-service synchronization, and blockchain event processing layers across the NRX Real World Asset (RWA) platform, structured by platform actors and mapped directly to database collections and schema keys.

---

## Master Persistence & CRUD Mappings (RWA Platform)

The following master reference table maps every database operation, linking logical actor-based lifecycles to MongoDB collection queries/updates, database boundaries, and blockchain events.

| Actor | Flow / Lifecycle Step | Database Layer | MongoDB Collection | CRUD Operation | Key Database Fields | Blockchain Event / Queue |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Investor** | Investor Registration | Auth DB (Central) <br> RWA DB (Local) | `investors` (Auth) <br> `investors` (RWA) | Insert (Auth) <br> Upsert (RWA Sync) | Insert: `email`, `userName`, `password`, `globalId` <br> Sync: `globalId`, `userName`, `email`, `status` | None |
| **Investor** | Investor Login & 2FA | Auth DB (Central) | `investors` | Read & Update | Read: `email`, `password`, `twoFaEnabled`, `twoFaSecret` \| Update: `tempToken`, `tempTokenExpiresAt` | None |
| **Investor** | SIWE Wallet Connection | RWA DB (Local) <br> Auth DB (Central) | `investors` (RWA) <br> `investors` (Auth) | Update (RWA) <br> Update (Auth Sync) | RWA: `walletAddress`, unsets `walletNonce`, `walletNonceExpiresAt` <br> Auth: `walletAddress` mapped to `globalId` | None |
| **Investor** | KYC Status Update | Auth DB (Central) | `investors` | Update | `KYCVerified`, `kycReviewedAt` | None |
| **Investor** | Profile Settings Update | RWA DB (Local) | `investors` | Update | `phoneNumber`, `address`, `city`, `country`, `postalCode` | None |
| **Investor** | Contact Support Request | RWA DB (Local) | `investorcontactsupports` | Insert | `investorId`, `name`, `email`, `subject`, `message`, `status` (pending) | None |
| **Investor** | Account Deletion Request | RWA DB (Local) | `investordeleteds` | Insert | `investorId`, `email`, `reason`, `status` (pending) | None |
| **Investor** | Investment Creation | RWA DB (Local) | `assetinvestments` <br> `assetinvestmentdocuments` | Insert (Investment) <br> Insert (Document) | Investment: `investorId`, `assetId`, `amountInvested`, `sharedQuantity`, `status` (pending), `holdingPeriod` <br> Document: `assetInvestmentId`, `documentUrl` | AWS S3 Upload (Mutual Agreement PDF) |
| **Investor** | Investment Approve (On-Chain Approve) | RWA DB (Local) | `assetinvestments` <br> `assetmanagements` | Update (Investment) <br> Update (Asset Metrics) | Investment: `status` (approved), `transactionHash`, `maturityDate`, `investmentId`, `ownershipPercentage` <br> Asset: `remainingTokens`, `progressBar`, `assetStatus` | `InvestmentApproved` (Contract) <br> Fireblocks Vault Import |
| **Investor** | Asset Completion (On-Chain Complete) | RWA DB (Local) | `assetinvestments` | Read & Update | Read: `assetId`, `investorId` \| Update: `maturityEmailSent`, `profitAmount`, `profitPercentage`, `maturityDate` | `AssetCompleted` (Contract) |
| **Investor** | Buyback Request (On-Chain Request) | RWA DB (Local) | `assetinvestments` | Read & Update | Read: `investmentId`, `investorId` \| Update: `withdrawalRequestId`, `withdrawalRequestedAmount`, `withdrawalStatus` (requested) | `BuyBackRequested` (Contract) |
| **Investor** | Buyback Approval (On-Chain Approve) | RWA DB (Local) | `assetinvestments` <br> `assetyieldmanagements` | Update (Investment) <br> Update (Yields) | Investment: `withdrawalStatus` (approved), `status` (completed), `withdrawalAmount`, `ownershipPercentage` (0) <br> Yields: `status` (completed), `isWithdrawn` (true) | `BuyBackApproved` (Contract) |
| **Investor** | Yield Claim / Withdrawal | RWA DB (Local) | `assetyieldmanagements` | Update & Upsert | Update: `isWithdrawn` (true), `status` (completed) on rental yields \| Upsert: Audit record with `yieldType` (withdraw) | None |
| **Investor** | Dividend Claiming | RWA DB (Local) | `dividendfundmanagements` | Update | `status` (completed), `transactionHash` | None |
| **Creator** | Creator Registration | RWA DB (Local) | `users` | Insert | `userName`, `email`, `password`, `status` (1), `nonce` (0), `twoFaEnabled` (false) | None |
| **Creator** | Creator Login & 2FA | RWA DB (Local) | `users` | Read & Update | Read: `email`, `password`, `twoFaEnabled` \| Update: `tempToken`, `tempTokenExpiresAt` | None |
| **Creator** | SIWE Wallet Connection | RWA DB (Local) | `users` | Update | `walletAddress`, unsets `walletNonce`, `walletNonceExpiresAt` | None |
| **Creator** | Profile Settings Update | RWA DB (Local) | `users` | Update | `phoneNumber`, `address`, `city`, `country`, `postalCode` | None |
| **Creator** | Contact Support Request | RWA DB (Local) | `usercontactsupports` | Insert | `userId`, `name`, `email`, `subject`, `message`, `status` (pending) | None |
| **Creator** | Account Deletion Request | RWA DB (Local) | `userdeleteds` | Insert | `userId`, `email`, `reason`, `status` (pending) | None |
| **Creator** | Asset Registration | RWA DB (Local) | `assetmanagements` | Insert | `creatorId`, `categoryName`, `assetManagementStatus` (0/draft), `progressBar` (0), `basicInformation` | None |
| **Admin** | Admin Login & 2FA | RWA DB (Local) | `admins` | Read & Update | Read: `email`, `password`, `twoFactorEnabled`, `twoFactorSecret` \| Update: `lastLoginAt`, `jwtTokens` | None |
| **Admin** | Admin Wallet Auth | RWA DB (Local) | `admins` | Read & Update | Read: `walletAddress`, `walletNonce`, `walletNonceExpiresAt` \| Update: `walletNonce` (nullified), `jwtTokens` | None |
| **Admin** | Resolve Investor Support ticket | RWA DB (Local) | `investorcontactsupports` | Update | `status` (resolved), `adminNote`, `resolvedAt` | None |
| **Admin** | Resolve Creator Support ticket | RWA DB (Local) | `usercontactsupports` | Update | `status` (resolved), `adminNote`, `resolvedAt` | None |
| **Admin** | Approve Investor Account Deletion | RWA DB (Local) | `investordeleteds` <br> `investors` | Update (Request) <br> Update (Profile) | Request: `status` (deleted), `deletedAt` <br> Profile: `status` (0 / Suspended) | None |
| **Admin** | Approve Creator Account Deletion | RWA DB (Local) | `userdeleteds` <br> `users` | Update (Request) <br> Update (Profile) | Request: `status` (deleted), `deletedAt` <br> Profile: `status` (0 / Suspended) | None |
| **Admin** | Asset Listing Approval | RWA DB (Local) | `assetmanagements` | Update | `AdminId`, `assetManagementStatus` (2/approved), `securityToken`, `transactionHash`, `assetStatus` | None |
| **Admin** | Asset Completion Ledger | RWA DB (Local) | `assetmanagements` <br> `assetmanagementcompletes` | Update (Asset) <br> Upsert (Ledger) | Asset: `assetStatus` (Closed), `assetCompletionStatus` (1) <br> Complete: `creatorId`, `assetId`, `assetCompletionDate`, `assetFinalAmount`, `profitGenerated` | `AssetCompleted` (Contract) |
| **Admin** | Rental Yield Allocation | RWA DB (Local) | `assetyieldmanagements` | Upsert | `AssetId`, `investerId`, `netAmount`, `netAmountDecimals`, `sharesHolding`, `yieldMonth`, `yieldYear`, `yieldType` (rental) | None |
| **Admin** | Dividend Distribution Setup | RWA DB (Local) | `dividendfundmanagements` | Upsert | `assetId`, `investorId`, `dividendAmount`, `dividendPercentage`, `payoutDate`, `status` (pending) | None |
| **Admin** | Site Settings Configuration | RWA DB (Local) | `site_settings` | Upsert | `maintenanceMode`, `allowedCountries`, `feeStructure`, `adminId` | None |
| **Admin** | Admin Panel Config | RWA DB (Local) | `adminsettings` | Upsert | `themePreference`, `notificationSettings`, `twoFactorConfig` | None |
| **Admin** | Block Checkpoint | RWA DB (Local) | `listenerblockcheckpoints` | Update ($max) | `blockNumber` (atomic monotonic update) | None |

---

## 1. Actor: Investor Flows

### 1.1 Investor Registration & Synchronization
When a new investor registers, their credentials and global identifiers are established centrally in the Auth DB before synchronizing downstream to the local RWA DB.

#### Database Operations & State Lifecycle
1. **Duplicate Account Check (Auth DB)**:
   * **Database**: Central Identity Database
   * **Collection**: `investors`
   * **Operation**: Read (`findOne`)
   * **Key Fields**:
     * `email`: Checked to verify no duplicate account exists.
     * `userName`: Checked to verify handle uniqueness.
2. **Central Account Insertion (Auth DB)**:
   * **Database**: Central Identity Database
   * **Collection**: `investors`
   * **Operation**: Insert (`insertOne`)
   * **Key Fields Stored**:
     * `email`: User login email.
     * `userName`: Chosen username.
     * `password`: Bcrypt hashed password.
     * `globalId`: Auto-generated UUID/ObjectId reference.
     * `twoFaEnabled`: Hardcoded to `false` on registration.
     * `status`: Hardcoded to `1` (Active).
     * `registeredPlatform`: Stored as `"NRX-RWA"` to track platform origin.
3. **Local Profile Sync (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `investors`
   * **Operation**: Upsert (`findOneAndUpdate` with `{ upsert: true }`) matching the unique `globalId`
   * **Key Fields Stored**:
     * `globalId`: Links the local record directly to the Auth DB credentials.
     * `email`: Synced from the Auth DB.
     * `userName`: Synced from the Auth DB.
     * `status`: Set to `1` (Active).

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Auth DB** | `investors` | `email` | String | Insert / Read | Investor's unique identifier/email. |
| **Auth DB** | `investors` | `userName` | String | Insert / Read | Unique display handle. |
| **Auth DB** | `investors` | `password` | String | Insert | Bcrypt-hashed login credential. |
| **Auth DB** | `investors` | `globalId` | UUID / String | Insert / Sync | Centralized ID linking all platform databases. |
| **Auth DB** | `investors` | `twoFaEnabled` | Boolean | Insert | Set to `false` on initialization. |
| **Auth DB** | `investors` | `status` | Number | Insert | Set to `1` (active) on initialization. |
| **Auth DB** | `investors` | `registeredPlatform` | String | Insert | Set to `"NRX-RWA"`. |
| **RWA DB** | `investors` | `globalId` | UUID / String | Upsert Match | Matches sync key. |
| **RWA DB** | `investors` | `email` | String | Update | Copy of central login email. |
| **RWA DB** | `investors` | `userName` | String | Update | Copy of central display handle. |
| **RWA DB** | `investors` | `status` | Number | Update | Local status initialized to `1`. |

---

### 1.2 Investor Login & 2FA Flow
Authenticating investor accounts, verifying passwords, and enforcing temporary Multi-Factor session states.

#### Database Operations & State Lifecycle
1. **Credentials Retrieval (Auth DB)**:
   * **Database**: Central Identity Database
   * **Collection**: `investors`
   * **Operation**: Read (`findOne`) matching the submitted email.
   * **Key Fields Read**:
     * `email`: Query filter.
     * `password`: Fetched to verify against user input using bcrypt comparison.
     * `twoFaEnabled`: Checked to verify if 2FA verification is required.
2. **Temporary 2FA Session Creation (Auth DB - If 2FA is active)**:
   * **Database**: Central Identity Database
   * **Collection**: `investors`
   * **Operation**: Update (`updateOne`)
   * **Key Fields Updated**:
     * `tempToken`: Writes a short-lived JSON Web Token string hash.
     * `tempTokenExpiresAt`: Writes a timestamp defining the 10-minute expiry window.
3. **2FA Verification (Auth DB - If 2FA is active)**:
   * **Database**: Central Identity Database
   * **Collection**: `investors`
   * **Operation**: Read (`findOne`) matching the verification token.
   * **Key Fields Read**:
     * `tempToken`: Query criteria.
     * `tempTokenExpiresAt`: Read to confirm the validation window is still active.
     * `twoFaSecret`: Stored TOTP secret used to check the user's OTP code.
4. **2FA Session Cleanup (Auth DB - If 2FA is active)**:
   * **Database**: Central Identity Database
   * **Collection**: `investors`
   * **Operation**: Update (`updateOne`)
   * **Key Fields Updated**:
     * `tempToken`: `$unset` or set to `null` to disable reuse.
     * `tempTokenExpiresAt`: `$unset` or set to `null` to clear.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Auth DB** | `investors` | `email` | String | Read Match | Target matching login identifier. |
| **Auth DB** | `investors` | `password` | String | Read | Stored bcrypt hash for verification. |
| **Auth DB** | `investors` | `twoFaEnabled` | Boolean | Read | Checks if Multi-Factor is enforced. |
| **Auth DB** | `investors` | `twoFaSecret` | String | Read | Enforces TOTP cryptographic checks. |
| **Auth DB** | `investors` | `tempToken` | String | Update / Unset | Ephemeral validation session code. |
| **Auth DB** | `investors` | `tempTokenExpiresAt`| Date | Update / Unset | Validation session timer. |

---

### 1.3 SIWE Wallet Connection Flow
Validating ownership of decentralised addresses on the blockchain and syncing them back to central profile records.

#### Database Operations & State Lifecycle
1. **Nonce Initialization (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `investors`
   * **Operation**: Update (`updateOne`) matching the investor's local `_id`.
   * **Key Fields Updated**:
     * `walletNonce`: Generates and writes a secure cryptographic nonce string.
     * `walletNonceExpiresAt`: Writes an expiry timestamp (5-minute window).
2. **Nonce Verification (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `investors`
   * **Operation**: Read (`findOne`) matching investor `_id`.
   * **Key Fields Read**:
     * `walletNonce`: Compared against the signed message payload.
     * `walletNonceExpiresAt`: Checked to ensure validation timeframe is active.
3. **Local Wallet Registration (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `investors`
   * **Operation**: Update (`updateOne`) matching investor `_id`.
   * **Key Fields Updated**:
     * `walletAddress`: Stores the normalized lowercase Ethereum address.
     * `walletNonce`: `$unset` or cleared.
     * `walletNonceExpiresAt`: `$unset` or cleared.
4. **Central Wallet Synchronization (Auth DB)**:
   * **Database**: Central Identity Database
   * **Collection**: `investors`
   * **Operation**: Update (`updateOne`) matching the `globalId`.
   * **Key Fields Updated**:
     * `walletAddress`: Syncs the Ethereum wallet address to the central record.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **RWA DB** | `investors` | `walletNonce` | String | Update / Unset | Random string used for verification. |
| **RWA DB** | `investors` | `walletNonceExpiresAt`| Date | Update / Unset | Expiration window for wallet connection. |
| **RWA DB** | `investors` | `walletAddress` | String | Update | Confirmed lowercase Ethereum address. |
| **Auth DB** | `investors` | `walletAddress` | String | Update | Synced lowercase address linked via `globalId`. |

---

### 1.4 KYC Verification Webhook
Updates central verification logs when Sumsub returns approval or rejection.

#### Database Operations & State Lifecycle
1. **KYC Verification Update (Auth DB)**:
   * **Database**: Central Identity Database
   * **Collection**: `investors`
   * **Operation**: Update (`findOneAndUpdate`) matching the user's `globalId`.
   * **Key Fields Updated**:
     * `KYCVerified` (or `kycStatus`): Updated to:
       * `1` (Approved) if the Sumsub hook returns `GREEN`.
       * `2` (Rejected) if the Sumsub hook returns `RED`.
     * `kycReviewedAt`: Updated to the current date and time.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Auth DB** | `investors` | `globalId` | UUID / String | Read Match | Filters target profile. |
| **Auth DB** | `investors` | `KYCVerified` | Number | Update | Verification status: `1` (Approved) or `2` (Rejected). |
| **Auth DB** | `investors` | `kycReviewedAt` | Date | Update | Datetime KYC webhook processed. |

---

### 1.5 Profile Settings Update
Allows investors to customize contact and geolocation data.

#### Database Operations & State Lifecycle
1. **Investor Contact Settings Update (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `investors`
   * **Operation**: Update (`updateOne`) matching investor `_id`.
   * **Key Fields Updated**:
     * `phoneNumber`: User contact telephone string.
     * `address`: Resident street address.
     * `city`: Resident city.
     * `country`: Resident country.
     * `postalCode`: Resident area zip code.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **RWA DB** | `investors` | `phoneNumber` | String | Update | Investor's phone number. |
| **RWA DB** | `investors` | `address` | String | Update | Resident street details. |
| **RWA DB** | `investors` | `city` | String | Update | Geolocation city. |
| **RWA DB** | `investors` | `country` | String | Update | Geolocation country. |
| **RWA DB** | `investors` | `postalCode` | String | Update | Geolocation postal/zip code. |

---

### 1.6 Contact Support Request
Allows investors to log troubleshooting issues directly with administrators.

#### Database Operations & State Lifecycle
1. **Support Ticket Registration (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `investorcontactsupports`
   * **Operation**: Insert (`insertOne`)
   * **Key Fields Stored**:
     * `investorId`: Reference link to local investor profile.
     * `name`: Contact name.
     * `email`: Contact email address.
     * `subject`: Brief ticket subject.
     * `message`: Detailed description of the issue.
     * `status`: Set to `"pending"`.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **RWA DB** | `investorcontactsupports` | `investorId` | ObjectId | Insert | Linked profile identifier. |
| **RWA DB** | `investorcontactsupports` | `name` | String | Insert | Customer name. |
| **RWA DB** | `investorcontactsupports` | `email` | String | Insert | Customer notification email. |
| **RWA DB** | `investorcontactsupports` | `subject` | String | Insert | Ticket subject line. |
| **RWA DB** | `investorcontactsupports` | `message` | String | Insert | Detailed description. |
| **RWA DB** | `investorcontactsupports` | `status` | String | Insert | Status initialized to `"pending"`. |

---

### 1.7 Account Deletion Request
Ensures regulatory compliance by permitting investors to trigger soft-deletion queues.

#### Database Operations & State Lifecycle
1. **Compliance Deletion Entry (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `investordeleteds`
   * **Operation**: Insert (`insertOne`)
   * **Key Fields Stored**:
     * `investorId`: Reference link to local investor profile.
     * `email`: Stored email address.
     * `reason`: Explanation text provided by the investor.
     * `status`: Set to `"pending"`.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **RWA DB** | `investordeleteds` | `investorId` | ObjectId | Insert | Linked profile identifier. |
| **RWA DB** | `investordeleteds` | `email` | String | Insert | Account email to flag. |
| **RWA DB** | `investordeleteds` | `reason` | String | Insert | Deletion reasoning statement. |
| **RWA DB** | `investordeleteds` | `status` | String | Insert | Initialized status: `"pending"`. |

---

### 1.8 Investment Creation
Initializes a real world asset tokenization purchase commitment.

#### Database Operations & State Lifecycle
1. **Asset Details Verification (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `assetmanagements`
   * **Operation**: Read (`findById`)
   * **Key Fields Read**:
     * `_id` (assetId): Confirms target listing registry.
     * `holdingPeriod`: Loaded to register lock duration.
     * `pricePerToken` / `numberOfTokens` / `remainingTokens`: Loaded to verify availability and calculate shares.
     * `assetManagementStatus`: Verified to ensure the status is `2` (Approved).
2. **Investment Record Registration (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `assetinvestments`
   * **Operation**: Insert (`insertOne`)
   * **Key Fields Stored**:
     * `investorId`: Local investor reference.
     * `assetId`: Target asset reference.
     * `amountInvested`: Net investment amount.
     * `transactionFee`: Stored calculated service fee.
     * `sharedQuantity`: Count of RWA security tokens purchased.
     * `status`: Hardcoded to `"pending"`.
     * `isInvestorAgreed`: Set to `true`.
     * `investorAgreementDate`: Current date timestamp.
     * `holdingPeriod`: Copied from asset.
3. **Mutual Agreement Registration (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `assetinvestmentdocuments`
   * **Operation**: Insert (`insertOne`)
   * **Key Fields Stored**:
     * `assetInvestmentId`: Linked investment reference.
     * `documentUrl`: Stores the AWS S3 URL to the generated PDF contract.
     * `documentType`: Hardcoded to `"mutual-agreement"`.
     * `investorSignatureUrl`: S3 link storing the png signature image.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **RWA DB** | `assetmanagements` | `_id` | ObjectId | Read Match | Target asset identifier. |
| **RWA DB** | `assetmanagements` | `assetManagementStatus`| Number | Read | Checks that asset is `2` (Approved). |
| **RWA DB** | `assetinvestments` | `investorId` | ObjectId | Insert | Linked investor ID. |
| **RWA DB** | `assetinvestments` | `assetId` | ObjectId | Insert | Linked asset ID. |
| **RWA DB** | `assetinvestments` | `amountInvested` | Number | Insert | Net investment volume. |
| **RWA DB** | `assetinvestments` | `sharedQuantity` | Number | Insert | Count of security tokens. |
| **RWA DB** | `assetinvestments` | `status` | String | Insert | Initialized status: `"pending"`. |
| **RWA DB** | `assetinvestments` | `isInvestorAgreed` | Boolean | Insert | Set to `true`. |
| **RWA DB** | `assetinvestmentdocuments` | `assetInvestmentId` | ObjectId | Insert | Associated investment ID. |
| **RWA DB** | `assetinvestmentdocuments` | `documentUrl` | String | Insert | Mutual agreement contract link. |

---

### 1.9 Investment Approval (`InvestmentApproved` event)
Triggers when the administrator completes transaction registration on-chain, updating ownership variables and decreasing available asset token pools. The blockchain event is parsed by the stateless contract listener and forwarded to the core database backend.

#### Database Operations & State Lifecycle
1. **On-Chain Event Detection**:
   * The stateless **`nrx-rwa-contract-backend`** listener monitors the blockchain.
   * On detecting `InvestmentApproved`, it extracts `investorAddress`, `assetAddress`, `sharedQuantity`, `investmentId`, and `maturityPeriod` and sends an HMAC-SHA256 callback to the core backend.
2. **Investment Verification & Approval (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `assetinvestments`
   * **Operation**: Read & Update (`findOneAndUpdate`)
   * **Key Fields Read**:
     * `assetId` (matched via `assetAddress`), `investorId` (matched via `investorAddress`), `sharedQuantity`, `isInvestorAgreed: true`, and verifies `transactionHash` is empty.
   * **Key Fields Updated**:
     * `status`: Transitions from `"pending"` to `"approved"`.
     * `transactionHash`: Stores the transaction hash confirming on-chain registration.
     * `maturityDate`: Derived and stored from the block's `maturityPeriod`.
     * `investmentId`: Stores the on-chain index ID.
     * `ownershipPercentage`: Calculated and saved (`[amountInvested / (pricePerToken * numberOfTokens)] * 100`).
3. **Asset Allocation Updates (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `assetmanagements`
   * **Operation**: Update (`updateOne`) matching the parent asset.
   * **Key Fields Updated**:
     * `remainingTokens`: Decreased by the purchased `sharedQuantity`.
     * `progressBar`: Recalculated and updated percentage.
     * `assetStatus`: Toggles to `"Closing Soon"` (> 70%) or `"Closed"` (100%).
     * `isInvestmentClosed`: Set to `true` if remaining tokens hit `0`.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **RWA DB** | `assetinvestments` | `status` | String | Update | Transitions to `"approved"`. |
| **RWA DB** | `assetinvestments` | `transactionHash` | String | Update | On-chain verification tx hash. |
| **RWA DB** | `assetinvestments` | `maturityDate` | Date / Number | Update | Sets mature date timestamp. |
| **RWA DB** | `assetinvestments` | `investmentId` | Number | Update | On-chain index identifier. |
| **RWA DB** | `assetinvestments` | `ownershipPercentage`| Number | Update | Calculated share weight. |
| **RWA DB** | `assetmanagements` | `remainingTokens` | Number | Update | Deducts purchased quantities. |
| **RWA DB** | `assetmanagements` | `progressBar` | Number | Update | Recalculates funding progress. |
| **RWA DB** | `assetmanagements` | `assetStatus` | String | Update | Set to `"Closing Soon"` or `"Closed"`. |

---

### 1.10 Asset Completion Event (`AssetCompleted`)
Triggers when the asset completes/matures on-chain. The event is captured by the stateless listener, which computes the final payouts and forwards them to the core database backend.

#### Database Operations & State Lifecycle
1. **Investment Profit updates (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `assetinvestments`
   * **Operation**: Read & Update (`updateMany` looping through the payout array)
   * **Key Fields Read**:
     * Locates all `approved` investments under the target `assetId` where `maturityEmailSent` is not true.
   * **Key Fields Updated**:
     * `maturityEmailSent`: Set to `true`.
     * `profitAmount`: Stored allocated profit (on-chain profit / 1e6 USDC scale).
     * `profitPercentage`: Stored yield return percentage.
     * `maturityDate`: Sets final date to current timestamp.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **RWA DB** | `assetinvestments` | `maturityEmailSent` | Boolean | Update | Set to `true`. |
| **RWA DB** | `assetinvestments` | `profitAmount` | Number | Update | Stored yield profit. |
| **RWA DB** | `assetinvestments` | `profitPercentage` | Number | Update | Stored yield rate. |
| **RWA DB** | `assetinvestments` | `maturityDate` | Date | Update | Overwrites with final mature date. |

---

### 1.11 Buyback Request (`BuyBackRequested` event)
Allows investors to submit withdrawal/liquidation requests to creators.

#### Database Operations & State Lifecycle
1. **Request Registration (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `assetinvestments`
   * **Operation**: Read & Update (`findOneAndUpdate` matching the unique on-chain `investmentId`)
   * **Key Fields Read**:
     * Confirms the investment belongs to the requesting investor.
   * **Key Fields Updated**:
     * `withdrawalStatus`: Set to `"requested"`.
     * `withdrawalRequestedAmount`: Stored USDC price normalized from blockchain/input.
     * `withdrawalRequestId`: Stored on-chain request ID.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **RWA DB** | `assetinvestments` | `investmentId` | Number | Read Match | Matches target on-chain ID. |
| **RWA DB** | `assetinvestments` | `withdrawalStatus`| String | Update | Set to `"requested"`. |
| **RWA DB** | `assetinvestments` | `withdrawalRequestedAmount`| Number | Update | Stored liquidating valuation. |
| **RWA DB** | `assetinvestments` | `withdrawalRequestId`| Number | Update | On-chain buyback ID. |

---

### 1.12 Buyback Approval (`BuyBackApproved` event)
Triggers when the administrator completes the buyback transaction on-chain. The event is processed by the stateless listener and pushed to the core backend.

#### Database Operations & State Lifecycle
1. **Investment Closure (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `assetinvestments`
   * **Operation**: Update (`updateOne`) matching the `withdrawalRequestId`.
   * **Key Fields Updated**:
     * `withdrawalStatus`: Transitions to `"approved"`.
     * `status`: Transitions to `"completed"`.
     * `withdrawalAmount`: Final payout price.
     * `ownershipPercentage`: Set to `0` (investor no longer holds active security shares).
2. **Historical Yield Termination (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `assetyieldmanagements`
   * **Operation**: Update (`updateMany`) matching the investor's yields.
   * **Key Fields Updated**:
     * `status`: Transitions to `"completed"`.
     * `isWithdrawn`: Set to `true` (forces closure on historical unclaimed rental payouts for this investment).

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **RWA DB** | `assetinvestments` | `withdrawalStatus`| String | Update | Transitions to `"approved"`. |
| **RWA DB** | `assetinvestments` | `status` | String | Update | Transitions to `"completed"`. |
| **RWA DB** | `assetinvestments` | `withdrawalAmount`| Number | Update | Final paid output. |
| **RWA DB** | `assetinvestments` | `ownershipPercentage`| Number | Update | Reset to `0`. |
| **RWA DB** | `assetyieldmanagements` | `status` | String | Update | Transitions to `"completed"`. |
| **RWA DB** | `assetyieldmanagements` | `isWithdrawn` | Boolean | Update | Forced to `true`. |

---

### 1.13 Yield Claim / Withdrawal Event (`/asset-withdrawal`)
Authenticates rental yield payouts.

#### Database Operations & State Lifecycle
1. **Yield Payout Updates (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `assetyieldmanagements`
   * **Operation**: Update (`updateOne`)
   * **Key Fields Updated**:
     * `isWithdrawn`: Updated to `true` on the targeted rental yield record.
     * `status`: Updated to `"completed"`.
     * `transactionHash`: Stores the payout transaction hash.
2. **Withdrawal Audit Log Registration (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `assetyieldmanagements`
   * **Operation**: Upsert (`findOneAndUpdate` with `{ upsert: true }`)
   * **Key Fields Stored**:
     * Creates an accompanying audit log with `yieldType` set to `"withdraw"` and `netAmount` set to the payout size.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **RWA DB** | `assetyieldmanagements` | `isWithdrawn` | Boolean | Update | Set to `true` on claim. |
| **RWA DB** | `assetyieldmanagements` | `status` | String | Update | Transitions to `"completed"`. |
| **RWA DB** | `assetyieldmanagements` | `transactionHash`| String | Update | Payout tx hash. |
| **RWA DB** | `assetyieldmanagements` | `yieldType` | String | Insert (Audit) | Set to `"withdraw"`. |
| **RWA DB** | `assetyieldmanagements` | `netAmount` | Number | Insert (Audit) | Yield payout volume. |

---

### 1.14 Dividend Claiming Flow
Allows investors to retrieve distributed dividend payouts.

#### Database Operations & State Lifecycle
1. **Dividend Retrieval (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `dividendfundmanagements`
   * **Operation**: Update (`updateOne`) matching dividend `_id`.
   * **Key Fields Updated**:
     * `status`: Set to `"completed"`.
     * `transactionHash`: Stored payout verification hash.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **RWA DB** | `dividendfundmanagements` | `status` | String | Update | Transitions to `"completed"`. |
| **RWA DB** | `dividendfundmanagements` | `transactionHash`| String | Update | On-chain claim verification hash. |

---

## 2. Actor: Creator (Asset Owner) Flows

### 2.1 Creator Registration
Creates profiles directly in the local RWA DB to list real world assets.

#### Database Operations & State Lifecycle
1. **Duplicate Account Check (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `users`
   * **Operation**: Read (`findOne`) matching the email.
2. **Account Creation (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `users`
   * **Operation**: Insert (`insertOne`)
   * **Key Fields Stored**:
     * `userName`: Stored creator username handle.
     * `email`: Login email.
     * `password`: Hashed bcrypt password.
     * `status`: Hardcoded to `1` (Active).
     * `nonce`: Set to `0`.
     * `twoFaEnabled`: Hardcoded to `false` on registration.
     * `KYCVerified` / `AdminVerified`: Set to `0` (unverified).
     * `isCrypto`: Set to `false`.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **RWA DB** | `users` | `userName` | String | Insert | Display handle. |
| **RWA DB** | `users` | `email` | String | Insert / Read | Unique login identifier. |
| **RWA DB** | `users` | `password` | String | Insert | Bcrypt hashed credentials. |
| **RWA DB** | `users` | `status` | Number | Insert | Set to `1` (active) on registration. |
| **RWA DB** | `users` | `nonce` | Number | Insert | Set to `0`. |
| **RWA DB** | `users` | `twoFaEnabled` | Boolean | Insert | Set to `false`. |

---

### 2.2 Creator Login & 2FA Flow
Authenticating creator accounts locally.

#### Database Operations & State Lifecycle
1. **Credentials Validation (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `users`
   * **Operation**: Read & Update (`findOneAndUpdate`)
   * **Key Fields Read**:
     * `email`: Query criteria.
     * `password`: Loaded to compare bcrypt hash.
     * `twoFaEnabled`: Checked to verify if 2FA is active.
   * **Key Fields Updated (If 2FA is active)**:
     * `tempToken` / `tempTokenExpiresAt`: Populates verification session fields.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **RWA DB** | `users` | `email` | String | Read Match | Filters target account. |
| **RWA DB** | `users` | `password` | String | Read | bcrypt hashed password. |
| **RWA DB** | `users` | `twoFaEnabled` | Boolean | Read | Checks Multi-Factor status. |
| **RWA DB** | `users` | `tempToken` | String | Update | Ephemeral session token. |

---

### 2.3 SIWE Wallet Connection Flow
Validates ownership of creator decentralized addresses.

#### Database Operations & State Lifecycle
1. **Local Wallet Registration (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `users`
   * **Operation**: Update (`updateOne`)
   * **Key Fields Updated**:
     * `walletAddress`: Stores the normalized lowercase Ethereum address.
     * `walletNonce` / `walletNonceExpiresAt`: Cleared/unset.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **RWA DB** | `users` | `walletAddress` | String | Update | lowercase Ethereum address. |
| **RWA DB** | `users` | `walletNonce` | String | Unset | Clears nonce. |

---

### 2.4 Profile Settings Update
Allows creators to customize contact and geolocation data.

#### Database Operations & State Lifecycle
1. **Creator Contact Settings Update (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `users`
   * **Operation**: Update (`updateOne`) matching user `_id`.
   * **Key Fields Updated**:
     * `phoneNumber`: User contact telephone string.
     * `address`: Resident street address.
     * `city`: Resident city.
     * `country`: Resident country.
     * `postalCode`: Resident area zip code.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **RWA DB** | `users` | `phoneNumber` | String | Update | Creator's phone number. |
| **RWA DB** | `users` | `address` | String | Update | Resident street details. |
| **RWA DB** | `users` | `city` | String | Update | Geolocation city. |
| **RWA DB** | `users` | `country` | String | Update | Geolocation country. |
| **RWA DB** | `users` | `postalCode` | String | Update | Geolocation postal/zip code. |

---

### 2.5 Contact Support Request
Allows creators to log troubleshooting issues.

#### Database Operations & State Lifecycle
1. **Support Ticket Registration (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `usercontactsupports`
   * **Operation**: Insert (`insertOne`)
   * **Key Fields Stored**:
     * `userId`: Reference link to local creator profile.
     * `name` & `email`: Contact details.
     * `subject` & `message`: Issue content.
     * `status`: Set to `"pending"`.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **RWA DB** | `usercontactsupports` | `userId` | ObjectId | Insert | Creator link index. |
| **RWA DB** | `usercontactsupports` | `name` | String | Insert | Customer name. |
| **RWA DB** | `usercontactsupports` | `email` | String | Insert | Contact email. |
| **RWA DB** | `usercontactsupports` | `subject` | String | Insert | Support subject. |
| **RWA DB** | `usercontactsupports` | `message` | String | Insert | Support description details. |
| **RWA DB** | `usercontactsupports` | `status` | String | Insert | Set to `"pending"`. |

---

### 2.6 Account Deletion Request
Permits creators to trigger soft-deletion queues.

#### Database Operations & State Lifecycle
1. **Compliance Deletion Entry (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `userdeleteds`
   * **Operation**: Insert (`insertOne`)
   * **Key Fields Stored**:
     * `userId`: Reference link to local creator profile.
     * `email`: Stored email address.
     * `reason`: Explanation text.
     * `status`: Set to `"pending"`.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **RWA DB** | `userdeleteds` | `userId` | ObjectId | Insert | Creator link index. |
| **RWA DB** | `userdeleteds` | `email` | String | Insert | Account email. |
| **RWA DB** | `userdeleteds` | `reason` | String | Insert | Deletion reasoning statement. |
| **RWA DB** | `userdeleteds` | `status` | String | Insert | Set to `"pending"`. |

---

### 2.7 Asset Registration (Listing Creation)
Allows creators to register tokenizable real world asset drafts.

#### Database Operations & State Lifecycle
1. **Asset Registry Creation (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `assetmanagements`
   * **Operation**: Insert (`insertOne`)
   * **Key Fields Stored**:
     * `creatorId`: Local creator ObjectId link.
     * `categoryName`: Categorization code (`"Art"`, `"Mine"`, `"Agro"`, `"Ai"`, `"Realty"`, `"Energy"`).
     * `assetManagementStatus`: Initialized to `0` (Draft).
     * `progressBar`: Set to `0`.
     * `remainingTokens`: Set to maximum asset token pool.
     * `basicInformation`: Nested category metadata objects containing basic project info, technical details, compliance certifications, and financial specifications.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **RWA DB** | `assetmanagements` | `creatorId` | ObjectId | Insert | Creator profile link. |
| **RWA DB** | `assetmanagements` | `categoryName` | String | Insert | Category code. |
| **RWA DB** | `assetmanagements` | `assetManagementStatus`| Number | Insert | Set to `0` (Draft). |
| **RWA DB** | `assetmanagements` | `progressBar` | Number | Insert | Set to `0`. |
| **RWA DB** | `assetmanagements` | `remainingTokens` | Number | Insert | Set to total pool capacity. |
| **RWA DB** | `assetmanagements` | `basicInformation` | Object | Insert | Contains project and compliance specifications. |

---

## 3. Actor: Admin Flows

### 3.1 Admin Login & 2FA Flow
Authenticating administrators.

#### Database Operations & State Lifecycle
1. **Credentials Validation (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `admins`
   * **Operation**: Read (`findOne`) matching the email.
   * **Key Fields Read**:
     * `email`, `password`, `twoFactorEnabled`, `twoFactorSecret`.
2. **Session Registration (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `admins`
   * **Operation**: Update (`updateOne`)
   * **Key Fields Updated**:
     * `lastLoginAt`: Updates to current timestamp.
     * `jwtTokens`: Appends the newly issued session JWT.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **RWA DB** | `admins` | `email` | String | Read Match | Target matching login email. |
| **RWA DB** | `admins` | `password` | String | Read | bcrypt hashed password. |
| **RWA DB** | `admins` | `twoFactorEnabled` | Boolean | Read | Checks Multi-Factor status. |
| **RWA DB** | `admins` | `twoFactorSecret` | String | Read | TOTP secret validation key. |
| **RWA DB** | `admins` | `lastLoginAt` | Date | Update | Timestamp of last access. |
| **RWA DB** | `admins` | `jwtTokens` | Array | Update | Appends active session token. |

---

### 3.2 Admin Wallet Authentication Flow (SIWE)
Administrators authenticate via cryptographic signatures.

#### Database Operations & State Lifecycle
1. **SIWE Verification & Session Check (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `admins`
   * **Operation**: Read & Update (`findOneAndUpdate`) matching `walletAddress` (normalized lowercase).
   * **Key Fields Read**:
     * `walletNonce`, `walletNonceExpiresAt`.
   * **Key Fields Updated**:
     * `walletNonce` / `walletNonceExpiresAt`: `$unset` or cleared.
     * `jwtTokens`: Appends the active session JWT.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **RWA DB** | `admins` | `walletNonce` | String | Update / Unset | Challenge connection nonce. |
| **RWA DB** | `admins` | `walletNonceExpiresAt`| Date | Update / Unset | Nonce active window. |
| **RWA DB** | `admins` | `walletAddress` | String | Read Match | lowercase Ethereum address. |
| **RWA DB** | `admins` | `jwtTokens` | Array | Update | Appends active session token. |

---

### 3.3 Resolve Support Tickets
Allows admins to resolve tickets filed by Investors or Creators.

#### Database Operations & State Lifecycle
1. **Investor Ticket Resolution (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `investorcontactsupports`
   * **Operation**: Update (`updateOne`)
   * **Key Fields Updated**:
     * `status`: Set to `"resolved"`.
     * `adminNote` & `resolvedAt`.
2. **Creator Ticket Resolution (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `usercontactsupports`
   * **Operation**: Update (`updateOne`)
   * **Key Fields Updated**:
     * `status`: Set to `"resolved"`.
     * `adminNote` & `resolvedAt`.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **RWA DB** | `investorcontactsupports` | `status` | String | Update | Set to `"resolved"`. |
| **RWA DB** | `investorcontactsupports` | `adminNote` | String | Update | Resolution comments. |
| **RWA DB** | `investorcontactsupports` | `resolvedAt` | Date | Update | Resolve timestamp. |
| **RWA DB** | `usercontactsupports` | `status` | String | Update | Set to `"resolved"`. |
| **RWA DB** | `usercontactsupports` | `adminNote` | String | Update | Resolution comments. |
| **RWA DB** | `usercontactsupports` | `resolvedAt` | Date | Update | Resolve timestamp. |

---

### 3.4 Approve Compliance Account Deletion
Allows admins to delete profiles filed by Investors or Creators.

#### Database Operations & State Lifecycle
1. **Investor Deletion Approval (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `investordeleteds` ➔ Update `status` to `"deleted"`, `deletedAt` to current timestamp.
   * **Collection**: `investors` ➔ Update status to `0` (Suspended).
2. **Creator Deletion Approval (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `userdeleteds` ➔ Update `status` to `"deleted"`, `deletedAt` to current timestamp.
   * **Collection**: `users` ➔ Update status to `0` (Suspended).

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **RWA DB** | `investordeleteds` | `status` | String | Update | Transitions to `"deleted"`. |
| **RWA DB** | `investordeleteds` | `deletedAt` | Date | Update | Processed timestamp. |
| **RWA DB** | `investors` | `status` | Number | Update | Sets profile status to `0` (Disabled). |
| **RWA DB** | `userdeleteds` | `status` | String | Update | Transitions to `"deleted"`. |
| **RWA DB** | `userdeleteds` | `deletedAt` | Date | Update | Processed timestamp. |
| **RWA DB** | `users` | `status` | Number | Update | Sets creator status to `0` (Disabled). |

---

### 3.5 Asset Listing Approval
Administrators review creator drafts and approve them for active listing, generating security token targets.

#### Database Operations & State Lifecycle
1. **Asset Listing Approval (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `assetmanagements`
   * **Operation**: Update (`updateOne`) matching the target asset.
   * **Key Fields Updated**:
     * `AdminId`: Approving admin ObjectId.
     * `assetManagementStatus`: Set to `2` (Approved/Listed).
     * `securityToken`: Stores the deployed contract address representing the asset on-chain.
     * `transactionHash`: Deployment contract hash.
     * `assetStatus`: Initialized to `"On Going"`.
2. **Asset Listing Rejection (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `assetmanagements`
   * **Operation**: Update (`updateOne`)
   * **Key Fields Updated**:
     * `AdminId`: Rejecting admin ObjectId.
     * `assetManagementStatus`: Set to `3` (Rejected).
     * `rejectedReason`: Stored explanation string.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **RWA DB** | `assetmanagements` | `AdminId` | ObjectId | Update | Approving admin reference. |
| **RWA DB** | `assetmanagements` | `assetManagementStatus`| Number | Update | Set to `2` (Approved) or `3` (Rejected). |
| **RWA DB** | `assetmanagements` | `securityToken` | String | Update | Deployed smart contract address. |
| **RWA DB** | `assetmanagements` | `transactionHash` | String | Update | Deploy contract tx hash. |
| **RWA DB** | `assetmanagements` | `assetStatus` | String | Update | Initialized status: `"On Going"`. |
| **RWA DB** | `assetmanagements` | `rejectedReason` | String | Update | Reason detail (if status `3`). |

---

### 3.6 Asset Completion Ledger Creation (`AssetCompleted` event)
Compiles final asset summaries upon maturity or liquidation.

#### Database Operations & State Lifecycle
1. **Asset Completion Ledger Upsert (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `assetmanagementcompletes`
   * **Operation**: Upsert (`findOneAndUpdate` with `{ upsert: true }`)
   * **Key Fields Stored**:
     * `creatorId` / `assetId`: Target keys.
     * `assetCompletionDate`: Set to current date.
     * `assetFinalAmount`: Realized funding volume.
     * `profitGenerated`: Total USDC yields generated by the asset.
2. **Asset Status Update (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `assetmanagements`
   * **Operation**: Update (`updateOne`)
   * **Key Fields Updated**:
     * `assetStatus`: Toggled to `"Closed"`.
     * `assetCompletionStatus`: Set to `1` (Completed).

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **RWA DB** | `assetmanagementcompletes` | `creatorId` | ObjectId | Upsert Match | Creator reference link. |
| **RWA DB** | `assetmanagementcompletes` | `assetId` | ObjectId | Upsert Match | Asset reference link. |
| **RWA DB** | `assetmanagementcompletes` | `assetCompletionDate`| Date | Insert / Update | Completion date log. |
| **RWA DB** | `assetmanagementcompletes` | `assetFinalAmount` | Number | Insert / Update | final funding sum. |
| **RWA DB** | `assetmanagementcompletes` | `profitGenerated` | Number | Insert / Update | Generated profit volume. |
| **RWA DB** | `assetmanagements` | `assetStatus` | String | Update | transitions to `"Closed"`. |
| **RWA DB** | `assetmanagements` | `assetCompletionStatus`| Number | Update | Transitions to `1`. |

---

### 3.7 Rental Yield Allocation (`/asset-yield`)
Allows administrators to distribute monthly yields to asset investors.

#### Database Operations & State Lifecycle
1. **Rental Yield Upsert (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `assetyieldmanagements`
   * **Operation**: Upsert (`findOneAndUpdate` with `{ upsert: true }`)
   * **Key Fields Stored**:
     * Compound Unique Filter: `{ AssetId, investerId, yieldYear, yieldMonth, yieldType: "rental" }`.
     * `netAmount`: Stored USDC yield payout value.
     * `netAmountDecimals`: Stored decimals representation.
     * `sharesHolding`: Stored share weight at generation time.
     * `yieldDay`: Stored day index.
     * `status`: Set to `"pending"`.
     * `isYieldClosed`: Set to `false`.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **RWA DB** | `assetyieldmanagements` | `AssetId` | ObjectId | Upsert Match | Matches target asset. |
| **RWA DB** | `assetyieldmanagements` | `investerId` | ObjectId | Upsert Match | Matches target investor. |
| **RWA DB** | `assetyieldmanagements` | `yieldYear` | Number | Upsert Match | Distribution calendar year. |
| **RWA DB** | `assetyieldmanagements` | `yieldMonth` | Number | Upsert Match | Distribution calendar month. |
| **RWA DB** | `assetyieldmanagements` | `yieldType` | String | Upsert Match | Filter set to `"rental"`. |
| **RWA DB** | `assetyieldmanagements` | `netAmount` | Number | Insert / Update | Yield allocation volume. |
| **RWA DB** | `assetyieldmanagements` | `sharesHolding` | Number | Insert / Update | Token balance at distribution time. |
| **RWA DB** | `assetyieldmanagements` | `status` | String | Insert | Set to `"pending"`. |
| **RWA DB** | `assetyieldmanagements` | `isYieldClosed` | Boolean | Insert | Set to `false`. |

---

### 3.8 Admin Dividend Distribution Setup
Allows administrators to configure and distribute dividend rewards.

#### Database Operations & State Lifecycle
1. **Dividend Distribution Setup (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `dividendfundmanagements`
   * **Operation**: Upsert (`findOneAndUpdate` with `{ upsert: true }` matching asset and investor keys)
   * **Key Fields Stored**:
     * `assetId` / `investorId`: Target identities.
     * `dividendAmount`: Stored USDC dividend value.
     * `dividendPercentage`: Stored dividend percentage.
     * `payoutDate`: Planned dividend release timestamp.
     * `status`: Set to `"pending"`.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **RWA DB** | `dividendfundmanagements` | `assetId` | ObjectId | Upsert Match | Target asset identifier. |
| **RWA DB** | `dividendfundmanagements` | `investorId` | ObjectId | Upsert Match | Target investor identifier. |
| **RWA DB** | `dividendfundmanagements` | `dividendAmount` | Number | Insert / Update | Distributed dividend capital. |
| **RWA DB** | `dividendfundmanagements` | `dividendPercentage`| Number | Insert / Update | Dividend percentage yield weight. |
| **RWA DB** | `dividendfundmanagements` | `payoutDate` | Date | Insert / Update | Planned payout date. |
| **RWA DB** | `dividendfundmanagements` | `status` | String | Insert | Set to `"pending"`. |

---

### 3.9 Site Settings Configuration
Maintains system variables.

#### Database Operations & State Lifecycle
1. **Configuration Registry Update (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `site_settings`
   * **Operation**: Upsert (`findOneAndUpdate` matching global identity)
   * **Key Fields Stored**:
     * `maintenanceMode`: Boolean flag.
     * `allowedCountries`: Array of geocodes allowed access.
     * `feeStructure`: Net percentage transaction fee configurations.
     * `adminId`: ObjectId of editing admin.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **RWA DB** | `site_settings` | `maintenanceMode` | Boolean | Upsert / Update | Platform maintenance flag. |
| **RWA DB** | `site_settings` | `allowedCountries` | Array | Upsert / Update | Allowed ISO codes. |
| **RWA DB** | `site_settings` | `feeStructure` | Object | Upsert / Update | Admin percentage settings. |
| **RWA DB** | `site_settings` | `adminId` | ObjectId | Upsert / Update | Editing admin reference. |

---

### 3.10 Admin Panel Configuration
Configures administrator workspace settings.

#### Database Operations & State Lifecycle
1. **Panel Preferences Update (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `adminsettings`
   * **Operation**: Upsert (`findOneAndUpdate` matching global admin settings identity)
   * **Key Fields Stored**:
     * `themePreference`: Workspace display modes (e.g. `"dark"`, `"light"`).
     * `notificationSettings`: Array of notification filters.
     * `twoFactorConfig`: Boolean settings for enforcement.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **RWA DB** | `adminsettings` | `themePreference` | String | Upsert / Update | workspace visual theme choice. |
| **RWA DB** | `adminsettings` | `notificationSettings`| Array | Upsert / Update | Alert notifications filters. |
| **RWA DB** | `adminsettings` | `twoFactorConfig` | Boolean | Upsert / Update | Enforces MFA logins. |

---

### 3.11 Block Checkpoint Update
Maintains the block pointer for stateless blockchain event listeners.

#### Database Operations & State Lifecycle
1. **Checkpoint Progress Commit (RWA DB)**:
   * **Database**: Local Real World Asset Business Database
   * **Collection**: `listenerblockcheckpoints`
   * **Operation**: Update (`updateOne` using `$max`) matching `_id: "global"`.
   * **Key Fields Updated**:
     * `blockNumber`: Atomically advances to the highest parsed block number, preventing older event replays from moving the pointer backward.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **RWA DB** | `listenerblockcheckpoints` | `_id` | String | Match | Filter set to `"global"`. |
| **RWA DB** | `listenerblockcheckpoints` | `blockNumber` | Number | Update ($max) | Increments block height check pointer. |
