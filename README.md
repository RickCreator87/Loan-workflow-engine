# Loan-workflow-engine
Loan Workflow Engine repo This is the missing piece that ties the three repos together.  It should handle:  - Loan creation   - AFR validation   - Tax classification (state + federal separation)   - Ledger entry creation   - Agreement generation   - Signature workflow   - Compliance checks   - Status transition


1. Loan Workflow Engine - Critical Enhancements

Missing Method Implementations:

```javascript
// workflow/afr-validation.js
class AFRValidator {
  constructor() {
    // Deterministic AFR source - IRS published rates
    this.afrSource = 'https://www.irs.gov/applicable-federal-rates';
    this.cacheDuration = 24 * 60 * 60 * 1000; // 24 hours
  }
  
  async getCurrentAFR(loanDate = new Date(), term = 'mid-term') {
    // Implementation with fallback logic
    try {
      // Try to fetch from IRS API or cached file
      const response = await this.fetchAFRFromSource();
      const afrTable = this.parseAFRTable(response);
      
      // Deterministic selection based on date and term
      const period = this.determineAFRPeriod(loanDate);
      const rates = afrTable[period];
      
      if (!rates) {
        throw new Error(`No AFR found for period: ${period}`);
      }
      
      return {
        shortTerm: rates.shortTerm || rates['1-3'] || 0,
        midTerm: rates.midTerm || rates['3-9'] || 0,
        longTerm: rates.longTerm || rates['9+'] || 0,
        period: period,
        effectiveDate: rates.effectiveDate,
        source: this.afrSource
      };
    } catch (error) {
      // Fallback to stored rates
      return this.getCachedAFR(loanDate);
    }
  }
  
  async getAFR(loanDate, termCategory) {
    const currentAFR = await this.getCurrentAFR(loanDate);
    
    // Term mapping: 'short', 'mid', 'long'
    const termMap = {
      'short': 'shortTerm',
      'mid': 'midTerm', 
      'long': 'longTerm'
    };
    
    const termKey = termMap[termCategory] || 'midTerm';
    return currentAFR[termKey];
  }
}

// workflow/loan-creation.js - Enhanced with error handling
class LoanCreator {
  async createLedgerEntry(loanData, classification) {
    try {
      // Validate required fields
      this.validateLoanData(loanData);
      
      // Generate unique, deterministic loan ID
      const loanId = this.generateLoanId(loanData);
      
      // Prepare ledger entry
      const ledgerEntry = {
        id: `ledger-${loanId}-${Date.now()}`,
        loanId: loanId,
        timestamp: new Date().toISOString(),
        type: 'loan_issuance',
        amount: loanData.principal,
        currency: loanData.currency || 'USD',
        parties: {
          lender: loanData.lender,
          borrower: loanData.borrower
        },
        taxClassification: classification,
        metadata: {
          source: 'workflow-engine',
          version: '1.0',
          hash: this.calculateHash(loanData)
        },
        previousHash: this.getLatestLedgerHash(), // For chain integrity
        status: 'pending'
      };
      
      // Write to ledger repo
      const result = await this.writeToLedger(ledgerEntry);
      
      // Update with write-back confirmation
      ledgerEntry.ledgerId = result.ledgerId;
      ledgerEntry.status = 'posted';
      ledgerEntry.postedBy = 'system';
      ledgerEntry.approvedBy = loanData.approver || 'auto-approved';
      
      return ledgerEntry;
      
    } catch (error) {
      console.error('Ledger creation failed:', error);
      
      // Create error ledger entry
      const errorEntry = {
        id: `error-${Date.now()}`,
        type: 'error',
        error: error.message,
        timestamp: new Date().toISOString(),
        data: loanData,
        status: 'failed'
      };
      
      await this.writeToErrorLog(errorEntry);
      throw new Error(`Ledger creation failed: ${error.message}`);
    }
  }
  
  generateLoanId(loanData) {
    // Deterministic ID generation
    const seed = `${loanData.lender.taxId}-${loanData.borrower.taxId}-${loanData.principal}-${new Date(loanData.startDate).getFullYear()}`;
    const hash = this.sha256(seed);
    
    // Format: L-YYYY-XXXXX
    const year = new Date().getFullYear();
    const sequence = hash.substring(0, 5).toUpperCase();
    
    return `L-${year}-${sequence}`;
  }
  
  async generateAgreement(loanData, classification) {
    // Integration with agreement repo
    const agreementTemplate = await this.fetchTemplate('loan-agreement');
    
    // Populate template with data
    const populatedAgreement = this.populateTemplate(agreementTemplate, {
      ...loanData,
      taxClassification: classification,
      afrDetails: await this.getCurrentAFR(),
      generatedDate: new Date().toISOString(),
      agreementHash: this.calculateAgreementHash(loanData)
    });
    
    // Generate multiple formats
    const results = await Promise.all([
      this.generatePDF(populatedAgreement),
      this.generateJSON(populatedAgreement),
      this.generateMarkdown(populatedAgreement)
    ]);
    
    return {
      id: `agreement-${loanData.loanId}`,
      url: `/agreements/${loanData.loanId}`,
      pdfUrl: results[0].url,
      jsonUrl: results[1].url,
      markdownUrl: results[2].url,
      hash: this.calculateHash(populatedAgreement),
      version: '1.0',
      generatedAt: new Date().toISOString()
    };
  }
}

// Enhanced AFR validation logic
class EnhancedAFRValidator extends AFRValidator {
  async validateAFR(loanRate, loanDate, term) {
    const currentAFR = await this.getAFR(loanDate, term);
    
    // Corrected validation: Loan must be AT or ABOVE AFR
    const isValid = loanRate >= currentAFR;
    
    // Optional tolerance (e.g., for rounding differences)
    const tolerance = this.getConfig('afrTolerance') || 0.01; // 1 basis point
    const isWithinTolerance = loanRate >= (currentAFR - tolerance);
    
    return {
      isValid: isValid || isWithinTolerance,
      currentAFR: currentAFR,
      loanRate: loanRate,
      difference: loanRate - currentAFR,
      passesMinimum: loanRate >= currentAFR,
      toleranceApplied: !isValid && isWithinTolerance,
      requirement: 'Loan rate must be ≥ Applicable Federal Rate'
    };
  }
}
```

