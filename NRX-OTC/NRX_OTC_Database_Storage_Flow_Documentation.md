# NRX OTC Database Storage Flow Documentation

This document provides a comprehensive, end-to-end technical reference for the data lifecycles, cross-service synchronization, and blockchain event processing layers across the NRX OTC platform, structured by platform actors and mapped directly to database collections and schema keys.

---

## Master Persistence & CRUD Mappings (OTC Platform)

The following master reference table maps every database operation, linking logical actor-based lifecycles to MongoDB collection queries/updates, database boundaries, and blockchain events.

| Actor | Flow / Lifecycle Step | Database Layer | MongoDB Collection | CRUD Operation | Key Database Fields | Blockchain Event / Queue |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Investor** | Investor Registration | Auth DB (Central) <br> OTC DB (Local) | `investors` (Auth) <br> `investors` (OTC) | Insert (Auth) <br> Upsert (OTC Sync) | Insert: `email`, `userName`, `password`, `globalId` <br> Sync: `globalId`, `userName`, `email`, `status` | None |
| **Investor** | Investor Login & 2FA | Auth DB (Central) | `investors` | Read & Update | Read: `email`, `password`, `twoFaEnabled`, `twoFaSecret` \| Update: `tempToken`, `tempTokenExpiresAt` | None |
| **Investor** | SIWE Wallet Connection | OTC DB (Local) <br> Auth DB (Central) | `investors` (OTC) <br> `investors` (Auth) | Update (OTC) <br> Update (Auth Sync) | OTC: `walletAddress`, unsets `walletNonce`, `walletNonceExpiresAt` <br> Auth: `walletAddress` mapped to `globalId` | None |
| **Investor** | KYC Status Update | Auth DB (Central) | `investors` | Update | `KYCVerified`, `kycReviewedAt` | None |
| **Investor** | Profile Settings Update | OTC DB (Local) | `investors` | Update | `phoneNumber`, `address`, `city`, `country`, `postalCode` | None |
| **Investor** | Contact Support Request | OTC DB (Local) | `investorcontactsupports` | Insert | `investorId`, `name`, `email`, `subject`, `message`, `status` (pending) | None |
| **Investor** | Account Deletion Request | OTC DB (Local) | `investordeleteds` | Insert | `investorId`, `email`, `reason`, `status` (pending) | None |
| **Investor** | Investment Creation | OTC DB (Local) | `fundinginvestments` | Insert & Update | Insert: `investorId`, `fundId`, `totalAmount`, `investedAmount`, `subscriptionFee`, `status` (pending) \| Update: `documentUrl` | AWS S3 Upload (Mutual Agreement PDF) |
| **Investor** | Investment Event (On-Chain) | OTC DB (Local) | `fundinginvestments` | Read & Update | Read: `walletAddress`, `status` (pending) \| Update: `investId` (stores on-chain tranche ID) | `InvestmentRequested` (Contract) |
| **Investor** | Withdrawal Request | OTC DB (Local) | `fundinginvestments` <br> `investmentwithdrawalmethods` | Update (Investment) <br> Upsert (Method) | Investment: `withdrawalStatus` (requested) <br> Method: `investorId`, `fundId`, `investmentId`, `withdrawalMethod` (bank credentials encrypted) | None |
| **Investor** | Airdrop Claiming | OTC DB (Local) | `successairdrops` <br> `airdrops` | Insert (Claim) <br> Update (Campaign) | Claim: `airdropId`, `investorId`, `walletAddress`, `amountClaimed`, `transactionHash` <br> Campaign: increments `totalClaimedAmount` | `AirdropClaimed` (Contract) |
| **Investor** | Buy Tokens (Purchase) | OTC DB (Local) | `buytokens` | Insert | `investorId`, `fundId`, `amount`, `tokenQuantity`, `pricePerToken`, `status` (pending) | None |
| **Investor** | Yield Claiming | OTC DB (Local) | `fundassetyieldmanagements` | Update | `status` (completed), `transactionHash` | None |
| **Admin** | Admin Login & 2FA | OTC DB (Local) | `admins` | Read & Update | Read: `email`, `password`, `twoFactorEnabled`, `twoFactorSecret` \| Update: `lastLoginAt`, `jwtTokens` | None |
| **Admin** | Admin Wallet Auth | OTC DB (Local) | `admins` | Read & Update | Read: `walletAddress`, `walletNonce`, `walletNonceExpiresAt` \| Update: `walletNonce` (nullified), `jwtTokens` | None |
| **Admin** | Resolve Support Ticket | OTC DB (Local) | `investorcontactsupports` | Update | `status` (resolved), `adminNote`, `resolvedAt` | None |
| **Admin** | Approve Account Deletion | OTC DB (Local) | `investordeleteds` <br> `investors` | Update (Request) <br> Update (Profile) | Request: `status` (deleted), `deletedAt` <br> Profile: `status` (0 / Suspended) | None |
| **Admin** | Fund Creation | OTC DB (Local) | `fundmanagements` | Insert | Stored fields in `generalInformation`, `investmentStructure`, `feesAndCosts`, `smartContractReference`, `fundStatus` | None |
| **Admin** | Investment Approval | OTC DB (Local) | `fundinginvestments` <br> `fundmanagements` | Update (Investment) <br> Update (Fund Metrics) | Investment: `status` (approved), `transactionHash`, `maturityTime`, `nextYieldClaimTime` <br> Fund: `fundedPercentage`, `remainingFundVolume`, `totalInvestedAmount`, `fundStatus` | `InvestmentMade` (Contract) |
| **Admin** | Investment Rejection | OTC DB (Local) | `fundinginvestments` | Update | `status` (rejected), `rejectedReason` | `InvestmentRejected` (Contract) |
| **Admin** | Airdrop Creation | OTC DB (Local) | `airdrops` | Insert | `title`, `description`, `totalAmount`, `tokenAddress`, `airdropStatus` (0 / Draft) | None |
| **Admin** | Buy Tokens Approval | OTC DB (Local) | `buytokens` | Update | `status` (approved), `transactionHash` | None |
| **Admin** | Yield Distribution Setup | OTC DB (Local) | `fundassetyieldmanagements` | Upsert | `fundId`, `investorId`, `yieldAmount`, `yieldPercentage`, `payoutDate`, `status` (pending) | None |
| **Admin** | Site Settings Configuration | OTC DB (Local) | `site_settings` | Upsert | `maintenanceMode`, `allowedCountries`, `feeStructure`, `adminId` | None |
| **Admin** | Block Checkpoint | OTC DB (Local) | `listenerblockcheckpoints` | Update ($max) | `blockNumber` (atomic monotonic update) | None |

