---
name: dokki-artifact
description: Create or edit interactive JSX artifacts — React components, charts (Recharts), diagrams (Mermaid), animations (Framer Motion). Use for anything interactive or visual beyond what a document can express.
argument-hint: <description-or-id> [instruction]
allowed-tools: mcp__dokki__create_artifact mcp__dokki__artifact_read mcp__dokki__artifact_update mcp__dokki__artifact_patch
---

# Dokki Artifact — Create & Edit

## Is an artifact the right resource?

| Need | Resource |
|------|----------|
| Static prose, lists, simple tables | `dokki-document` |
| Structured rows/columns with sort/filter | `dokki-table` |
| Interactive widget, custom layout, animation | **artifact** |
| Chart / graph from data | **artifact** (Recharts) |
| Flowchart, sequence diagram, state diagram | **artifact** (Mermaid) |

---

## Runtime Environment

Sandboxed React runtime. Only these imports work:

```jsx
// React 18 — hooks
import { useState, useEffect, useMemo, useRef } from 'react';

// Animations
import { motion, AnimatePresence } from 'framer-motion';

// Icons (600+)
import { Check, X, ArrowRight, /* any Lucide icon */ } from 'lucide-react';

// Charts
import {
  LineChart, BarChart, AreaChart, PieChart, RadarChart,
  Line, Bar, Area, Pie, Radar,
  XAxis, YAxis, CartesianGrid, Tooltip, Legend, ResponsiveContainer
} from 'recharts';

// Diagrams — SPECIAL: import path differs
import { Mermaid } from 'mermaid';

// Styling — Tailwind classes only (no CSS files)
```

Nothing else will resolve. No filesystem, no network, no npm install.

---

## CREATE Mode

### Tool

```
create_artifact:
  name: string
  workspace_id: string
  parent_id?: string
  source: string    # Complete JSX module, must export default
```

### Required source structure

```jsx
import { useState } from 'react';
// other imports...

export default function MyComponent() {
  // state & logic
  return (
    <div className="...">
      {/* markup */}
    </div>
  );
}
```

### Template: Chart Dashboard

```jsx
import { BarChart, Bar, XAxis, YAxis, Tooltip, ResponsiveContainer, CartesianGrid } from 'recharts';

const data = [
  { month: 'Jan', revenue: 42000, expenses: 28000 },
  { month: 'Feb', revenue: 51000, expenses: 31000 },
  { month: 'Mar', revenue: 47000, expenses: 29000 },
];

export default function RevenueDashboard() {
  return (
    <div className="p-6 bg-white rounded-xl shadow-md">
      <h2 className="text-2xl font-bold mb-2">Q1 Revenue</h2>
      <p className="text-sm text-gray-500 mb-4">Monthly revenue vs. expenses</p>
      <ResponsiveContainer width="100%" height={320}>
        <BarChart data={data}>
          <CartesianGrid strokeDasharray="3 3" />
          <XAxis dataKey="month" />
          <YAxis />
          <Tooltip />
          <Bar dataKey="revenue" fill="#6366f1" />
          <Bar dataKey="expenses" fill="#ef4444" />
        </BarChart>
      </ResponsiveContainer>
    </div>
  );
}
```

### Template: Mermaid Diagram

```jsx
import { Mermaid } from 'mermaid';

export default function ArchFlow() {
  return (
    <div className="p-6">
      <Mermaid chart={`graph TD
        Client-->|HTTPS|CDN
        CDN-->|Origin|App[Next.js App]
        App-->|RPC|DB[(Supabase)]
        App-->|WebSocket|Hocuspocus
      `} />
    </div>
  );
}
```

### Template: Interactive Widget

```jsx
import { useState } from 'react';
import { motion } from 'framer-motion';
import { Check, ChevronRight } from 'lucide-react';

const STEPS = ['Plan', 'Build', 'Test', 'Ship'];

export default function Checklist() {
  const [done, setDone] = useState(new Set());
  const toggle = (i) => {
    const next = new Set(done);
    next.has(i) ? next.delete(i) : next.add(i);
    setDone(next);
  };

  return (
    <div className="p-6 max-w-md mx-auto">
      <h2 className="text-xl font-bold mb-4">Release Checklist</h2>
      <ul className="space-y-2">
        {STEPS.map((step, i) => (
          <motion.li
            key={i}
            whileHover={{ x: 4 }}
            onClick={() => toggle(i)}
            className={`flex items-center gap-3 p-3 rounded-lg cursor-pointer border ${
              done.has(i) ? 'bg-green-50 border-green-300' : 'bg-white border-gray-200'
            }`}
          >
            <div className={`w-6 h-6 rounded-full flex items-center justify-center ${
              done.has(i) ? 'bg-green-500 text-white' : 'bg-gray-100'
            }`}>
              {done.has(i) ? <Check size={14} /> : <ChevronRight size={14} />}
            </div>
            <span className={done.has(i) ? 'line-through text-gray-500' : ''}>{step}</span>
          </motion.li>
        ))}
      </ul>
    </div>
  );
}
```

---

## EDIT Mode

### Operation Decision Tree

```
How big is the change?
│
├── New component from scratch / near-total rewrite
│     └── artifact_update (full source)
│
└── Localized change (rename prop, adjust style, fix bug)
      └── artifact_patch (old_string → new_string)
```

### Always Read First

`artifact_read` returns the **current** source — which may differ from what you created earlier if a human edited it. Reading first avoids overwriting their changes.

`artifact_patch` uses 3-way merge for concurrent-safe edits, but it needs `old_string` to match **exactly** — including whitespace.

---

## Pitfalls

| Pitfall | Why it hurts | Fix |
|---------|-------------|-----|
| Importing from packages not whitelisted | Runtime error, blank artifact | Stick to the listed imports |
| Missing `export default` | Component won't render | Every artifact must `export default` |
| Using CSS files / styled-components | No CSS pipeline in sandbox | Use Tailwind utility classes only |
| Forgetting `ResponsiveContainer` on charts | Charts render at 0px | Wrap Recharts in `<ResponsiveContainer width="100%" height={N}>` |
| Mermaid imported from `mermaid-js` | Wrong path — fails | Use `import { Mermaid } from 'mermaid'` |
| `artifact_patch` with fuzzy match | Fails silently or wrong spot | `old_string` must be exact (incl. whitespace & indentation) |
| Giant single-file artifact (>500 lines) | Slow to render, hard to edit | Keep it focused; compose inline sub-components |
| External data fetching | No network in sandbox | Inline the data or fetch it via MCP and hardcode into source |

## Cross-Skill Follow-Ups

- Need the data first? → `dokki-table` to create/read the table, then inline the data in the artifact
- Publish the artifact in a docs site? → `dokki-publish`
- Place next to related docs? → `dokki-workspace` `move_resource`
