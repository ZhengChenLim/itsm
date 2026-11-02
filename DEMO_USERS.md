# IT Portal demo user accounts

Fictional demo accounts from `manage.py seed_demo`. Sign in with the **User ID** (or the email, where there is one) and the shared demo password: `DEFAULT_DEMO_PASSWORD` in `backend/core/management/commands/seed_demo.py` (or `PORTAL_DEMO_PASSWORD` if set). Never use these accounts outside a local demo.

| User ID | Name | Email | Role | Department | IT pool | Manager | Status |
|---|---|---|---|---|---|---|---|
| E1001 | Victor VP Technology | victor.vp.technology@example.local | VP | IT | - | - | Active |
| E1002 | Valerie CFO | valerie.cfo@example.local | VP | Finance | - | - | Active |
| E1003 | Vincent VP Operations | vincent.vp.operations@example.local | VP | Operations | - | - | Active |
| E1004 | Chloe Chief Executive | chloe.chief.executive@example.local | C-suite | - | - | - | Active |
| E1005 | Hana Director | hana.director@example.local | Head of IT | IT | - | Victor VP Technology | Active |
| E1006 | Marcus HQ Lead | marcus.hq.lead@example.local | IT manager | IT | ITHQ | Hana Director | Active |
| E1007 | Aisha App Lead | aisha.app.lead@example.local | IT manager | IT | ITAA | Hana Director | Active |
| E1008 | Ben BI Lead | ben.bi.lead@example.local | IT manager | IT | ITBA | Hana Director | Active |
| E1009 | Ivan Network | ivan.network@example.local | IT member | IT | ITHQ | Marcus HQ Lead | Active |
| E1010 | Grace Desktop | grace.desktop@example.local | IT member | IT | ITHQ | Marcus HQ Lead | Active |
| E1011 | Leo Junior | leo.junior@example.local | IT member | IT | ITHQ | Marcus HQ Lead | Active |
| E1012 | Nora SAP | nora.sap@example.local | IT member | IT | ITAA | Aisha App Lead | Active |
| E1013 | Omar POS | omar.pos@example.local | IT member | IT | ITAA | Aisha App Lead | Active |
| E1014 | Priya Apps | priya.apps@example.local | IT member | IT | ITAA | Aisha App Lead | Active |
| E1015 | Quinn Data | quinn.data@example.local | IT member | IT | ITBA | Ben BI Lead | Active |
| E1016 | Rita Report | rita.report@example.local | IT member | IT | ITBA | Ben BI Lead | Active |
| E1017 | Fiona Finance Head | fiona.finance.head@example.local | Finance | Finance | - | Valerie CFO | Active |
| E1018 | Farah Finance Exec | farah.finance.exec@example.local | Finance | Finance | - | Fiona Finance Head | Active |
| E1019 | Felix AP Officer | felix.ap.officer@example.local | Finance | Finance | - | Fiona Finance Head | Active |
| E1020 | Adam System Admin | adam.system.admin@example.local | System admin | IT | - | Hana Director | Active |
| E1021 | Oscar Ops Head | oscar.ops.head@example.local | Employee | Operations | - | Vincent VP Operations | Active |
| E1022 | Ella Finance | ella.finance@example.local | Employee | Finance | - | Fiona Finance Head | Active |
| E1023 | Sam Store Ops | sam.store.ops@example.local | Employee | Operations | - | Oscar Ops Head | Active |
| E1024 | Tina Store Ops | (none - sign in with user ID) | Employee | Operations | - | Oscar Ops Head | Active |
| E1025 | Iris Audit Head | iris.audit.head@example.local | Employee | Internal Audit | - | Chloe Chief Executive | Active |
| E1026 | Uma Auditor | (none - sign in with user ID) | Employee | Internal Audit | - | Iris Audit Head | Active |

## Department approvers

| Department | HOD | VP | C-suite (used when no VP) |
|---|---|---|---|
| Finance | Fiona Finance Head | Valerie CFO | - |
| IT | Hana Director | Victor VP Technology | - |
| Internal Audit | Iris Audit Head | - | Chloe Chief Executive |
| Operations | Oscar Ops Head | Vincent VP Operations | - |

Notes: the CFO is the VP of Finance (Valerie CFO). The Finance HOD (Fiona Finance Head) sets claim approval limits. Adam System Admin manages users, system settings and the audit log.
