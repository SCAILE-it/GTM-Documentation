# GTM Dashboard MVP - 5-Day Sprint PRD

## Product Context & Description

**GTM Dashboard** is a B2B SaaS platform that provides go-to-market teams with a unified analytics and AI-powered business intelligence solution. The platform aggregates data from multiple marketing and sales tools (Google Analytics 4, Google Search Console, HubSpot) into a centralized BigQuery data warehouse, where users can interact with their data through natural language queries powered by Google's ADK (Agent Development Kit) framework.

### Core Value Proposition
- **Single Source of Truth**: Consolidate all GTM data in one place
- **AI-Powered Insights**: Natural language queries to complex data without SQL knowledge
- **Automated Visualization**: Instant charts and dashboards from conversational queries
- **Multi-tenant Architecture**: Secure, isolated data environments for each customer

### Target Users
Go-to-market professionals including:
- Marketing managers needing unified campaign performance views
- Sales leaders tracking funnel metrics across channels
- Revenue operations teams requiring cross-platform analytics
- Growth teams analyzing conversion paths and attribution

## Technical Architecture Overview

### Microservices Architecture
```
┌─────────────────────────────────────────────┐
│            API Gateway (Cloud Run)          │
│         Routes requests to services         │
└─────────────────────────────────────────────┘
                    │
    ┌───────────────┼───────────────┐
    ▼               ▼               ▼
┌──────────┐  ┌──────────┐  ┌───────────┐
│   Auth   │  │Analytics │  │Integration│
│ Service  │  │ Service  │  │  Service  │
└──────────┘  └──────────┘  └───────────┘
     │             │              │
     ▼             ▼              ▼
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Supabase │  │   ADK    │  │ BigQuery │
│ Postgres │  │  Agents  │  │ Datasets │
└──────────┘  └──────────┘  └──────────┘
```

### Data Flow Architecture
1. **User Authentication** → Supabase Auth (Google OAuth + Email/Password)
2. **Tenant Creation** → Isolated BigQuery dataset per customer
3. **Data Integration** → Native connectors + API ingestion to BigQuery
4. **AI Analysis** → ADK agents query tenant-specific views
5. **Visualization** → ChartSpec JSON rendered with Recharts

## Sprint Timeline & Team Division

> NOTE: For the MVP launch, we will only be showing integrations with GA4, GSC and a third tool that will be decided by Anton based on time constraints - HubSpot / Google Ads / Instantly / Phantom Buster, etc.

### Phase 1: Core Backend Setup (Day 1-2)
> **All team members collaborate on foundation**

#### Day 1: Infrastructure & Database Foundation

**Anton - Data Layer**
- Set up GCP project and BigQuery
- Create dataset template structure
- Design table schemas for GA4, GSC, HubSpot/Google Ads/etc. (Anton chooses)
- Document data types and partitioning strategy
```sql
-- Just suggestions:
-- Template for tenant datasets
-- ga4_events_* (partitioned by day)
-- gsc_search_data (partitioned by date)
-- hubspot_contacts (clustered by email)
-- hubspot_deals (clustered by deal_id)
```

