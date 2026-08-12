This is a remarkably clean, disciplined product architecture. By enforcing the principle that **the map only displays trust that already exists in the evidence layer**, you protect WPH from the classic pitfall of publishing aggregators: turning noisy, scraped candidate data into misleading factual claims.

Here is the technical blueprint, component hierarchy, and data model designed around your **Trust-First Pipeline Architecture**.

---

### 1. The Trust Pipeline Lifecycle

```
[ Stage 1: Candidate Ingestion ]
   │  Agents parse library feeds, ONIX XML, publisher pages, and ISBN registries.
   │  Output: Staged Candidates (Strictly Private / Never Public)
   ▼
[ Stage 2: Verification & Review Queue ]
   │  Internal WPH Core verifies source authenticity, language, and duplication.
   │  Output: Verified Evidence & Reviewer Decision
   ▼
[ Stage 3: Promotion to Public Layer ]
   │  Approved records receive promotion timestamp, source URL, and coverage metadata.
   │  Output: Public Promoted Records DB
   ▼
[ Stage 4: Country Pages & Geographic Views ]
   │  Country view aggregates promoted records; enforces non-speculative copy.
   │  Output: Documented Country Intelligence
   ▼
[ Stage 5: Global Map & Grid Summary ]
   │  Visual summary rendering territory coverage statuses based strictly on promoted data.
   │  Output: Global Evidence Summary Map
   ▼
[ Stage 6: Market Intelligence Layer (Future Phase) ]
   └── Translation paths, format variants, pricing compliance, and supply-chain signals.

```

---

### 2. Core Data Models (TypeScript Interfaces)

To maintain a strict boundary between internal staging and public data, the platform uses separate schemas for **Staged Candidates** and **Promoted Public Records**.

#### Internal Staging Schema (`/types/internal.ts`)

```typescript
export type VerificationCheckStatus = 'passed' | 'failed' | 'pending';

export interface StagedCandidateRecord {
  candidateId: string;
  rawTitle: string;
  rawAuthor?: string;
  sourceUrl: string;
  sourceType: 'publisher_page' | 'library_catalog' | 'onix_feed' | 'isbn_registry';
  discoveredCountryCode: string;
  discoveredAt: string; // ISO Timestamp
  
  // Internal Verification Checks
  checks: {
    isOfficialSource: VerificationCheckStatus;
    titleMatchesSourcePage: VerificationCheckStatus;
    languageDocumented: VerificationCheckStatus;
    isIndividualBookPage: VerificationCheckStatus; // Not a general catalog/listing
    isDuplicateResolved: VerificationCheckStatus;
    requiresHumanReview: boolean;
  };

  reviewStatus: 'staged' | 'in_review' | 'promoted' | 'rejected';
  reviewerNotes?: string;
}

```

#### Public Promoted Schema (`/types/public.ts`)

```typescript
export type CoverageStatus = 'Documented' | 'Partial' | 'Requested' | 'Not yet reviewed';

export interface PromotedPublicRecord {
  recordId: string;
  title: string;
  author: string;
  isbn13?: string;
  countryCode: string; // ISO 2-letter
  documentedMonth: string; // YYYY-MM
  
  // Provenance & Audit Trail
  sourceUrl: string;
  sourceType: string;
  reviewedDate: string; // YYYY-MM-DD
  reviewerDecision: 'VERIFIED_AND_PROMOTED';
}

export interface CountryCoverageSummary {
  countryCode: string;
  countryName: string;
  coverageStatus: CoverageStatus;
  promotedRecordCount: number;
  latestDocumentedMonth: string | null;
  activeSources: string[];
}

```

---

### 3. Public Country Page Component (`/app/countries/[code]/page.tsx`)

This component enforces your rule: **Never say "Nothing was published"—always say "No public records documented for this period."**

