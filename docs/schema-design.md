# Enhanced Database Schema Design

> **Comprehensive database architecture for 92% C-Suite report coverage**

## Architecture Overview

The enhanced schema transforms the current 4-table structure into a comprehensive **star schema design** optimized for analytical reporting and operational efficiency. The design supports **cross-report integration** while maintaining **performance** and **scalability**.

### Design Principles
- **Dimensional Modeling**: Star schema for optimal query performance
- **Normalized Core**: Transactional integrity with analytical efficiency
- **Extensible Framework**: Easy addition of new capabilities
- **Integration Ready**: API-friendly structure for external connections

## 🏗️ Enhanced Schema Architecture

### Core Dimensions (Master Data)

#### 1. Enhanced Suppliers Dimension
```sql
CREATE TABLE dim_suppliers (
    supplier_id INTEGER PRIMARY KEY,
    supplier_code VARCHAR(20) UNIQUE NOT NULL,
    supplier_name VARCHAR(255) NOT NULL,
    
    -- Contact Information
    primary_contact_name VARCHAR(100),
    primary_contact_email VARCHAR(150),
    primary_contact_phone VARCHAR(50),
    
    -- Geographic Data
    street_address VARCHAR(255),
    city VARCHAR(100),
    state_province VARCHAR(100),
    postal_code VARCHAR(20),
    country_code CHAR(3),
    country_name VARCHAR(100),
    region_classification VARCHAR(50), -- APAC, EMEA, AMERICAS
    
    -- Business Classification
    business_size VARCHAR(20), -- SME, LARGE, ENTERPRISE
    industry_code VARCHAR(10),
    industry_description VARCHAR(100),
    supplier_type VARCHAR(30), -- STRATEGIC, PREFERRED, APPROVED, TACTICAL
    diversity_certifications TEXT, -- JSON array of certifications
    
    -- Financial & Risk Data
    annual_revenue DECIMAL(15,2),
    employee_count INTEGER,
    credit_rating VARCHAR(10),
    risk_score DECIMAL(3,2), -- 0.00 to 5.00
    esg_score DECIMAL(3,2), -- 0.00 to 100.00
    
    -- Operational Flags
    is_active BOOLEAN DEFAULT TRUE,
    is_preferred BOOLEAN DEFAULT FALSE,
    is_strategic BOOLEAN DEFAULT FALSE,
    is_diverse_supplier BOOLEAN DEFAULT FALSE,
    
    -- Audit Fields
    created_date DATETIME DEFAULT CURRENT_TIMESTAMP,
    last_updated DATETIME DEFAULT CURRENT_TIMESTAMP,
    created_by VARCHAR(50),
    last_updated_by VARCHAR(50)
);
```

#### 2. Enhanced Categories Dimension
```sql
CREATE TABLE dim_categories (
    category_id INTEGER PRIMARY KEY,
    category_code VARCHAR(20) UNIQUE NOT NULL,
    category_name VARCHAR(100) NOT NULL,
    
    -- Hierarchical Structure
    parent_category_id INTEGER,
    category_level INTEGER, -- 1=Level1, 2=Level2, etc.
    category_path VARCHAR(500), -- /Level1/Level2/Level3
    
    -- Strategic Classification
    strategic_importance VARCHAR(20), -- CRITICAL, HIGH, MEDIUM, LOW
    sourcing_complexity VARCHAR(20), -- SIMPLE, MODERATE, COMPLEX
    market_maturity VARCHAR(20), -- EMERGING, GROWING, MATURE, DECLINING
    
    -- Financial Thresholds
    approval_threshold DECIMAL(12,2),
    competitive_threshold DECIMAL(12,2),
    
    -- Category Manager
    category_manager_id INTEGER,
    category_manager_name VARCHAR(100),
    
    -- Audit Fields
    is_active BOOLEAN DEFAULT TRUE,
    created_date DATETIME DEFAULT CURRENT_TIMESTAMP,
    last_updated DATETIME DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (parent_category_id) REFERENCES dim_categories(category_id)
);
```