---

## 1. Actor: Investor Flows

### 1.1 Investor Registration & Synchronization
When a new investor registers, their credentials and global identifiers are established centrally in the Auth DB before synchronizing downstream to the local OTC DB.

#### Database Operations & State Lifecycle
1. **Duplicate Account Check (Auth DB)**:
   * **Database**: Central Identity Database
   * **Collection**: `investors`
   * **Operation**: Read (`findOne`)
   * **Key Fields**:
     * `email`: Checked to verify no existing account uses this login email.
     * `userName`: Checked to verify the unique handle is free.
2. **Central Account Insertion (Auth DB)**:
   * **Database**: Central Identity Database
   * **Collection**: `investors`
   * **Operation**: Insert (`insertOne`)
   * **Key Fields Stored**:
     * `email`: User login email.
     * `userName`: Chosen username.
     * `password`: Stored as a hashed string using `bcrypt`.
     * `globalId`: Auto-generated UUID/ObjectId reference.
     * `twoFaEnabled`: Hardcoded to `false` on registration.
     * `status`: Hardcoded to `1` (Active).
     * `registeredPlatform`: Stored as `"NRX-OTC"` to track origin.
3. **Local Profile Downstream Sync (OTC DB)**:
   * **Database**: Local OTC Business Database
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
| **Auth DB** | `investors` | `registeredPlatform` | String | Insert | Set to `"NRX-OTC"`. |
| **OTC DB** | `investors` | `globalId` | UUID / String | Upsert Match | Matches sync key. |
| **OTC DB** | `investors` | `email` | String | Update | Copy of central login email. |
| **OTC DB** | `investors` | `userName` | String | Update | Copy of central display handle. |
| **OTC DB** | `investors` | `status` | Number | Update | Local status initialized to `1`. |

---

### 1.2 Investor Login & 2FA Flow
Authenticating investor accounts, verifying passwords, and enforcing temporary Multi-Factor session states.

#### Database Operations & State Lifecycle
1. **Credentials Retrieval (Auth DB)**:
   * **Database**: Central Identity Database
   * **Collection**: `investors`
   * **Operation**: Read (`findOne`) matching the submitted email.
   * **Key Fields Read**:
     * `email`: Query criteria.
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
1. **Nonce Initialization (OTC DB)**:
   * **Database**: Local OTC Business Database
   * **Collection**: `investors`
   * **Operation**: Update (`updateOne`) matching the investor's local `_id`.
   * **Key Fields Updated**:
     * `walletNonce`: Generates and writes a secure cryptographic nonce string.
     * `walletNonceExpiresAt`: Writes an expiry timestamp (5-minute window).
2. **Nonce Verification (OTC DB)**:
   * **Database**: Local OTC Business Database
   * **Collection**: `investors`
   * **Operation**: Read (`findOne`) matching investor `_id`.
   * **Key Fields Read**:
     * `walletNonce`: Compared against the signed message payload.
     * `walletNonceExpiresAt`: Checked to ensure validation timeframe is active.