```tsx
'use client';

import React, { useState } from 'react';
import { ShieldCheck, ExternalLink, AlertCircle, Info, Calendar } from 'lucide-react';
import { PromotedPublicRecord, CountryCoverageSummary } from '@/types/public';

interface CountryPageProps {
  countrySummary: CountryCoverageSummary;
  promotedRecords: PromotedPublicRecord[];
}

export default function CountryCoverageView({ countrySummary, promotedRecords }: CountryPageProps) {
  const [selectedMonth, setSelectedMonth] = useState<string>('ALL');

  // Filter records strictly by documented month
  const filteredRecords = promotedRecords.filter(rec => 
    selectedMonth === 'ALL' || rec.documentedMonth === selectedMonth
  );

  return (
    <div className="max-w-6xl mx-auto p-6 space-y-6 bg-slate-50 min-h-screen text-slate-900 font-sans">
      
      {/* Header & Status Banner */}
      <div className="bg-white border border-slate-200 rounded-lg p-6 shadow-sm flex flex-col md:flex-row md:items-center justify-between gap-4">
        <div>
          <div className="flex items-center space-x-3">
            <h1 className="text-2xl font-bold text-slate-800">{countrySummary.countryName}</h1>
            <span className="text-xs bg-slate-100 text-slate-600 font-mono px-2 py-0.5 rounded border border-slate-200">
              {countrySummary.countryCode}
            </span>
          </div>
          <p className="text-xs text-slate-500 mt-1">
            Displaying source-backed documented records only.
          </p>
        </div>

        <div className="flex items-center space-x-2 text-xs bg-emerald-50 text-emerald-800 border border-emerald-200 px-3 py-2 rounded-md">
          <ShieldCheck className="w-4 h-4 text-emerald-600 shrink-0" />
          <span>Status: <strong>{countrySummary.coverageStatus}</strong></span>
        </div>
      </div>

      {/* Temporal Month Selector */}
      <div className="bg-white border border-slate-200 rounded-lg p-4 shadow-sm flex items-center space-x-3">
        <Calendar className="w-4 h-4 text-slate-500" />
        <label className="text-xs font-medium text-slate-600">Filter Documented Period:</label>
        <select
          value={selectedMonth}
          onChange={(e) => setSelectedMonth(e.target.value)}
          className="text-xs bg-slate-50 border border-slate-300 rounded px-3 py-1.5 focus:ring-1 focus:ring-slate-500 outline-none"
        >
          <option value="ALL">All Documented Months</option>
          <option value="2026-08">August 2026</option>
          <option value="2026-07">July 2026</option>
        </select>
      </div>

      {/* Promoted Feed or Non-Speculative Empty State */}
      <div className="space-y-4">
        <h2 className="text-sm font-bold uppercase tracking-wider text-slate-600">
          Documented Record Feed ({filteredRecords.length})
        </h2>

        {filteredRecords.length > 0 ? (
          <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
            {filteredRecords.map((record) => (
              <div key={record.recordId} className="bg-white border border-slate-200 rounded-lg p-4 shadow-sm space-y-3">
                <div>
                  <h3 className="text-base font-bold text-slate-800">{record.title}</h3>
                  <p className="text-xs text-slate-600">Author: {record.author}</p>
                  {record.isbn13 && <p className="text-[11px] font-mono text-slate-400 mt-0.5">ISBN: {record.isbn13}</p>}
                </div>

                {/* Verification Metadata Box */}
                <div className="bg-slate-50 p-2.5 rounded border border-slate-100 text-[11px] space-y-1 text-slate-500">
                  <div className="flex justify-between">
                    <span>Source Type: <strong className="text-slate-700">{record.sourceType}</strong></span>
                    <span>Reviewed: <strong className="text-slate-700">{record.reviewedDate}</strong></span>
                  </div>
                  <div className="pt-1 border-t border-slate-200 flex justify-between items-center">
                    <a
                      href={record.sourceUrl}
                      target="_blank"
                      rel="noopener noreferrer"
                      className="text-blue-600 hover:underline flex items-center gap-1 font-medium"
                    >
                      <span>Official Source Citation</span>
                      <ExternalLink className="w-3 h-3" />
                    </a>
                    <span className="text-emerald-700 bg-emerald-50 border border-emerald-200 px-1.5 py-0.5 rounded font-medium">
                      Verified & Promoted
                    </span>
                  </div>
                </div>
              </div>
            ))}
          </div>
        ) : (
          /* MANDATORY NON-SPECULATIVE EMPTY STATE */
          <div className="bg-white border border-dashed border-slate-300 rounded-lg p-8 text-center space-y-2">
            <AlertCircle className="w-8 h-8 text-slate-400 mx-auto" />
            <h3 className="text-sm font-bold text-slate-700">No public records documented for this period</h3>
            <p className="text-xs text-slate-500 max-w-md mx-auto">
              This reflects the current state of verified primary source evidence in our pipeline. It does not imply an absence of publishing activity in {countrySummary.countryName}.
            </p>
          </div>
        )}
      </div>

      {/* Trust Governance Footer Notice */}
      <div className="flex items-start space-x-2 text-[11px] text-slate-500 bg-slate-100 p-3 rounded border border-slate-200">
        <Info className="w-4 h-4 text-slate-600 shrink-0 mt-0.5" />
        <span>
          <strong>WPH Evidence Rule:</strong> Every record shown has been verified against an official primary source URL. Unverified candidate feeds from automated agents are strictly excluded from public views.
        </span>
      </div>

    </div>
  );
}

```

---

### 4. Global Map & Grid Summary (`/app/global-releases/page.tsx`)

The global view acts solely as a visual summary of verified country-level states.