#### 3. Enhanced Users Dimension  
```sql
CREATE TABLE dim_users (
    user_id INTEGER PRIMARY KEY,
    employee_id VARCHAR(20) UNIQUE,
    username VARCHAR(50) UNIQUE NOT NULL,
    
    -- Personal Information
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(150) UNIQUE NOT NULL,
    phone VARCHAR(50),
    
    -- Organizational Data
    department VARCHAR(100),
    cost_center VARCHAR(20),
    manager_user_id INTEGER,
    job_title VARCHAR(100),
    
    -- Procurement Authority
    approval_limit DECIMAL(12,2),
    can_approve_contracts BOOLEAN DEFAULT FALSE,
    can_create_suppliers BOOLEAN DEFAULT FALSE,
    procurement_role VARCHAR(50), -- REQUESTER, BUYER, APPROVER, ADMIN
    
    -- System Access
    is_active BOOLEAN DEFAULT TRUE,
    last_login DATETIME,
    created_date DATETIME DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (manager_user_id) REFERENCES dim_users(user_id)
);
```

#### 4. Date Dimension
```sql
CREATE TABLE dim_date (
    date_key INTEGER PRIMARY KEY, -- YYYYMMDD format
    full_date DATE NOT NULL,
    
    -- Date Components
    year INTEGER,
    quarter INTEGER,
    month INTEGER,
    week_of_year INTEGER,
    day_of_year INTEGER,
    day_of_month INTEGER,
    day_of_week INTEGER,
    
    -- Descriptive Fields
    month_name VARCHAR(10),
    day_name VARCHAR(10),
    quarter_name VARCHAR(2), -- Q1, Q2, Q3, Q4
    
    -- Business Calendar
    is_weekday BOOLEAN,
    is_weekend BOOLEAN,
    is_holiday BOOLEAN,
    holiday_name VARCHAR(100),
    fiscal_year INTEGER,
    fiscal_quarter INTEGER,
    fiscal_month INTEGER,
    
    -- Period Flags
    is_month_end BOOLEAN,
    is_quarter_end BOOLEAN,
    is_year_end BOOLEAN
);
```

### Fact Tables (Transactional Data)

#### 1. Enhanced Purchase Orders Fact
```sql
CREATE TABLE fact_purchase_orders (
    po_fact_id INTEGER PRIMARY KEY,
    
    -- Dimension Keys
    po_id INTEGER UNIQUE NOT NULL,
    supplier_id INTEGER NOT NULL,
    requester_user_id INTEGER NOT NULL,
    approver_user_id INTEGER,
    category_id INTEGER NOT NULL,
    po_date_key INTEGER NOT NULL,
    approval_date_key INTEGER,
    
    -- Financial Measures
    po_amount DECIMAL(15,2) NOT NULL,
    po_amount_usd DECIMAL(15,2), -- Standardized currency
    tax_amount DECIMAL(15,2),
    total_amount DECIMAL(15,2),
    currency_code CHAR(3),
    exchange_rate DECIMAL(10,6),
    
    -- Operational Measures
    line_item_count INTEGER,
    delivery_days_requested INTEGER,
    
    -- Status and Workflow
    po_status VARCHAR(20), -- DRAFT, PENDING, APPROVED, ISSUED, RECEIVED, CLOSED
    approval_workflow_id INTEGER,
    
    -- Performance Tracking
    created_to_approved_days INTEGER,
    approved_to_issued_days INTEGER,
    
    -- Audit Trail
    created_datetime DATETIME DEFAULT CURRENT_TIMESTAMP,
    last_updated_datetime DATETIME,
    
    FOREIGN KEY (supplier_id) REFERENCES dim_suppliers(supplier_id),
    FOREIGN KEY (requester_user_id) REFERENCES dim_users(user_id),
    FOREIGN KEY (approver_user_id) REFERENCES dim_users(user_id),
    FOREIGN KEY (category_id) REFERENCES dim_categories(category_id),
    FOREIGN KEY (po_date_key) REFERENCES dim_date(date_key)
);
```