3. **Local Wallet Registration (OTC DB)**:
   * **Database**: Local OTC Business Database
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
| **OTC DB** | `investors` | `walletNonce` | String | Update / Unset | Random string used for verification. |
| **OTC DB** | `investors` | `walletNonceExpiresAt`| Date | Update / Unset | Expiration window for wallet connection. |
| **OTC DB** | `investors` | `walletAddress` | String | Update | Confirmed lowercase Ethereum address. |
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
1. **Investor Contact Settings Update (OTC DB)**:
   * **Database**: Local OTC Business Database
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
| **OTC DB** | `investors` | `phoneNumber` | String | Update | Investor's phone number. |
| **OTC DB** | `investors` | `address` | String | Update | Resident street details. |
| **OTC DB** | `investors` | `city` | String | Update | Geolocation city. |
| **OTC DB** | `investors` | `country` | String | Update | Geolocation country. |
| **OTC DB** | `investors` | `postalCode` | String | Update | Geolocation postal/zip code. |

---

### 1.6 Contact Support Request
Allows investors to log troubleshooting issues directly with administrators.

#### Database Operations & State Lifecycle
1. **Support Ticket Registration (OTC DB)**:
   * **Database**: Local OTC Business Database
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
| **OTC DB** | `investorcontactsupports` | `investorId` | ObjectId | Insert | Linked profile identifier. |
| **OTC DB** | `investorcontactsupports` | `name` | String | Insert | Customer name. |
| **OTC DB** | `investorcontactsupports` | `email` | String | Insert | Customer notification email. |
| **OTC DB** | `investorcontactsupports` | `subject` | String | Insert | Ticket subject line. |
| **OTC DB** | `investorcontactsupports` | `message` | String | Insert | Detailed description. |
| **OTC DB** | `investorcontactsupports` | `status` | String | Insert | Status initialized to `"pending"`. |

---

### 1.7 Account Deletion Request
Ensures regulatory compliance by permitting investors to trigger soft-deletion queues.

#### Database Operations & State Lifecycle
1. **Compliance Deletion Entry (OTC DB)**:
   * **Database**: Local OTC Business Database
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
| **OTC DB** | `investordeleteds` | `investorId` | ObjectId | Insert | Linked profile identifier. |
| **OTC DB** | `investordeleteds` | `email` | String | Insert | Account email to flag. |
| **OTC DB** | `investordeleteds` | `reason` | String | Insert | Deletion reasoning statement. |
| **OTC DB** | `investordeleteds` | `status` | String | Insert | Initialized status: `"pending"`. |

---

### 1.8 Investment Creation
Initializes a private investment commitment pending Web3 on-chain confirmation.

#### Database Operations & State Lifecycle
1. **Fund Validation (OTC DB)**:
   * **Database**: Local OTC Business Database
   * **Collection**: `fundmanagements`
   * **Operation**: Read (`findById`)
   * **Key Fields Read**:
     * `_id` / `fundId`: Checked to confirm the fund is registered.
     * `fundStatus`: Verified to ensure the status is `"On Going"`.
2. **Investment Record Initialization (OTC DB)**:
   * **Database**: Local OTC Business Database
   * **Collection**: `fundinginvestments`
   * **Operation**: Insert (`insertOne`)
   * **Key Fields Stored**:
     * `investorId`: Local investor profile ObjectId.
     * `fundId`: Target fund management ObjectId.
     * `totalAmount`: Gross investment volume (including fees).
     * `investedAmount`: Net investment volume.
     * `subscriptionFee`: Stored fee value.
     * `status`: Hardcoded to `"pending"`.
     * `documentType`: Hardcoded to `"investment-agreement"`.
3. **Agreement Document Registration (OTC DB)**:
   * **Database**: Local OTC Business Database
   * **Collection**: `fundinginvestments`
   * **Operation**: Update (`updateOne`) matching the investment record.
   * **Key Fields Updated**:
     * `documentUrl`: Stores the AWS S3 link pointing to the PDF mutual agreement generated from HTML.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **OTC DB** | `fundmanagements` | `_id` | ObjectId | Read Match | Target fund identifier. |
| **OTC DB** | `fundmanagements` | `fundStatus` | String | Read | Checks that fund is `"On Going"`. |
| **OTC DB** | `fundinginvestments` | `investorId` | ObjectId | Insert | Linked investor identifier. |
| **OTC DB** | `fundinginvestments` | `fundId` | ObjectId | Insert | Linked fund identifier. |
| **OTC DB** | `fundinginvestments` | `totalAmount` | Number | Insert | Gross investment volume. |
| **OTC DB** | `fundinginvestments` | `investedAmount` | Number | Insert | Net investment volume. |
| **OTC DB** | `fundinginvestments` | `subscriptionFee` | Number | Insert | Charged fee metrics. |
| **OTC DB** | `fundinginvestments` | `status` | String | Insert | Initialized status: `"pending"`. |
| **OTC DB** | `fundinginvestments` | `documentType` | String | Insert | Set to `"investment-agreement"`. |
| **OTC DB** | `fundinginvestments` | `documentUrl` | String | Update | PDF link hosted on S3. |