**Yatharth - Auth & Tenant Management**
- Set up Supabase project
- Configure Google OAuth
- Create tenant management schema
- Implement RLS policies if needed
- Connect with a frontend Auth screen
```sql
-- Core tables in app schema - Just suggestions and must be refined:
CREATE SCHEMA app;
CREATE TABLE app.tenants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  owner_id UUID REFERENCES auth.users(id),
  bq_dataset TEXT NOT NULL,
  subscription_status TEXT DEFAULT 'trial',
  stripe_customer_id TEXT,
  stripe_subscription_id TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE app.tenant_memberships (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID REFERENCES app.tenants(id),
  user_id UUID REFERENCES auth.users(id),
  role TEXT DEFAULT 'member',
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE app.integrations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID REFERENCES app.tenants(id),
  source TEXT NOT NULL,
  status TEXT DEFAULT 'pending',
  config JSONB DEFAULT '{}',
  last_sync TIMESTAMPTZ,
  error_message TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

**Sid - FastAPI & ADK Setup**
- Initialize Cloud Run project structure
- Set up FastAPI with microservices architecture
- Deploy existing ADK project with Google Cloud Run
- Create base API structure
- Create analytics API endpoints
- Add preliminary logging

#### Day 2: Service Implementation & Views

**Anton - BigQuery Transformations, Views and Tool Integrations**
- Set up data transformation logic and views for each tool
- Implement API/ELT based integrations for each tool using BigQuery Data Transfer Service, HubSpot native integration, CSV upload, etc. for GA4, GSC and third chosen tool, each
- Essentially the Tenant should be able to call an endpoint to Trigger an end-to-end data loading process for each tool they start the integration process.
- Go for low hanging fruit for now and implement the data loading process for GA4, GSC and the third chosen tool each.

**Yatharth - Payment Integration & Tenant Services**
- Set up Stripe payment services
- Create subscription management tables and logic for each tenant in Supabase
- Implement billing logic (simplified for MVP)
- Connect with a frontend Payment screen


**Sid - Tenant Dataset Creation & API Services**
- Implement per-tenant dataset creation with JWT token from Supabase
- Set up integration status tracking
- Create management endpoints
```python
# Tenant Dataset Service
class TenantDatasetService:
    def __init__(self):
        self.bq_client = bigquery.Client()
    
    async def create_tenant_dataset(self, tenant_id: str):
        """Create isolated BigQuery dataset for tenant"""
        dataset_id = f"tenant_{tenant_id}"
        dataset = bigquery.Dataset(f"gtm-prod.{dataset_id}")
        dataset.location = "US"
        dataset.description = f"Data for tenant {tenant_id}"
        
        # Create dataset
        dataset = self.bq_client.create_dataset(dataset, exists_ok=True)
        
        # Apply default views
        await self.create_default_views(dataset_id)
        
        return dataset_id
    
    async def create_default_views(self, dataset_id: str):
        """Create standard analytical views"""
        views = [
            ("vw_sessions_daily", self.get_sessions_daily_sql()),
            ("vw_funnel_by_channel", self.get_funnel_sql()),
            ("vw_top_search_queries", self.get_search_queries_sql())
        ]
        
        for view_name, view_sql in views:
            query = view_sql.format(dataset_id=dataset_id)
            self.bq_client.query(query).result()

# Integration Status Service
@app.get("/v1/integrations/status/{tenant_id}")
async def get_integration_status(tenant_id: str):
    """Check status of all integrations"""
    statuses = {}
    
    # Check GA4
    ga4_tables = check_ga4_tables(tenant_id)
    statuses['ga4'] = {
        'connected': len(ga4_tables) > 0,
        'tables': ga4_tables,
        'last_sync': get_last_table_update(tenant_id, 'ga4_events')
    }
    
    # Check GSC
    gsc_data = check_gsc_data(tenant_id)
    statuses['gsc'] = {
        'connected': gsc_data['row_count'] > 0,
        'rows': gsc_data['row_count'],
        'last_sync': gsc_data['last_update']
    }
    
    # Check HubSpot
    hubspot_data = check_hubspot_data(tenant_id)
    statuses['hubspot'] = {
        'connected': hubspot_data['has_data'],
        'contacts': hubspot_data['contact_count'],
        'deals': hubspot_data['deal_count'],
        'last_sync': hubspot_data['last_sync']
    }
    
    return statuses
```
### Phase 2: Integration & Frontend Connection (Day 3-4)

#### Day 3: Data Integrations & Frontend Base

**Anton - Finish up on the Tool Integration Microservices**
- Refine the Tool Integration Microservices to be more robust
- Add API endpoint trigger for manual refresh for each tool
- Prepare everything (API endpoints, Microservices, etc.) for the FE to call to trigger the data loading process and storage in the BQ dataset per tenant with Sid

**Yatharth - Frontend Integration**
- Wire authentication -> subscription -> Chat interface frontend
- Connect chat interface to Backend FastAPI service facilitated by Cloud Run
- Connect dashboard -> pinned charts functionality to backend ADK API service as well as User's preferences in Supabase
- Skip integration setup for now - Assume that the integrations are already hardcoded
```jsx
// Auth setup with Supabase
import { createClient } from '@supabase/supabase-js'

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY
)