#### 2. Line Items Fact
```sql
CREATE TABLE fact_line_items (
    line_item_fact_id INTEGER PRIMARY KEY,
    
    -- Dimension Keys
    line_item_id INTEGER UNIQUE NOT NULL,
    po_fact_id INTEGER NOT NULL,
    supplier_id INTEGER NOT NULL,
    category_id INTEGER NOT NULL,
    
    -- Product Information
    product_code VARCHAR(50),
    product_description TEXT,
    unit_of_measure VARCHAR(20),
    
    -- Financial Measures
    quantity DECIMAL(12,3),
    unit_price DECIMAL(10,4),
    line_total DECIMAL(15,2),
    unit_price_usd DECIMAL(10,4),
    line_total_usd DECIMAL(15,2),
    
    -- Delivery Information
    requested_delivery_date DATE,
    promised_delivery_date DATE,
    actual_delivery_date DATE,
    
    -- Quality and Compliance
    quality_requirements TEXT,
    compliance_flags TEXT, -- JSON array
    
    FOREIGN KEY (po_fact_id) REFERENCES fact_purchase_orders(po_fact_id),
    FOREIGN KEY (supplier_id) REFERENCES dim_suppliers(supplier_id),
    FOREIGN KEY (category_id) REFERENCES dim_categories(category_id)
);
```

### Contract Management Tables

#### 1. Contracts Master
```sql
CREATE TABLE contracts (
    contract_id INTEGER PRIMARY KEY,
    contract_number VARCHAR(50) UNIQUE NOT NULL,
    contract_name VARCHAR(200) NOT NULL,
    
    -- Parties
    supplier_id INTEGER NOT NULL,
    contract_owner_user_id INTEGER NOT NULL,
    legal_entity VARCHAR(100),
    
    -- Contract Details
    contract_type VARCHAR(50), -- MASTER, CALL_OFF, SPOT, FRAMEWORK
    contract_status VARCHAR(20), -- DRAFT, ACTIVE, EXPIRED, TERMINATED
    
    -- Financial Terms
    contract_value DECIMAL(15,2),
    currency_code CHAR(3),
    payment_terms_days INTEGER,
    early_payment_discount DECIMAL(5,2),
    
    -- Dates
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    notice_period_days INTEGER,
    auto_renewal_flag BOOLEAN DEFAULT FALSE,
    renewal_notice_days INTEGER,
    
    -- Terms and Conditions
    liability_cap DECIMAL(15,2),
    has_sla_terms BOOLEAN DEFAULT FALSE,
    has_kpi_terms BOOLEAN DEFAULT FALSE,
    termination_clause TEXT,
    
    -- Document Management
    contract_document_path VARCHAR(500),
    contract_document_hash VARCHAR(64),
    
    -- Audit Fields
    created_date DATETIME DEFAULT CURRENT_TIMESTAMP,
    last_updated DATETIME DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (supplier_id) REFERENCES dim_suppliers(supplier_id),
    FOREIGN KEY (contract_owner_user_id) REFERENCES dim_users(user_id)
);
```

#### 2. Contract Performance Tracking
```sql
CREATE TABLE contract_performance (
    performance_id INTEGER PRIMARY KEY,
    contract_id INTEGER NOT NULL,
    measurement_date DATE NOT NULL,
    
    -- KPI Measurements
    kpi_name VARCHAR(100) NOT NULL,
    target_value DECIMAL(10,4),
    actual_value DECIMAL(10,4),
    measurement_unit VARCHAR(20),
    
    -- Performance Status
    performance_status VARCHAR(20), -- ABOVE_TARGET, MET, BELOW_TARGET, CRITICAL
    variance_percentage DECIMAL(5,2),
    
    -- Actions
    corrective_action_required BOOLEAN DEFAULT FALSE,
    corrective_action_description TEXT,
    
    FOREIGN KEY (contract_id) REFERENCES contracts(contract_id)
);
```