Enhanced Tax Classification:

```javascript
// workflow/tax-classification.js
class TaxClassifier {
  constructor(configPath = './config/compliance-rules.json') {
    this.rules = this.loadRules(configPath);
    this.stateRules = this.loadStateRules();
  }
  
  async classifyTax(loanData, afrData) {
    const federal = this.classifyFederal(loanData, afrData);
    const state = await this.classifyState(loanData);
    const imputed = this.calculateImputedInterest(loanData, afrData);
    
    return {
      federal: {
        ...federal,
        imputedInterest: imputed,
        imputedAmount: imputed ? this.calculateImputedAmount(loanData, afrData) : 0,
        formsRequired: this.determineForms(federal, imputed),
        filingDeadlines: this.getFilingDeadlines()
      },
      state: {
        ...state,
        jurisdiction: loanData.borrower.state,
        reportingRequired: state.reportingRequired,
        formsRequired: state.forms || [],
        thresholds: state.thresholds || {}
      },
      separation: {
        method: 'entity-level-separation',
        tracking: 'dual-ledger',
        reconciliation: 'monthly'
      },
      metadata: {
        classifiedAt: new Date().toISOString(),
        ruleVersion: this.rules.version,
        classifierVersion: '2.0'
      }
    };
  }
  
  calculateImputedInterest(loanData, afrData) {
    // Imputed interest exists if loan rate < AFR
    const imputedExists = loanData.interestRate < afrData.midTerm;
    
    if (imputedExists) {
      return {
        exists: true,
        afrRate: afrData.midTerm,
        loanRate: loanData.interestRate,
        difference: afrData.midTerm - loanData.interestRate,
        annualAmount: this.calculateAnnualImputed(
          loanData.principal,
          loanData.interestRate,
          afrData.midTerm
        ),
        irsSection: '7872',
        treatment: 'Lender must report as interest income'
      };
    }
    
    return { exists: false };
  }
}

// config/compliance-rules.json structure
const complianceRules = {
  "version": "2024.1",
  "federal": {
    "bonaFideDebt": {
      "requirements": [
        "writtenAgreement",
        "enforceablePromise",
        "reasonableExpectationOfRepayment",
        "actualConsideration"
      ],
      "afrCompliance": {
        "minimum": "applicableFederalRate",
        "tolerance": "0.01",
        "documentation": "afrReferenceRequired"
      }
    },
    "imputedInterest": {
      "calculationMethod": "afrComparison",
      "reportingThreshold": "anyAmount",
      "forms": ["Form 1099-INT", "Schedule B"]
    }
  },
  "state": {
    "CA": {
      "classification": "businessLoan",
      "reportingRequired": true,
      "threshold": 2500,
      "forms": ["CA 592", "CA 592-B"],
      "interestTreatment": "taxableIncome"
    },
    "NY": {
      "classification": "personalLoan",
      "reportingRequired": true,
      "threshold": 1000,
      "forms": ["IT-201"]
    }
  }
};
```