// Chat component
function ChatInterface() {
  const [messages, setMessages] = useState([])
  
  const sendMessage = async (question) => {
    const { data: { session } } = await supabase.auth.getSession()
    
    const response = await fetch('/api/v1/analytics/query', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${session.access_token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ question })
    })
    
    const result = await response.json()
    
    setMessages([...messages, {
      type: 'user',
      content: question
    }, {
      type: 'assistant',
      content: result.insights,
      chart: result.chart_spec,
      data: result.data
    }])
  }
  
  return (
    <div className="chat-container">
      <MessageList messages={messages} />
      <ChatInput onSend={sendMessage} />
    </div>
  )
}
```

**Sid - ADK BQ Connection + Tool Integration API Endpoints**
- Improve ADK agents for BigQuery access
- Help Anton with Tool Integrations Microservices setup and Deployment
- Add comprehensive error handling

#### Day 4: Integration UI & Advanced Features

**Anton - Microservices to Trigger data loading to BigQuery from UI**
- Finish and deploy the Tool Integration API Microservices for each of the 3 tools so that the FE can actually call these APIs from the deployed cloud Endpoints to trigger the data loading process and storage in the BQ dataset per tenant

**Yatharth - Integration UI & Dashboard**
- Build integration setup screens
- Utilise the Tool Integration API Endpoints to trigger the data loading process and storage in the BQ dataset per tenant
- Show feedback to the user on the status of the data loading process

**Sid - CI/CD & Monitoring**
- Set up Preliminary monitoring
- Setup Preliminary CI/CD pipeline

### Phase 3: Polish & Demo Preparation (Day 5)

#### Day 5: Testing & Demo Flow

**All Team Members - Morning (4 hours each)**
- End-to-end testing of complete flow
- Bug fixes from Day 4
- Performance optimization
- Data validation

**Demo Flow Checklist**
1. [ ] Sign up with Google OAuth
2. [ ] Automatic tenant creation with trial status
3. [ ] Connect GA4 (show linking instructions)
4. [ ] Connect GSC (show bulk export setup)
5. [ ] Connect HubSpot (use test account)
6. [ ] Ask: "What are my top traffic sources last 30 days?"
7. [ ] Receive chart and insights
8. [ ] Pin chart to dashboard
9. [ ] Ask: "Show conversion funnel by channel"
10. [ ] View dashboard with multiple pinned charts

## API Specification - Rough - Must be refined

### Authentication Endpoints
```
POST   /v1/auth/signup          - Create new user account
POST   /v1/auth/login           - Login with email/password
POST   /v1/auth/google          - Google OAuth flow
GET    /v1/auth/me              - Get current user & tenant
POST   /v1/auth/logout          - Logout user
```

### Tenant Management
```
POST   /v1/tenants              - Create new tenant
GET    /v1/tenants/current      - Get current tenant details
PUT    /v1/tenants/{id}         - Update tenant settings
POST   /v1/tenants/{id}/invite  - Invite team member
```

### Integration Management
```
GET    /v1/integrations                    - List all integrations
POST   /v1/integrations/ga4/setup          - Get GA4 setup instructions
POST   /v1/integrations/ga4/verify         - Verify GA4 connection
POST   /v1/integrations/gsc/setup          - Setup GSC export
POST   /v1/integrations/hubspot/connect    - Connect HubSpot
POST   /v1/integrations/hubspot/sync       - Trigger HubSpot sync
GET    /v1/integrations/status             - Get all integration statuses
```

### Analytics & AI
```
POST   /v1/analytics/query      - Send natural language query
GET    /v1/analytics/schema     - Get available data schema
POST   /v1/analytics/sql        - Execute raw SQL (admin only)
```

### Charts & Dashboard
```
POST   /v1/charts/pin           - Pin a chart
DELETE /v1/charts/{id}/unpin    - Unpin a chart
GET    /v1/charts/pinned        - Get all pinned charts
PUT    /v1/charts/{id}          - Update chart settings
```

## Data Models

### ChartSpec JSON Format
```json
{
  "type": "line|bar|area|pie|scatter|funnel",
  "title": "Chart Title",
  "data": [...],
  "config": {
    "xAxis": {
      "dataKey": "date",
      "label": "Date",
      "type": "category|number|date"
    },
    "yAxis": {
      "dataKey": "value",
      "label": "Sessions",
      "type": "number"
    },
    "series": [
      {
        "dataKey": "sessions",
        "name": "Sessions",
        "color": "#8884d8",
        "type": "line|bar|area"
      }
    ]
  },
  "insights": "Text explanation of the data"
}
```

### Integration Configuration
```json
{
  "ga4": {
    "property_id": "123456789",
    "dataset_id": "tenant_abc123",
    "export_type": "daily",
    "first_export_date": "2024-01-01"
  },
  "gsc": {
    "site_url": "https://example.com",
    "dataset_id": "tenant_abc123",
    "service_account": "search-console-data-export@system.gserviceaccount.com"
  },
  "hubspot": {
    "portal_id": "12345678",
    "access_token": "encrypted_token",
    "scopes": ["contacts", "deals"],
    "sync_frequency": "daily"
  }
}
```

## Security & Compliance

### Multi-tenant Isolation
- **Database Level**: Supabase RLS policies ensure users only access their tenant data
- **BigQuery Level**: Separate datasets per tenant with IAM controls
- **API Level**: JWT validation and tenant context injection

### Data Security
- All API endpoints require valid JWT tokens
- BigQuery access limited to specific views (vw_* prefix)
- Sensitive tokens stored encrypted in Secret Manager
- HTTPS/TLS for all data transmission

## Success Metrics

### MVP Demo Success Criteria
- [ ] 3 test users can sign up and create separate tenants
- [ ] Each user can connect at least one data source
- [ ] Natural language queries return accurate data
- [ ] Charts render correctly from ChartSpec
- [ ] Dashboard shows pinned charts persistently
- [ ] No data leakage between tenants

### Performance Targets
- API response time < 2 seconds
- BigQuery query time < 5 seconds
- Chart rendering < 1 second
- Authentication flow < 3 seconds

## Deployment Configuration

### Environment Variables
```env
# Supabase
SUPABASE_URL=https://xxx.supabase.co
SUPABASE_ANON_KEY=xxx
SUPABASE_SERVICE_KEY=xxx