### Supplier Performance Tables

#### 1. Supplier Performance Metrics
```sql
CREATE TABLE supplier_performance_metrics (
    metric_id INTEGER PRIMARY KEY,
    supplier_id INTEGER NOT NULL,
    measurement_period_start DATE NOT NULL,
    measurement_period_end DATE NOT NULL,
    
    -- Delivery Performance
    total_orders INTEGER DEFAULT 0,
    on_time_orders INTEGER DEFAULT 0,
    in_full_orders INTEGER DEFAULT 0,
    otif_orders INTEGER DEFAULT 0,
    otif_percentage DECIMAL(5,2),
    average_lead_time_days DECIMAL(5,1),
    
    -- Quality Performance
    total_receipts INTEGER DEFAULT 0,
    defective_receipts INTEGER DEFAULT 0,
    defect_rate DECIMAL(5,4),
    quality_score DECIMAL(3,1), -- 1.0 to 10.0
    
    -- Financial Performance
    total_invoices INTEGER DEFAULT 0,
    accurate_invoices INTEGER DEFAULT 0,
    invoice_accuracy_rate DECIMAL(5,2),
    average_payment_days DECIMAL(5,1),
    
    -- Service Performance
    responsiveness_score DECIMAL(3,1), -- 1.0 to 10.0
    communication_score DECIMAL(3,1), -- 1.0 to 10.0
    overall_satisfaction DECIMAL(3,1), -- 1.0 to 10.0
    
    -- Calculated Scores
    overall_performance_score DECIMAL(5,2), -- Weighted average
    performance_tier VARCHAR(20), -- EXCELLENT, GOOD, ACCEPTABLE, POOR
    
    FOREIGN KEY (supplier_id) REFERENCES dim_suppliers(supplier_id)
);
```

#### 2. Supplier Risk Assessments
```sql
CREATE TABLE supplier_risk_assessments (
    assessment_id INTEGER PRIMARY KEY,
    supplier_id INTEGER NOT NULL,
    assessment_date DATE NOT NULL,
    assessment_type VARCHAR(50), -- ANNUAL, QUARTERLY, AD_HOC, INCIDENT
    
    -- Risk Categories (Scale 1-5, 5 being highest risk)
    financial_risk_score DECIMAL(2,1),
    operational_risk_score DECIMAL(2,1),
    strategic_risk_score DECIMAL(2,1),
    reputational_risk_score DECIMAL(2,1),
    compliance_risk_score DECIMAL(2,1),
    cybersecurity_risk_score DECIMAL(2,1),
    geographic_risk_score DECIMAL(2,1),
    
    -- Overall Risk
    overall_risk_score DECIMAL(3,2),
    risk_rating VARCHAR(20), -- LOW, MEDIUM, HIGH, CRITICAL
    
    -- Risk Details
    key_risks_identified TEXT,
    mitigation_actions TEXT,
    next_review_date DATE,
    
    -- Assessment Metadata
    assessed_by_user_id INTEGER,
    assessment_method VARCHAR(50), -- MANUAL, AUTOMATED, HYBRID
    
    FOREIGN KEY (supplier_id) REFERENCES dim_suppliers(supplier_id),
    FOREIGN KEY (assessed_by_user_id) REFERENCES dim_users(user_id)
);
```

### Savings and Value Tracking

