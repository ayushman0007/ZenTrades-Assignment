# Clara AI — Synthetic Test Dataset
## Pipeline Validation Dataset v1.0

This dataset provides **5 demo call transcripts** (Pipeline A inputs) and **5 onboarding call transcripts** (Pipeline B inputs) for testing the Clara AI automation pipeline end-to-end.

---

## Dataset Structure

```
clara_ai_dataset/
├── transcripts/
│   ├── demo/
│   │   ├── demo_001_arctic_hvac.txt          → ACC-001
│   │   ├── demo_002_riverstone_plumbing.txt  → ACC-002
│   │   ├── demo_003_fireguard.txt            → ACC-003
│   │   ├── demo_004_volt_masters.txt         → ACC-004
│   │   └── demo_005_shieldpest.txt           → ACC-005
│   └── onboarding/
│       ├── onboarding_001_arctic_hvac.txt    → ACC-001
│       ├── onboarding_002_riverstone_plumbing.txt → ACC-002
│       ├── onboarding_003_fireguard.txt      → ACC-003
│       ├── onboarding_004_volt_masters.txt   → ACC-004
│       └── onboarding_005_shieldpest.txt     → ACC-005
└── expected_outputs/
    └── accounts/
        ├── ACC-001_memo_v1.json              → Full reference v1 memo
        ├── ACC-001_memo_v2.json              → Full reference v2 memo
        ├── ACC-001_changelog.md              → Reference changelog
        ├── ACC-001_agent_spec_v1.json        → Full reference agent spec
        └── all_accounts_v1_v2_summary.json  → v1→v2 change summary for ACC-002–005
```

---

## Account Overview

| Account ID | Company | Industry | Key Complexity |
|---|---|---|---|
| ACC-001 | Arctic Comfort HVAC Solutions | HVAC | Rotating on-call, zone-based seasonal logic |
| ACC-002 | Riverstone Plumbing & Drain | Plumbing | Spanish language handling, multi-trigger emergencies |
| ACC-003 | FireGuard Protection Services | Fire Protection | Zone-based, AHJ legal constraints, ServiceTrade restriction |
| ACC-004 | Volt Masters Electrical | Electrical | Safety language nuance, VIP routing, DBA change |
| ACC-005 | ShieldPest Control Group | Pest Control | Zone-dispatch by address, platform migration, weekend shutdown |

---

## What Your Pipeline Should Extract from Each Demo Transcript

Your LLM extraction step should produce a JSON with these fields per account:

- `account_id` — assigned by your pipeline (use ACC-00X format)
- `company_name`
- `business_hours` — days, start/end, timezone, any day-specific variation
- `office_address` — full structured address
- `services_supported` — list
- `emergency_definition` — list of triggers
- `emergency_routing_rules` — primary, secondary, fallback with phone numbers and timeouts
- `non_emergency_routing_rules` — what to say, what to collect
- `call_transfer_rules` — where to transfer, timeout rings, voicemail behavior
- `integration_constraints` — software restrictions (never touch X)
- `forbidden_phrases` — things agent must never say
- `after_hours_flow_summary`
- `office_hours_flow_summary`
- `greeting`
- `questions_or_unknowns` — only truly missing info
- `notes`

---

## What Your Pipeline Should Extract from Each Onboarding Transcript

Your onboarding extraction step should detect and apply a **patch** to the v1 memo, producing:

- `v2` memo JSON with updated fields
- `changes.md` or `changes.json` changelog documenting what changed and why

### Summary of Changes by Account

| Account | Key v1→v2 Changes |
|---|---|
| ACC-001 | Primary on-call replaced; 2 new services; 1 new emergency type; VIP routing rule added; website in after-hours close |
| ACC-002 | Saturday hours added; emergency line number changed; gas smell emergency + evacuation instruction added; 2 new services; greeting updated |
| ACC-003 | Secondary on-call number updated; AHJ violation added as emergency with direct routing; kitchen hood suppression added; Friday hours extended; guarantee banned; greeting updated |
| ACC-004 | Emergency routing order swapped; Saturday hours added; VIP Greenfield Properties routing; solar + battery services; DBA name shortened; Aaron's title updated |
| ACC-005 | Weekend hours fully dropped; zone structure rebuilt + new tech added; hospital/elder care rodent emergency added; wildlife removal service; platform migrated to ServiceTitan; greeting updated |