2. Agreement Templates Repo - Schema Improvements

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Loan Agreement v2.0",
  "type": "object",
  "required": [
    "metadata",
    "parties", 
    "loanTerms",
    "taxSection",
    "signatures",
    "compliance"
  ],
  "properties": {
    "metadata": {
      "type": "object",
      "properties": {
        "agreementId": {"type": "string"},
        "version": {"type": "string", "pattern": "^\\d+\\.\\d+$"},
        "generatedDate": {"type": "string", "format": "date-time"},
        "agreementHash": {"type": "string"},
        "previousHash": {"type": "string"},
        "templateVersion": {"type": "string"}
      },
      "required": ["agreementId", "version", "generatedDate", "agreementHash"]
    },
    "parties": {
      "type": "object",
      "properties": {
        "lender": {
          "type": "object",
          "properties": {
            "name": {"type": "string"},
            "entityType": {"type": "string"},
            "taxId": {"type": "string"},
            "address": {"type": "string"},
            "signatory": {
              "type": "object",
              "properties": {
                "name": {"type": "string"},
                "title": {"type": "string"},
                "authority": {"type": "string"}
              }
            }
          },
          "required": ["name", "entityType", "taxId"]
        },
        "borrower": {
          "type": "object",
          "properties": {
            "name": {"type": "string"},
            "entityType": {"type": "string"},
            "taxId": {"type": "string"},
            "address": {"type": "string"},
            "jurisdiction": {"type": "string"},
            "signatory": {
              "type": "object",
              "properties": {
                "name": {"type": "string"},
                "title": {"type": "string"},
                "authority": {"type": "string"}
              }
            }
          },
          "required": ["name", "entityType", "taxId", "jurisdiction"]
        }
      },
      "required": ["lender", "borrower"]
    },
    "compliance": {
      "type": "object",
      "properties": {
        "afrCertification": {
          "type": "object",
          "properties": {
            "period": {"type": "string"},
            "rate": {"type": "number"},
            "compliance": {"type": "boolean"},
            "verificationDate": {"type": "string", "format": "date"}
          }
        },
        "bonaFideDebt": {
          "type": "object",
          "properties": {
            "certified": {"type": "boolean"},
            "certificationDate": {"type": "string", "format": "date"},
            "basis": {"type": "string"}
          }
        },
        "signatureRules": {
          "type": "object",
          "properties": {
            "order": {"type": "array", "items": {"type": "string"}},
            "required": {"type": "array", "items": {"type": "string"}},
            "witnessRequired": {"type": "boolean"},
            "notaryRequired": {"type": "boolean"}
          }
        }
      }
    }
  }
}
```

3. Loaner Ledger Repo - Blockchain-like Enhancement

```json
{
  "ledgerSchema": {
    "entry": {
      "type": "object",
      "required": ["id", "previousHash", "hash", "timestamp", "type", "data"],
      "properties": {
        "id": {"type": "string"},
        "previousHash": {
          "type": "string",
          "description": "Hash of previous entry for chain integrity"
        },
        "hash": {
          "type": "string",
          "description": "SHA-256 hash of this entry"
        },
        "timestamp": {"type": "string", "format": "date-time"},
        "type": {
          "type": "string",
          "enum": ["loan_issuance", "payment", "adjustment", "write_off", "error"]
        },
        "data": {"type": "object"},
        "metadata": {
          "type": "object",
          "properties": {
            "postedBy": {"type": "string"},
            "approvedBy": {"type": "string"},
            "approvalDate": {"type": "string", "format": "date-time"},
            "source": {"type": "string"},
            "version": {"type": "string"}
          }
        },
        "taxBlock": {
          "type": "object",
          "properties": {
            "federal": {
              "type": "object",
              "properties": {
                "classification": {"type": "string"},
                "incomeType": {"type": "string"},
                "reportingYear": {"type": "integer"},
                "form": {"type": "string"}
              }
            },
            "state": {
              "type": "object",
              "properties": {
                "jurisdiction": {"type": "string"},
                "classification": {"type": "string"},
                "reportingRequired": {"type": "boolean"}
              }
            }
          }
        },
        "status": {
          "type": "string",
          "enum": ["pending", "posted", "reconciled", "audited", "archived"]
        },
        "auditTrail": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "action": {"type": "string"},
              "user": {"type": "string"},
              "timestamp": {"type": "string", "format": "date-time"},
              "changes": {"type": "object"}
            }
          }
        }
      }
    }
  }
}
```

4. Enhanced README with Legal Basis & Security

```markdown
# Personal Credit Authority v3 - Complete Financial Framework