---

### 1.9 Investment Blockchain Event (`InvestmentRequested`)
Links the Web2 pending commitment to the on-chain blockchain tranche after detecting the user's blockchain deposit.

#### Database Operations & State Lifecycle
1. **On-Chain Event Detection**:
   * The stateless **`nrx-otc-contract-backend`** listens to the Sepolia blockchain.
   * On detecting `InvestmentRequested`, it extracts `walletAddress`, `fundAddress`, and `investId` (tranche ID) and sends an HMAC-SHA256 callback to the core backend.
2. **Pending Investment Record Match (OTC DB)**:
   * **Database**: Local OTC Business Database
   * **Collection**: `fundinginvestments`
   * **Operation**: Read (`findOne` matching target investor and fund).
   * **Key Fields Read**:
     * `investorId` & `fundId`: Matching queries.
     * `status`: Must be `"pending"`.
3. **On-Chain Identifier Registration (OTC DB)**:
   * **Database**: Local OTC Business Database
   * **Collection**: `fundinginvestments`
   * **Operation**: Update (`updateOne`)
   * **Key Fields Updated**:
     * `investId`: Set to the parsed on-chain numeric tranche ID.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **OTC DB** | `fundinginvestments` | `investorId` | ObjectId | Read Match | Matches target investor. |
| **OTC DB** | `fundinginvestments` | `fundId` | ObjectId | Read Match | Matches target fund. |
| **OTC DB** | `fundinginvestments` | `status` | String | Read Match | Confirms status is `"pending"`. |
| **OTC DB** | `fundinginvestments` | `investId` | Number | Update | Sets the blockchain tranche ID. |

---

### 1.10 Withdrawal Request
Allows investors to submit exit details for mature investments.

#### Database Operations & State Lifecycle
1. **Maturity Verification (OTC DB)**:
   * **Database**: Local OTC Business Database
   * **Collection**: `fundinginvestments`
   * **Operation**: Read & Update (`findOneAndUpdate`)
   * **Key Fields Read**:
     * `_id` (investmentId) and `investorId`: Query parameters.
     * `status`: Verified to ensure it is `"approved"`.
   * **Key Fields Updated**:
     * `withdrawalStatus`: Updated to `"requested"`.
2. **Payout Methods Registration (OTC DB)**:
   * **Database**: Local OTC Business Database
   * **Collection**: `investmentwithdrawalmethods`
   * **Operation**: Upsert (`findOneAndUpdate` with `{ upsert: true }`)
   * **Key Fields Stored**:
     * `investorId` / `fundId` / `investmentId`: Stored linking identifiers.
     * `withdrawalMethod`: Stored method (`"nrx token"`, `"usdc"`, or `"euro"`).
     * `bankName`, `iban`, `swift`, `accountNumber` (etc.): Hashed/encrypted banking details.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **OTC DB** | `fundinginvestments` | `status` | String | Read | Checked that investment status is `"approved"`. |
| **OTC DB** | `fundinginvestments` | `withdrawalStatus`| String | Update | Updated to `"requested"`. |
| **OTC DB** | `investmentwithdrawalmethods` | `investorId` | ObjectId | Upsert Match | Linked investor identity. |
| **OTC DB** | `investmentwithdrawalmethods` | `fundId` | ObjectId | Upsert Match | Linked fund identity. |
| **OTC DB** | `investmentwithdrawalmethods` | `investmentId` | ObjectId | Upsert Match | Linked investment identity. |
| **OTC DB** | `investmentwithdrawalmethods` | `withdrawalMethod`| String | Insert / Update | Method chosen: `"nrx token"`, `"usdc"`, `"euro"`. |
| **OTC DB** | `investmentwithdrawalmethods` | `bankName` | String | Insert / Update | Bank name credential (encrypted). |
| **OTC DB** | `investmentwithdrawalmethods` | `iban` | String | Insert / Update | Bank IBAN credential (encrypted). |

---

### 1.11 Airdrop Claiming Flow
Allows eligible investors to claim promotional tokens on-chain.

#### Database Operations & State Lifecycle
1. **Claim Verification (OTC DB)**:
   * **Database**: Local OTC Business Database
   * **Collection**: `airdrops`
   * **Operation**: Read (`findOne`) matching the active campaign.
   * **Key Fields Read**:
     * `airdropStatus`: Must equal `1` (Active).
     * `totalAmount` & `totalClaimedAmount`: Checked to confirm remaining tokens are available.