---

## Pipeline Validation Checklist

### Pipeline A — Demo → v1 (run per demo transcript)

- [ ] Extracts correct `company_name`
- [ ] Extracts correct `business_hours` including timezone and any day variations
- [ ] Extracts correct `office_address`
- [ ] Lists all `services_supported`
- [ ] Lists all `emergency_definition` triggers (count matches)
- [ ] Emergency routing has correct phone numbers and timeouts
- [ ] `integration_constraints` captures the specific software name
- [ ] `forbidden_phrases` are captured correctly
- [ ] `greeting` matches what client specified
- [ ] No hallucinated facts — `questions_or_unknowns` flags only truly missing info
- [ ] Retell agent spec generated with correct `system_prompt` flow
- [ ] Agent spec includes business hours flow AND after-hours flow AND emergency flow
- [ ] Agent spec never mentions function calls, software, or internal tools to caller
- [ ] Artifact stored in repo/DB
- [ ] Task tracker item created

### Pipeline B — Onboarding → v2 (run per onboarding transcript, linked to account_id)

- [ ] Correctly identifies account_id to patch
- [ ] Applies all changes (count: ACC-001=5, ACC-002=5, ACC-003=6, ACC-004=6, ACC-005=7)
- [ ] Does NOT modify unchanged fields
- [ ] Generates changelog with old value, new value, and reason for each change
- [ ] Generates v2 agent spec with updated system_prompt
- [ ] Stores v2 alongside v1 in versioned path
- [ ] Idempotent: running onboarding transcript twice produces the same v2 (no duplicate patches)

---

## Edge Cases to Watch For

### ACC-001
- On-call phone numbers must be updated (Mike Salazar OUT, Chris Vega IN) — test that old number does not appear in v2

### ACC-002
- Gas smell emergency requires a specific agent instruction (evacuate + call utility) — not just "route to tech"
- Saturday hours: verify correct hours (9 AM–1 PM), not copied from demo call

### ACC-003
- AHJ violation emergency bypasses dispatch and goes directly to Rick — test routing logic handles this branch
- `guarantee`/`guaranteed` is a HARD forbidden word — test your prompt generator never outputs it
- Friday hours change is `effective 2025-01-01` — test your pipeline stores effective date correctly

### ACC-004
- Emergency routing ORDER is swapped (Aaron first, then Linda) — old order was Linda first, Aaron second
- DBA name change: "Volt Masters Electrical" → "Volt Masters" — must appear in greeting and agent_name
- Greenfield Properties VIP routing: bypass standard queue, go direct to Linda

### ACC-005
- Zone restructuring: verify old zones are fully replaced, not appended
- Platform migration: FieldRoutes → ServiceTitan — old integration constraint must be updated
- Sunday call handling: v1 accepted Sunday scheduling calls; v2 = fully closed on weekends

---

## Prompt Quality Evaluation Criteria

For each generated agent system_prompt, verify:

1. **Business hours greeting** — personalized, warm, no IVR tone
2. **Collects name and number** before any transfer
3. **Emergency detection** — correctly identifies all triggers from the account
4. **Emergency routing** — primary → secondary → fallback with correct phones
5. **Transfer fail protocol** — what agent says when transfer fails
6. **After-hours non-emergency** — correct hours stated, correct collection, no transfer
7. **"Anything else"** check before close
8. **Forbidden phrases** — none present in generated prompt
9. **No internal tool mentions** — agent does not say "I'll create a ticket in X"
10. **Address collection** — for emergencies, address is always collected (critical for dispatch)

---

## Notes on Zero-Cost Compliance

If you use the Anthropic Claude API for extraction:
- Use Claude Haiku for extraction (cheapest, typically stays within free credits for a test run of 10 files)
- Or use a local model (Ollama + Llama3, Mistral, etc.)
- Document which LLM is used and estimated token counts per run

If using Groq API: free tier supports Llama3 and Mixtral with generous rate limits — good choice.

---

*This dataset is synthetic and does not contain real customer data. Phone numbers are fictional (555 prefix). Company names and addresses are invented for testing purposes only.*