# Google Cloud
GOOGLE_CLOUD_PROJECT=gtm-prod
GOOGLE_CLOUD_REGION=us-central1
BIGQUERY_DATASET_LOCATION=US

# ADK/Gemini
GOOGLE_AI_MODEL=gemini-2.0-flash
ADK_DEPLOYMENT_NAME=gtm-analytics-agent

# Stripe (for future)
STRIPE_SECRET_KEY=sk_live_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx

# API Configuration
API_BASE_URL=https://api.gtmdashboard.com
FRONTEND_URL=https://app.gtmdashboard.com
```

### Cloud Run Deployment
```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: gtm-api-gateway
spec:
  template:
    metadata:
      annotations:
        run.googleapis.com/cpu-throttling: "false"
    spec:
      containers:
      - image: gcr.io/gtm-prod/api-gateway:latest
        resources:
          limits:
            cpu: "2"
            memory: "2Gi"
        env:
        - name: ENVIRONMENT
          value: "production"
```

## Risk Mitigation

### Technical Risks
1. **GA4/GSC Export Delays**: Have pre-loaded demo data ready
2. **API Rate Limits**: Implement request queuing and caching
3. **BigQuery Costs**: Set up cost alerts and query limits
4. **Integration Failures**: Manual CSV upload as fallback

### Business Risks
1. **Demo Day Failure**: Record backup demo video
2. **Scale Issues**: Start with max 10 beta customers
3. **Data Accuracy**: Implement data validation checks
4. **Security Breach**: Regular security audits, pen testing

---