2. **Claim Record Log (OTC DB)**:
   * **Database**: Local OTC Business Database
   * **Collection**: `successairdrops`
   * **Operation**: Insert (`insertOne`)
   * **Key Fields Stored**:
     * `airdropId`: Reference campaign ID.
     * `investorId`: Claiming investor ObjectId.
     * `walletAddress`: Linked claiming wallet.
     * `amountClaimed`: Tokens retrieved.
     * `transactionHash`: Stored on-chain claim payout hash.
     * `claimedAt`: Current timestamp.
3. **Airdrop Campaign Balance Update (OTC DB)**:
   * **Database**: Local OTC Business Database
   * **Collection**: `airdrops`
   * **Operation**: Update (`updateOne`)
   * **Key Fields Updated**:
     * `totalClaimedAmount`: Incremented by the claimed token amount.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **OTC DB** | `airdrops` | `airdropStatus` | Number | Read | Checks that campaign status is `1` (active). |
| **OTC DB** | `airdrops` | `totalAmount` | Number | Read | Total pool cap. |
| **OTC DB** | `airdrops` | `totalClaimedAmount`| Number | Read / Update | Increments on claims. |
| **OTC DB** | `successairdrops` | `airdropId` | ObjectId | Insert | Associated campaign ID. |
| **OTC DB** | `successairdrops` | `investorId` | ObjectId | Insert | Claiming investor. |
| **OTC DB** | `successairdrops` | `walletAddress` | String | Insert | Receiving wallet address. |
| **OTC DB** | `successairdrops` | `amountClaimed` | Number | Insert | Tokens distributed. |
| **OTC DB** | `successairdrops` | `transactionHash` | String | Insert | Distribution tx hash. |

---

### 1.12 Buy Tokens (Purchase Request)
Allows investors to buy platform utility tokens.

#### Database Operations & State Lifecycle
1. **Purchase Request Register (OTC DB)**:
   * **Database**: Local OTC Business Database
   * **Collection**: `buytokens`
   * **Operation**: Insert (`insertOne`)
   * **Key Fields Stored**:
     * `investorId` / `fundId`: Reference IDs.
     * `amount`: Fiat/USDC paid.
     * `tokenQuantity`: Purchased count.
     * `pricePerToken`: Stored unit price.
     * `status`: Initialized to `"pending"`.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **OTC DB** | `buytokens` | `investorId` | ObjectId | Insert | Purchaser ID. |
| **OTC DB** | `buytokens` | `fundId` | ObjectId | Insert | Reference fund ID. |
| **OTC DB** | `buytokens` | `amount` | Number | Insert | Paid capital. |
| **OTC DB** | `buytokens` | `tokenQuantity` | Number | Insert | Target token count. |
| **OTC DB** | `buytokens` | `pricePerToken` | Number | Insert | Token price weight. |
| **OTC DB** | `buytokens` | `status` | String | Insert | Set to `"pending"`. |

---

### 1.13 Yield Claiming Flow
Retrieval of monthly yields.

#### Database Operations & State Lifecycle
1. **Yield Payout Update (OTC DB)**:
   * **Database**: Local OTC Business Database
   * **Collection**: `fundassetyieldmanagements`
   * **Operation**: Update (`updateOne`) matching the yield record.
   * **Key Fields Updated**:
     * `status`: Set to `"completed"`.
     * `transactionHash`: Deployed smart contract payout hash.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **OTC DB** | `fundassetyieldmanagements` | `status` | String | Update | Transitions to `"completed"`. |
| **OTC DB** | `fundassetyieldmanagements` | `transactionHash`| String | Update | On-chain claim verification hash. |

---

## 2. Actor: Admin Flows

### 2.1 Admin Login & 2FA Flow
Administrators log into the backend panels via password validation combined with TOTP checks.

#### Database Operations & State Lifecycle
1. **Credentials Validation (OTC DB)**:
   * **Database**: Local OTC Business Database
   * **Collection**: `admins`
   * **Operation**: Read (`findOne`) matching the email.
   * **Key Fields Read**:
     * `email`: Query criteria.
     * `password`: Loaded to compare bcrypt hash.
     * `twoFactorEnabled`: Checked to decide if 2FA verification is required.
2. **TOTP Check (OTC DB - If 2FA active)**:
   * **Database**: Local OTC Business Database
   * **Collection**: `admins`
   * **Operation**: Read (`findById`).
   * **Key Fields Read**:
     * `twoFactorSecret`: Loaded to verify the submitted OTP code.
3. **Session Registration (OTC DB)**:
   * **Database**: Local OTC Business Database
   * **Collection**: `admins`
   * **Operation**: Update (`updateOne`)
   * **Key Fields Updated**:
     * `lastLoginAt`: Updates to the current date and time.
     * `jwtTokens`: Appends the newly issued session JWT to the active tokens array.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **OTC DB** | `admins` | `email` | String | Read Match | Login identifier. |