#### 1. Savings Initiatives
```sql
CREATE TABLE savings_initiatives (
    initiative_id INTEGER PRIMARY KEY,
    initiative_name VARCHAR(200) NOT NULL,
    initiative_type VARCHAR(50), -- NEGOTIATION, BUNDLING, SPEC_CHANGE, SOURCING
    
    -- Scope
    category_id INTEGER,
    supplier_id INTEGER,
    business_unit VARCHAR(100),
    
    -- Financial Projections
    baseline_annual_spend DECIMAL(15,2),
    projected_annual_savings DECIMAL(15,2),
    savings_percentage DECIMAL(5,2),
    one_time_savings DECIMAL(15,2),
    
    -- Timeline
    initiative_start_date DATE,
    expected_completion_date DATE,
    actual_completion_date DATE,
    
    -- Status Tracking
    initiative_status VARCHAR(20), -- PLANNED, IN_PROGRESS, COMPLETED, CANCELLED
    completion_percentage DECIMAL(5,2),
    
    -- Ownership
    initiative_owner_user_id INTEGER,
    business_sponsor_user_id INTEGER,
    
    -- Documentation
    business_case TEXT,
    implementation_plan TEXT,
    
    FOREIGN KEY (category_id) REFERENCES dim_categories(category_id),
    FOREIGN KEY (supplier_id) REFERENCES dim_suppliers(supplier_id),
    FOREIGN KEY (initiative_owner_user_id) REFERENCES dim_users(user_id)
);
```

#### 2. Savings Realization Tracking
```sql
CREATE TABLE savings_realization (
    realization_id INTEGER PRIMARY KEY,
    initiative_id INTEGER NOT NULL,
    measurement_period DATE NOT NULL, -- Monthly measurement
    
    -- Baseline Comparison
    baseline_spend DECIMAL(15,2),
    actual_spend DECIMAL(15,2),
    variance_amount DECIMAL(15,2),
    
    -- Savings Calculation
    realized_savings DECIMAL(15,2),
    cumulative_savings DECIMAL(15,2),
    realization_percentage DECIMAL(5,2),
    
    -- Validation Status
    finance_validated BOOLEAN DEFAULT FALSE,
    validation_date DATE,
    validated_by_user_id INTEGER,
    
    -- Variance Explanation
    variance_reason VARCHAR(500),
    corrective_action TEXT,
    
    FOREIGN KEY (initiative_id) REFERENCES savings_initiatives(initiative_id),
    FOREIGN KEY (validated_by_user_id) REFERENCES dim_users(user_id)
);
```

### Demand Planning and Forecasting

#### 1. Demand Forecasts
```sql
CREATE TABLE demand_forecasts (
    forecast_id INTEGER PRIMARY KEY,
    category_id INTEGER NOT NULL,
    forecast_period_start DATE NOT NULL,
    forecast_period_end DATE NOT NULL,
    
    -- Forecast Details
    forecast_type VARCHAR(50), -- ANNUAL, QUARTERLY, MONTHLY
    forecast_method VARCHAR(50), -- STATISTICAL, MANUAL, HYBRID
    
    -- Demand Projections
    forecasted_volume DECIMAL(15,3),
    forecasted_value DECIMAL(15,2),
    currency_code CHAR(3),
    
    -- Confidence and Accuracy
    confidence_level DECIMAL(3,1), -- Percentage
    forecast_accuracy DECIMAL(5,2), -- Historical accuracy
    
    -- Business Context
    business_assumptions TEXT,
    market_factors TEXT,
    risk_factors TEXT,
    
    -- Ownership
    created_by_user_id INTEGER,
    approved_by_user_id INTEGER,
    approval_date DATE,
    
    -- Version Control
    forecast_version INTEGER DEFAULT 1,
    is_current_version BOOLEAN DEFAULT TRUE,
    
    FOREIGN KEY (category_id) REFERENCES dim_categories(category_id),
    FOREIGN KEY (created_by_user_id) REFERENCES dim_users(user_id)
);
```

#### 2. Demand vs Actual Tracking
```sql
CREATE TABLE demand_actual_comparison (
    comparison_id INTEGER PRIMARY KEY,
    forecast_id INTEGER NOT NULL,
    actual_period DATE NOT NULL,
    
    -- Actual vs Forecast
    forecasted_demand DECIMAL(15,3),
    actual_demand DECIMAL(15,3),
    variance_volume DECIMAL(15,3),
    variance_percentage DECIMAL(5,2),
    
    -- Value Comparison
    forecasted_value DECIMAL(15,2),
    actual_value DECIMAL(15,2),
    value_variance DECIMAL(15,2),
    
    -- Analysis
    variance_category VARCHAR(50), -- MARKET_CHANGE, BUSINESS_CHANGE, FORECAST_ERROR
    variance_explanation TEXT,
    
    FOREIGN KEY (forecast_id) REFERENCES demand_forecasts(forecast_id)
);
```

