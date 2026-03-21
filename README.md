🫀 HealthVault — Personal Health & Insurance Tracker
A mobile-first web app for tracking annual health checkups, doctor visits, and insurance coverage across multiple family profiles. Built with iOS26 glassmorphism design.
---
Features
Dashboard: At-a-glance health status, active alerts (LDL high, premium due), BMI, BP, key metrics with year-on-year deltas
Health Records: Year-by-year health data (CBC, Lipids, Kidney, Liver, Imaging) with normal range indicators — pre-loaded with your Rutchapon & Kanokpai data
Trends: Multi-year charts (cholesterol, weight, blood sugar, kidney), AI-generated health insights per profile
Compare: Side-by-side comparison across profiles with family insights
Insurance: Full policy management (health + life), coverage details, premium due dates, payment tracking
Photo Upload: AI-powered extraction of health data from scanned reports (uses Anthropic API)
Settings / Admin: Profile management, user access control, export
---
Tech Stack
Layer	Tech
Frontend	Static HTML/CSS/JS — deploy to GitHub Pages
Database	Supabase PostgreSQL
Auth	Supabase Auth with Row Level Security
Storage	Supabase Storage (report photos)
Charts	Chart.js 4
---
Quick Start
1. Deploy Frontend to GitHub Pages
Create a GitHub repo (e.g. `health-tracker`)
Upload `index.html` to the repo root
Go to Settings → Pages → Deploy from main branch
Your app is live at `https://yourusername.github.io/health-tracker/`
2. Set Up Supabase
Create a project at supabase.com
Open SQL Editor and paste `supabase_schema.sql`
Run the SQL to create all tables, RLS policies, and triggers
Go to Project Settings → API and copy:
Project URL (e.g. `https://xxxx.supabase.co`)
anon/public key
3. Connect Frontend to Supabase
Open `index.html` and update lines 3-4:
```javascript
const SUPABASE_URL = 'https://YOUR-PROJECT.supabase.co';
const SUPABASE_KEY = 'your-anon-public-key';
```
4. Create First Account
Open the app, click Register
Log in with your email/password
Supabase automatically creates your user profile
Go to Settings → Profiles to add Rutchapon & Kanokpai profiles
---
User Access Model
Role	Access
Admin (you)	Full access to all profiles
Member (wife)	Own profile only
Future (kids)	View-only to parents' profiles
How to grant access to your wife:
She registers with her email
You (admin) go to Settings → Access Control
Click "Invite User" → enter her email → select profile → set access level
This is enforced at DB level via Supabase Row Level Security (RLS).
---
Enable AI Photo Upload (Optional)
For automatic health data extraction from photos:
Get an Anthropic API key at console.anthropic.com
Create a Supabase Edge Function to proxy the API call (keeps key server-side):
```bash
# In Supabase CLI
supabase functions new extract-health-data
```
Edge function code (`extract-health-data/index.ts`):
```typescript
import Anthropic from 'npm:@anthropic-ai/sdk';

const client = new Anthropic({ apiKey: Deno.env.get('ANTHROPIC_API_KEY') });

Deno.serve(async (req) => {
  const { imageBase64, mimeType } = await req.json();
  
  const response = await client.messages.create({
    model: 'claude-opus-4-6',
    max_tokens: 2000,
    messages: [{
      role: 'user',
      content: [{
        type: 'image',
        source: { type: 'base64', media_type: mimeType, data: imageBase64 }
      }, {
        type: 'text',
        text: `Extract all health indicators from this medical report. Return JSON with these keys: 
               check_date, hospital, weight_kg, height_cm, blood_pressure, pulse_rate, hemoglobin, 
               hematocrit, wbc, cholesterol, ldl, hdl, triglyceride, blood_sugar, hba1c, creatinine, 
               egfr, sgot, sgpt, alp, uric_acid, afp, cea, doctor_notes. Use null for missing values.`
      }]
    }]
  });
  
  return new Response(response.content[0].text, {
    headers: { 'Content-Type': 'application/json' }
  });
});
```
Set env var: `supabase secrets set ANTHROPIC_API_KEY=your-key`
Deploy: `supabase functions deploy extract-health-data`
---
Data Architecture
```
user_accounts (Supabase auth users)
    │
    ├── health_profiles (Rutchapon, Kanokpai, kids...)
    │       ├── health_records (one per year)
    │       ├── doctor_visits (surgeries, screenings...)
    │       └── profile_access (who can see this profile)
    │
    └── insurance_policies
            └── premium_payments (per year)
```
---
Pre-loaded Data (Demo Mode)
The app launches in demo mode with your actual health data from the Excel file:
Rutchapon (2021–2025):
LDL trending upward: 161 → 138 → 135 → 162 → 171 ⚠️
Mild fatty liver (2024)
GB polyp 0.3cm (stable)
Appendectomy (May 2021), Hemorrhoidectomy (Jun 2021)
Kanokpai (2017–2025):
LDL improved in 2025 (144→130) ✅
Kidney stones (bilateral, monitoring)
H. pylori treated (2022)
EST treadmill test: excellent (2024)
Shingles vaccine (2025)
---
Future Enhancements
[ ] Apple Health integration (via HealthKit bridge app)
[ ] Push notifications for premium due / checkup reminders
[ ] PDF report generation
[ ] Medication tracking
[ ] Bone density tracking
[ ] Family share via QR code (for ER situations)
---
Security Notes
Never store national ID or passport numbers in plain text — add AES encryption via Supabase Vault before production use
RLS policies ensure zero cross-profile data leakage
The `anon` key is safe to include in frontend — RLS enforces all access rules
API key for AI extraction must stay in Edge Function (never in frontend)