| **OTC DB** | `admins` | `password` | String | Read | Bcrypt hashed credential. |
| **OTC DB** | `admins` | `twoFactorEnabled` | Boolean | Read | Checks 2FA status. |
| **OTC DB** | `admins` | `twoFactorSecret` | String | Read | Checks TOTP secret token. |
| **OTC DB** | `admins` | `lastLoginAt` | Date | Update | Timestamp of last access. |
| **OTC DB** | `admins` | `jwtTokens` | Array | Update | Appends active session token. |

---

### 2.2 Admin Wallet Authentication Flow (SIWE)
Administrators link blockchain wallets and authenticate directly by signing messages.

#### Database Operations & State Lifecycle
1. **Nonce Registration (OTC DB)**:
   * **Database**: Local OTC Business Database
   * **Collection**: `admins`
   * **Operation**: Update (`updateOne`).
   * **Key Fields Updated**:
     * `walletNonce`: Generates and stores the cryptographic nonce.
     * `walletNonceExpiresAt`: Sets the expiry window.
2. **SIWE Verification & Session Check (OTC DB)**:
   * **Database**: Local OTC Business Database
   * **Collection**: `admins`
   * **Operation**: Read & Update (`findOneAndUpdate`) matching `walletAddress` (normalized lowercase).
   * **Key Fields Read**:
     * `walletNonce`: Verified against the nonce parameter in the message payload.
     * `walletNonceExpiresAt`: Verified to ensure the validation window is active.
   * **Key Fields Updated**:
     * `walletNonce`: `$unset` or cleared.
     * `walletNonceExpiresAt`: `$unset` or cleared.
     * `jwtTokens`: Appends the active session JWT.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **OTC DB** | `admins` | `walletNonce` | String | Update / Unset | Cryptographic connection challenge nonce. |
| **OTC DB** | `admins` | `walletNonceExpiresAt`| Date | Update / Unset | Nonce active lifetime. |
| **OTC DB** | `admins` | `walletAddress` | String | Read Match | lowercase Ethereum address. |
| **OTC DB** | `admins` | `jwtTokens` | Array | Update | Appends session token. |

---

### 2.3 Resolve Support Ticket
Allows administrators to manage, comment on, and resolve support logs.

#### Database Operations & State Lifecycle
1. **Support Ticket Resolution (OTC DB)**:
   * **Database**: Local OTC Business Database
   * **Collection**: `investorcontactsupports`
   * **Operation**: Update (`updateOne`) matching ticket `_id`.
   * **Key Fields Updated**:
     * `status`: Updated to `"resolved"`.
     * `adminNote`: Notes detailing resolution steps.
     * `resolvedAt`: Resolution timestamp.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **OTC DB** | `investorcontactsupports` | `status` | String | Update | Transitions to `"resolved"`. |
| **OTC DB** | `investorcontactsupports` | `adminNote` | String | Update | Admin text explanation notes. |
| **OTC DB** | `investorcontactsupports` | `resolvedAt` | Date | Update | Resolved datetime log. |

---

### 2.4 Approve Account Deletion
Allows administrators to process and authorize account deletion requests.

#### Database Operations & State Lifecycle
1. **Compliance Request Update (OTC DB)**:
   * **Database**: Local OTC Business Database
   * **Collection**: `investordeleteds`
   * **Operation**: Update (`updateOne`) matching delete entry `_id`.
   * **Key Fields Updated**:
     * `status`: Set to `"deleted"`.
     * `deletedAt`: Current timestamp.
2. **Investor Account Suspension (OTC DB)**:
   * **Database**: Local OTC Business Database
   * **Collection**: `investors`
   * **Operation**: Update (`updateOne`) matching investor profile `globalId`.
   * **Key Fields Updated**:
     * `status`: Updated to `0` (Disabled/Suspended profile status).

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **OTC DB** | `investordeleteds` | `status` | String | Update | Transitions to `"deleted"`. |
| **OTC DB** | `investordeleteds` | `deletedAt` | Date | Update | Processed timestamp. |
| **OTC DB** | `investors` | `status` | Number | Update | Sets status to `0` (Disabled/Suspended). |

---

### 2.5 Fund Creation
Administrators seed and configure funds (OTC products) on the platform.