### Compliance and Audit Tracking

#### 1. Compliance Events
```sql
CREATE TABLE compliance_events (
    event_id INTEGER PRIMARY KEY,
    event_type VARCHAR(50), -- POLICY_VIOLATION, AUDIT_FINDING, CORRECTIVE_ACTION
    
    -- Scope
    supplier_id INTEGER,
    contract_id INTEGER,
    po_fact_id INTEGER,
    user_id INTEGER,
    
    -- Event Details
    event_date DATE NOT NULL,
    severity_level VARCHAR(20), -- LOW, MEDIUM, HIGH, CRITICAL
    compliance_area VARCHAR(100), -- FINANCIAL, OPERATIONAL, LEGAL, ESG
    
    -- Description
    event_description TEXT NOT NULL,
    root_cause_analysis TEXT,
    
    -- Resolution
    corrective_action TEXT,
    action_owner_user_id INTEGER,
    target_resolution_date DATE,
    actual_resolution_date DATE,
    resolution_status VARCHAR(20), -- OPEN, IN_PROGRESS, RESOLVED, CLOSED
    
    -- Impact Assessment
    financial_impact DECIMAL(15,2),
    reputational_impact VARCHAR(20), -- LOW, MEDIUM, HIGH
    operational_impact VARCHAR(20), -- LOW, MEDIUM, HIGH
    
    FOREIGN KEY (supplier_id) REFERENCES dim_suppliers(supplier_id),
    FOREIGN KEY (contract_id) REFERENCES contracts(contract_id),
    FOREIGN KEY (user_id) REFERENCES dim_users(user_id)
);
```

### External Data Integration Tables

#### 1. External Data Sources
```sql
CREATE TABLE external_data_sources (
    source_id INTEGER PRIMARY KEY,
    source_name VARCHAR(100) NOT NULL,
    source_type VARCHAR(50), -- MARKET_DATA, RISK_INTELLIGENCE, REGULATORY
    provider_name VARCHAR(100),
    
    -- Integration Details
    api_endpoint VARCHAR(500),
    update_frequency VARCHAR(50), -- REAL_TIME, DAILY, WEEKLY, MONTHLY
    last_successful_update DATETIME,
    next_scheduled_update DATETIME,
    
    -- Data Quality
    data_quality_score DECIMAL(3,2),
    availability_score DECIMAL(3,2),
    
    -- Business Value
    annual_cost DECIMAL(10,2),
    business_justification TEXT,
    reports_impacted TEXT, -- JSON array
    
    -- Status
    is_active BOOLEAN DEFAULT TRUE,
    integration_status VARCHAR(20) -- ACTIVE, PAUSED, ERROR, MAINTENANCE
);
```

### Performance and Analytics Views