## Legal Basis & Compliance

### Applicable Federal Rate (AFR) Compliance
- **Source**: IRS Revenue Rulings, Section 1274(d)
- **Requirement**: All loans must meet or exceed AFR for corresponding term
- **Documentation**: Monthly AFR table verification with audit trail
- **Exceptions**: Only allowable for bona fide gift loans under $10,000

### Bona Fide Debt Requirements
1. **Written Agreement** - Executed contract with terms
2. **Enforceable Promise** - Legal obligation to repay
3. **Reasonable Expectation** - Demonstrated repayment capacity
4. **Actual Consideration** - Cash transfer with documentation
5. **AFR Compliance** - Interest at or above federal minimum

### IRS Imputed Interest Rules (Section 7872)
- **Below-Market Loans**: Interest imputed at AFR
- **Gift Loans**: Tax consequences for lender
- **Reporting**: Form 1099-INT for imputed interest
- **Exceptions**: Compensation-related or corporation-shareholder loans under $10,000

### State Reporting Obligations
- **Varies by Jurisdiction**: CA, NY, IL have specific requirements
- **Thresholds**: Typically $600-$2,500 annual interest
- **Forms**: State-specific interest reporting forms
- **Withholding**: Some states require withholding on non-resident interest

## Security Model

### Repository Sensitivity Levels
| Level | Repositories | Access | Encryption |
|-------|-------------|---------|------------|
| **P1** | agreements-repo, ledger-repo | Founders + Legal | End-to-end |
| **P2** | workflow-engine | Founders + Developers | At-rest |
| **P3** | templates, schemas | Public-read, Write-controlled | None |

### Access Control Implementation
```yaml
access_control:
  owners:
    - RickCreator87
    - GitDigital-LLC
  
  legal_team:
    permissions: [read, verify, approve]
    mfa_required: true
    audit_logging: true
  
  developers:
    permissions: [read, deploy]
    repositories: [workflow-engine]
    ip_restrictions: ["corporate-vpn"]
  
  auditors:
    permissions: [read-only]
    time_bound: "2024-01-01 to 2024-12-31"
```

Signature Verification Workflow