#### Database Operations & State Lifecycle
1. **Fund Registry Creation (OTC DB)**:
   * **Database**: Local OTC Business Database
   * **Collection**: `fundmanagements`
   * **Operation**: Insert (`insertOne`)
   * **Key Fields Stored**:
     * `generalInformation`: Subdocument containing `fundName`, `fundCategory`, `description`, `fundLaunchDate`, `fundCurrency`.
     * `investmentStructure`: Subdocument containing `minimumInvestment`, `maximumInvestment`, `totalFundVolume`, `maturityPeriod`, `yieldType`.
     * `feesAndCosts`: Subdocument containing `subscriptionFee`, `managementFee`, `administrativeCustodyFee`.
     * `complianceAndGovernance`: Subdocument containing `regulatoryJurisdiction`, `governingLaw`.
     * `smartContractReference`: Stored smart contract address.
     * `fundedPercentage` & `totalInvestedAmount`: Defaulted to `0`.
     * `fundStatus`: Defaulted to `"On Going"`.
     * `remainingFundVolume`: Set to total fund volume.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **OTC DB** | `fundmanagements` | `generalInformation` | Object | Insert | Nested details (names, descriptions, currency, categories). |
| **OTC DB** | `fundmanagements` | `investmentStructure` | Object | Insert | Volumes, limits, maturity times, yields. |
| **OTC DB** | `fundmanagements` | `feesAndCosts` | Object | Insert | Management, subscription, and custody fees. |
| **OTC DB** | `fundmanagements` | `smartContractReference`| String | Insert | Contract address deployed on-chain. |
| **OTC DB** | `fundmanagements` | `fundedPercentage` | Number | Insert | default `0`. |
| **OTC DB** | `fundmanagements` | `totalInvestedAmount`| Number | Insert | default `0`. |
| **OTC DB** | `fundmanagements` | `fundStatus` | String | Insert | default `"On Going"`. |
| **OTC DB** | `fundmanagements` | `remainingFundVolume`| Number | Insert | Initialized to total capacity volume. |

---

### 2.6 Investment Approval (`InvestmentMade` event)
Triggers when the administrator approves the investment tranche on-chain. The event is captured by the stateless contract backend listener and pushed to the core service to update records and fund statistics.

#### Database Operations & State Lifecycle
1. **Investment State Transition (OTC DB)**:
   * **Database**: Local OTC Business Database
   * **Collection**: `fundinginvestments`
   * **Operation**: Update (`updateOne`) matching the unique on-chain `investId`.
   * **Key Fields Updated**:
     * `status`: Transitions from `"pending"` to `"approved"`.
     * `transactionHash`: Stores the approval transaction hash.
     * `maturityTime`: Set to the on-chain timestamp.
     * `nextYieldClaimTime`: Set to the on-chain claim times array.
2. **Fund Progress Recalculation (OTC DB)**:
   * **Database**: Local OTC Business Database
   * **Collection**: `fundmanagements`
   * **Operation**: Update (`updateOne`) matching the parent fund.
   * **Key Fields Updated**:
     * `fundedPercentage`: Recalculates and updates the funding completion ratio.
     * `remainingFundVolume`: Reduced by the investment amount.
     * `totalInvestedAmount`: Increased by the investment amount.
     * `fundStatus`: Toggles to `"Closing Soon"` (> 70%) or `"Closed"` (100% capacity).

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **OTC DB** | `fundinginvestments` | `status` | String | Update | Transitions to `"approved"`. |
| **OTC DB** | `fundinginvestments` | `transactionHash`| String | Update | On-chain approval transaction hash. |
| **OTC DB** | `fundinginvestments` | `maturityTime` | Number | Update | On-chain maturity epoch timestamp. |
| **OTC DB** | `fundinginvestments` | `nextYieldClaimTime`| Array | Update | Future yield claims list (epochs). |
| **OTC DB** | `fundmanagements` | `fundedPercentage` | Number | Update | Updated percentage ratio. |
| **OTC DB** | `fundmanagements` | `remainingFundVolume`| Number | Update | Capacity reduced by purchase size. |
| **OTC DB** | `fundmanagements` | `totalInvestedAmount`| Number | Update | Capital total increased. |
| **OTC DB** | `fundmanagements` | `fundStatus` | String | Update | Set to `"Closing Soon"` or `"Closed"`. |

---

### 2.7 Investment Rejection (`InvestmentRejected` event)
Triggers when the administrator rejects the investment tranche on-chain. The event is parsed by the stateless listener and pushed to the core backend.

#### Database Operations & State Lifecycle
1. **Investment Rejection State (OTC DB)**:
   * **Database**: Local OTC Business Database
   * **Collection**: `fundinginvestments`
   * **Operation**: Update (`updateOne`) matching the `investId`.
   * **Key Fields Updated**:
     * `status`: Transitions to `"rejected"`.
     * `rejectedReason`: Stored reason string provided by the on-chain event.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **OTC DB** | `fundinginvestments` | `status` | String | Update | Transitions to `"rejected"`. |
| **OTC DB** | `fundinginvestments` | `rejectedReason` | String | Update | Stored rejection reason. |

---

### 2.8 Airdrop Campaign Creation
Launches new promotional programs.

