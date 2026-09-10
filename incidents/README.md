# Engineering incidents

Vertex is a commercial system, so these are public-safe engineering incident
notes, not complete internal postmortems.

They document selected failures and near-failures through the parts that are
useful to an engineer outside the company:

- the invariant that failed
- the observable symptoms
- why the original mental model was incomplete
- the systemic correction
- the regression strategy
- the broader engineering lesson

Implementation details, production identifiers, customer data,
security-sensitive information, and proprietary architecture are intentionally
omitted. Examples that use times or values are illustrative unless explicitly
identified otherwise.

## Notes

- [Incident 001 — When Authentication Wasn't a Platform Invariant](001-session-contract-drift.md)
- [Incident 002 — When Deployment Provenance Could Not Prove Itself](002-deployment-provenance.md)
- [Incident 003 — When Future Data Counted as Fresh](003-temporal-freshness.md)