1. Generation: Agreement hash created from content
2. Signing: Digital signature using PGP/GPG keys
3. Verification: Hash comparison on retrieval
4. Storage: Signed documents in encrypted storage
5. Audit: Each verification logged with timestamp

Implementation Priority

Phase 1: Core Infrastructure (Week 1-2)

1. ✅ Complete missing method implementations
2. ✅ Add deterministic ID generation
3. ✅ Implement error handling framework
4. ✅ Create AFR validation service

Phase 2: Tax & Compliance (Week 3-4)

1. ✅ Enhance tax classification with state rules
2. ✅ Add imputed interest calculation
3. ✅ Implement compliance rule engine
4. ✅ Create audit trail system

Phase 3: Security & Integration (Week 5-6)

1. ✅ Add cryptographic verification
2. ✅ Implement access controls
3. ✅ Create API endpoints
4. ✅ Build monitoring dashboard

Phase 4: Production Readiness (Week 7-8)

1. ✅ Complete documentation
2. ✅ Security audit
3. ✅ Compliance review
4. ✅ Deployment automation

```

## 5. Enhanced Founder Loan Workflow

```json
{
  "workflowId": "GDP-FOUNDER-001",
  "version": "2.0",
  "status": "initiated",
  "createdAt": "2024-01-01T10:00:00Z",
  
  "parties": {
    "lender": {
      "name": "RickCreator87 Personal Credit Authority",
      "taxId": "XXX-XX-XXXX",
      "entity": "Individual Lender",
      "signatory": "Rick Creator",
      "authority": "Sole Proprietor"
    },
    "borrower": {
      "name": "GitDigital Products LLC",
      "taxId": "XX-XXXXXXX",
      "entity": "Delaware LLC",
      "signatory": "Co-Founder A",
      "authority": "Managing Member"
    },
    "guarantors": [
      {
        "name": "Founder A",
        "role": "Personal Guarantor",
        "liability": "Joint and Several"
      },
      {
        "name": "Founder B", 
        "role": "Personal Guarantor",
        "liability": "Joint and Several"
      }
    ]
  },
  
  "financialTerms": {
    "principal": 25000.00,
    "currency": "USD",
    "interestRate": 5.5,
    "termMonths": 60,
    "paymentSchedule": "Monthly",
    "firstPaymentDate": "2024-02-15",
    "maturityDate": "2028-12-15"
  },
  
  "afrCompliance": {
    "reference": {
      "period": "January 2024",
      "shortTerm": 4.57,
      "midTerm": 4.01,
      "longTerm": 3.73,
      "source": "IRS Rev. Rul. 2024-1",
      "verifiedDate": "2024-01-05"
    },
    "validation": {
      "complies": true,
      "margin": 1.49,
      "certification": "Bona Fide Debt Instrument"
    }
  },
  
  "taxTreatment": {
    "federal": {
      "classification": "Business Debt Instrument",
      "interestTreatment": "Taxable Interest Income",
      "imputedInterest": false,
      "formsRequired": ["1099-INT"],
      "filingDeadline": "2025-01-31"
    },
    "state": {
      "jurisdiction": "CA",
      "classification": "Business Loan",
      "reportingRequired": true,
      "formsRequired": ["CA 592", "CA 592-B"],
      "thresholdExceeded": true,
      "filingDeadline": "2025-01-31"
    }
  },
  
  "documentation": {
    "agreement": {
      "id": "AGMT-GDP-001",
      "version": "1.0",
      "hash": "sha256:abc123...",
      "url": "/agreements/AGMT-GDP-001.pdf",
      "signedDate": "2024-01-15"
    },
    "ledgerEntries": [
      {
        "id": "LEDGER-GDP-001",
        "type": "loan_issuance",
        "url": "/ledger/entries/LEDGER-GDP-001.json",
        "hash": "sha256:def456..."
      },
      {
        "id": "LEDGER-PERSONAL-001",
        "type": "personal_ledger",
        "url": "/personal-ledger/entries/001.json"
      }
    ]
  },
  
  "governance": {
    "approvalWorkflow": {
      "required approval"
}
```