#### Database Operations & State Lifecycle
1. **Campaign Registration (OTC DB)**:
   * **Database**: Local OTC Business Database
   * **Collection**: `airdrops`
   * **Operation**: Insert (`insertOne`)
   * **Key Fields Stored**:
     * `title`: Campaign name.
     * `description`: Campaign text details.
     * `totalAmount`: Stored token allocation count.
     * `tokenAddress`: Address of the distributed token on-chain.
     * `airdropStatus`: Set to `0` (Draft) on setup, then updated to `1` (Active) when launching.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **OTC DB** | `airdrops` | `title` | String | Insert | Airdrop title. |
| **OTC DB** | `airdrops` | `description` | String | Insert | Campaign descriptions. |
| **OTC DB** | `airdrops` | `totalAmount` | Number | Insert | Tokens total pool. |
| **OTC DB** | `airdrops` | `tokenAddress` | String | Insert | EVM token contract address. |
| **OTC DB** | `airdrops` | `airdropStatus` | Number | Insert / Update | `0` (Draft) or `1` (Active). |

---

### 2.9 Buy Tokens Approval
Updates the status of token purchase requests.

#### Database Operations & State Lifecycle
1. **Purchase State update (OTC DB)**:
   * **Database**: Local OTC Business Database
   * **Collection**: `buytokens`
   * **Operation**: Update (`updateOne`) matching token order `_id`.
   * **Key Fields Updated**:
     * `status`: Updated to `"approved"` (or `"rejected"`).
     * `transactionHash`: Stored payout verification transaction hash.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **OTC DB** | `buytokens` | `status` | String | Update | Set to `"approved"` or `"rejected"`. |
| **OTC DB** | `buytokens` | `transactionHash`| String | Update | Tx confirmation hash. |

---

### 2.10 Yield Distribution Setup
Seeds yield payouts for active fund investors.

#### Database Operations & State Lifecycle
1. **Yield Record Generation (OTC DB)**:
   * **Database**: Local OTC Business Database
   * **Collection**: `fundassetyieldmanagements`
   * **Operation**: Upsert (`findOneAndUpdate` with `{ upsert: true }` matching fund and investor key combination)
   * **Key Fields Stored**:
     * `fundId` / `investorId`: Target identities.
     * `yieldAmount`: Calculated yield.
     * `yieldPercentage`: Relative return percentage.
     * `payoutDate`: Planned yield release timestamp.
     * `status`: Set to `"pending"`.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **OTC DB** | `fundassetyieldmanagements` | `fundId` | ObjectId | Upsert Match | Reference fund identifier. |
| **OTC DB** | `fundassetyieldmanagements` | `investorId` | ObjectId | Upsert Match | Reference investor identifier. |
| **OTC DB** | `fundassetyieldmanagements` | `yieldAmount` | Number | Insert / Update | Distributed yield capital. |
| **OTC DB** | `fundassetyieldmanagements` | `yieldPercentage`| Number | Insert / Update | Yield rate weight. |
| **OTC DB** | `fundassetyieldmanagements` | `payoutDate` | Date | Insert / Update | Release date timestamp. |
| **OTC DB** | `fundassetyieldmanagements` | `status` | String | Insert | Set to `"pending"`. |

---

### 2.11 Site Settings Configuration
Maintains system variables.

#### Database Operations & State Lifecycle
1. **Configuration Registry Update (OTC DB)**:
   * **Database**: Local OTC Business Database
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
| **OTC DB** | `site_settings` | `maintenanceMode` | Boolean | Upsert / Update | Platform maintenance flag. |
| **OTC DB** | `site_settings` | `allowedCountries`| Array | Upsert / Update | List of allowed ISO codes. |
| **OTC DB** | `site_settings` | `feeStructure` | Object | Upsert / Update | Admin percentage settings. |
| **OTC DB** | `site_settings` | `adminId` | ObjectId | Upsert / Update | Editing admin link. |

---

### 2.12 Block Checkpoint Update
Maintains the block pointer for stateless blockchain event listeners.

#### Database Operations & State Lifecycle
1. **Checkpoint Progress Commit (OTC DB)**:
   * **Database**: Local OTC Business Database
   * **Collection**: `listenerblockcheckpoints`
   * **Operation**: Update (`updateOne` using `$max`) matching `_id: "global"`.
   * **Key Fields Updated**:
     * `blockNumber`: Atomically advances to the highest parsed block number, preventing older event replays from moving the pointer backward.

#### Collection Keys & Database Fields
| Database | Collection | Key / Field | Type | Operation | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **OTC DB** | `listenerblockcheckpoints` | `_id` | String | Match | Filter set to `"global"`. |
| **OTC DB** | `listenerblockcheckpoints` | `blockNumber` | Number | Update ($max) | Increments block height check pointer. |