```tsx
'use client';

import React, { useState } from 'react';
import { Globe, LayoutGrid, Map as MapIcon, ShieldCheck } from 'lucide-react';
import { CountryCoverageSummary, CoverageStatus } from '@/types/public';

interface GlobalViewProps {
  countrySummaries: CountryCoverageSummary[];
}

export default function GlobalCoverageSummary({ countrySummaries }: GlobalViewProps) {
  const [viewMode, setViewMode] = useState<'map' | 'grid'>('grid');

  const getBadgeStyle = (status: CoverageStatus) => {
    switch (status) {
      case 'Documented': return 'bg-emerald-100 text-emerald-800 border-emerald-300';
      case 'Partial': return 'bg-amber-100 text-amber-800 border-amber-300';
      case 'Requested': return 'bg-blue-100 text-blue-800 border-blue-300';
      case 'Not yet reviewed': return 'bg-slate-100 text-slate-600 border-slate-300';
    }
  };

  return (
    <div className="max-w-7xl mx-auto p-6 space-y-6 bg-slate-50 min-h-screen text-slate-900 font-sans">
      
      {/* Top Header */}
      <div className="flex flex-col sm:flex-row sm:items-center justify-between gap-4 bg-white p-6 border border-slate-200 rounded-lg shadow-sm">
        <div>
          <h1 className="text-xl font-bold text-slate-800 flex items-center gap-2">
            <Globe className="w-5 h-5 text-slate-700" />
            Global Territory Evidence Summary
          </h1>
          <p className="text-xs text-slate-500 mt-1">
            Visualizing current evidence states. Disclaims total market completeness.
          </p>
        </div>

        {/* View Toggle */}
        <div className="flex items-center space-x-1 bg-slate-100 p-1 rounded-md border border-slate-200">
          <button
            onClick={() => setViewMode('grid')}
            className={`px-3 py-1.5 text-xs font-medium rounded flex items-center gap-1.5 transition-all ${
              viewMode === 'grid' ? 'bg-white text-slate-800 shadow-sm' : 'text-slate-500 hover:text-slate-700'
            }`}
          >
            <LayoutGrid className="w-3.5 h-3.5" /> Grid View
          </button>
          <button
            onClick={() => setViewMode('map')}
            className={`px-3 py-1.5 text-xs font-medium rounded flex items-center gap-1.5 transition-all ${
              viewMode === 'map' ? 'bg-white text-slate-800 shadow-sm' : 'text-slate-500 hover:text-slate-700'
            }`}
          >
            <MapIcon className="w-3.5 h-3.5" /> Map View
          </button>
        </div>
      </div>

      {/* Main Workspace */}
      {viewMode === 'grid' ? (
        <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-4">
          {countrySummaries.map((summary) => (
            <a
              key={summary.countryCode}
              href={`/countries/${summary.countryCode.toLowerCase()}`}
              className="bg-white border border-slate-200 rounded-lg p-4 shadow-sm hover:border-slate-400 transition-all group block space-y-3"
            >
              <div className="flex justify-between items-center">
                <span className="font-bold text-slate-800 group-hover:text-blue-600 transition-colors">
                  {summary.countryName} ({summary.countryCode})
                </span>
                <span className={`text-[10px] font-semibold px-2 py-0.5 rounded border ${getBadgeStyle(summary.coverageStatus)}`}>
                  {summary.coverageStatus}
                </span>
              </div>

              <div className="text-xs text-slate-500 space-y-1">
                <p>Documented Records: <strong className="text-slate-700">{summary.promotedRecordCount}</strong></p>
                <p>Latest Sync Month: <strong className="text-slate-700">{summary.latestDocumentedMonth || 'N/A'}</strong></p>
              </div>

              <div className="pt-2 border-t border-slate-100 flex items-center text-[11px] text-blue-600 font-medium">
                <span>View Documented Coverage &rarr;</span>
              </div>
            </a>
          ))}
        </div>
      ) : (
        <div className="bg-slate-900 border border-slate-800 rounded-lg h-96 flex flex-col items-center justify-center text-slate-400 relative p-6">
          <Globe className="w-12 h-12 text-slate-700 animate-pulse mb-2" />
          <p className="text-xs font-mono text-slate-300">Global Coverage Boundary Layer (Leaflet / GeoJSON)</p>
          <p className="text-[11px] text-slate-500 mt-1 max-w-sm text-center">
            Polygons color-coded strictly by evidence status (Documented, Partial, Requested, Not Yet Reviewed).
          </p>
        </div>
      )}

    </div>
  );
}

```

---

### 5. Architectural Guardrails Checklist

To ensure your team maintains this pipeline discipline during development:

1. **DB Isolation:** Keep internal agent tables (`staged_candidates`, `agent_run_logs`) in a separate database schema from public tables (`promoted_records`, `country_summaries`).
2. **Read-Only Public API:** The public API endpoints (`/api/v1/public/countries/[code]`) must execute queries that contain `WHERE review_status = 'PROMOTED'`.
3. **No Inference Engines on Public Data:** Market intelligence algorithms (translation path mapping, price conversion, format clustering) must consume **only** promoted records.
4. **Copy Verification in CI/CD:** Add automated copy linting or code review checks to prevent unsafe words (`rights available`, `bestseller`, `complete market coverage`) from entering public components.