#### 1. Supplier Scorecard View
```sql
CREATE VIEW v_supplier_scorecards AS
SELECT 
    s.supplier_id,
    s.supplier_name,
    s.supplier_type,
    s.country_name,
    
    -- Performance Metrics (Latest Period)
    p.otif_percentage,
    p.defect_rate,
    p.overall_performance_score,
    p.performance_tier,
    
    -- Risk Assessment (Latest)
    r.overall_risk_score,
    r.risk_rating,
    
    -- Financial Summary (YTD)
    SUM(po.po_amount_usd) as ytd_spend,
    COUNT(DISTINCT po.po_id) as ytd_orders,
    AVG(po.po_amount_usd) as avg_order_value,
    
    -- Contract Status
    c.contract_status,
    c.end_date as contract_expiry,
    
    -- Latest Update
    GREATEST(p.measurement_period_end, r.assessment_date) as last_updated

FROM dim_suppliers s
LEFT JOIN supplier_performance_metrics p ON s.supplier_id = p.supplier_id
    AND p.measurement_period_end = (
        SELECT MAX(measurement_period_end) 
        FROM supplier_performance_metrics p2 
        WHERE p2.supplier_id = s.supplier_id
    )
LEFT JOIN supplier_risk_assessments r ON s.supplier_id = r.supplier_id
    AND r.assessment_date = (
        SELECT MAX(assessment_date) 
        FROM supplier_risk_assessments r2 
        WHERE r2.supplier_id = s.supplier_id
    )
LEFT JOIN fact_purchase_orders po ON s.supplier_id = po.supplier_id
    AND po.po_date_key >= CAST(strftime('%Y', 'now') || '0101' AS INTEGER)
LEFT JOIN contracts c ON s.supplier_id = c.supplier_id
    AND c.contract_status = 'ACTIVE'

WHERE s.is_active = TRUE
GROUP BY s.supplier_id, s.supplier_name, s.supplier_type, s.country_name,
         p.otif_percentage, p.defect_rate, p.overall_performance_score, 
         p.performance_tier, r.overall_risk_score, r.risk_rating,
         c.contract_status, c.end_date;
```

## 🎯 Performance Optimization Strategy

### Indexing Strategy
```sql
-- Primary performance indexes
CREATE INDEX idx_fact_po_supplier_date ON fact_purchase_orders(supplier_id, po_date_key);
CREATE INDEX idx_fact_po_category_date ON fact_purchase_orders(category_id, po_date_key);
CREATE INDEX idx_fact_po_requester_date ON fact_purchase_orders(requester_user_id, po_date_key);

-- Supplier performance indexes
CREATE INDEX idx_supplier_perf_supplier_period ON supplier_performance_metrics(supplier_id, measurement_period_end);
CREATE INDEX idx_supplier_risk_supplier_date ON supplier_risk_assessments(supplier_id, assessment_date);

-- Contract management indexes
CREATE INDEX idx_contracts_supplier_status ON contracts(supplier_id, contract_status);
CREATE INDEX idx_contracts_expiry_status ON contracts(end_date, contract_status);

-- Savings tracking indexes
CREATE INDEX idx_savings_category_status ON savings_initiatives(category_id, initiative_status);
CREATE INDEX idx_savings_real_initiative_period ON savings_realization(initiative_id, measurement_period);
```

### Data Archiving Strategy
```sql
-- Archive old transactional data (>3 years)
CREATE TABLE fact_purchase_orders_archive AS 
SELECT * FROM fact_purchase_orders 
WHERE po_date_key < CAST((strftime('%Y', 'now') - 3) || '0101' AS INTEGER);

-- Archive old performance metrics (>2 years)
CREATE TABLE supplier_performance_metrics_archive AS
SELECT * FROM supplier_performance_metrics
WHERE measurement_period_end < date('now', '-2 years');
```

## 📊 Implementation Considerations

### Migration Strategy
1. **Phase 1**: Create new schema alongside existing tables
2. **Phase 2**: Migrate historical data with transformation
3. **Phase 3**: Implement new data flows and ETL processes
4. **Phase 4**: Switch reporting to new schema
5. **Phase 5**: Archive old schema after validation

### Data Quality Framework
- **Automated Validation**: Constraint-based data quality checks
- **Reference Data Management**: Centralized master data governance
- **Audit Trails**: Complete change tracking for all critical data
- **Data Lineage**: End-to-end data flow documentation

### Security and Access Control
- **Role-Based Access**: User permissions based on procurement roles
- **Data Encryption**: Sensitive data encrypted at rest and in transit
- **Audit Logging**: Complete access and change logging
- **Data Masking**: PII protection in non-production environments

---

**🎯 This enhanced schema provides the foundation for 92% C-Suite report coverage while maintaining performance, scalability, and data integrity.**