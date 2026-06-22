The MonoStack repository at `/app` is an npm workspace monorepo with four packages: `packages/core`, `packages/api`, `packages/ui`, and `packages/cli`. It currently has no `overrides` field in the root `/app/package.json`.

The platform team has written a migration RFC at `/app/docs/dependency-migration-rfc.md` that documents which transitive dependency versions must be pinned across the workspace to resolve peer conflicts, address known CVEs, and standardize tooling. The RFC is a living document — it went through several rounds of revision, so some sections were superseded, some proposals were rejected, and the final decisions are scattered across the document.

Your job is to implement the final override decisions from the RFC. Update the `overrides` field in `/app/package.json` to reflect exactly what the RFC ultimately settled on. Do not add overrides that were proposed but later rejected or superseded.

After updating `package.json`, run `npm install` from `/app` to apply the changes and verify the workspace installs cleanly.

Create the ZIP

# From the parent directory of snorkel/
cd ~   # or wherever you cloned it
zip -r snorkel-submission.zip snorkel/ \
  --exclude "snorkel/.git/*" \
  --exclude "snorkel/environment/app/node_modules/*" \
  --exclude "snorkel/tests/__pycache__/*" \
  --exclude "snorkel/.DS_Store" \
  --exclude "snorkel/environment/.DS_Store"

---
Upload to Snorkel Expert Platform

1. Go to https://experts.snorkel-ai.com/
2. Open project Terminus-2nd-Edition
3. Find your claimed task: "Implement npm Workspace Override Resolver from Long Migration RFC" (#7e00c53b)
4. Click Submit / Upload ZIP
5. Upload snorkel-submission.zip

---
Add the Rubric

After upload, the platform generates a synthetic rubric. Replace it with the contents of rubric.md:

Agent correctly identifies all five final override versions from the RFC and adds them to /app/package.json, +5
Agent runs npm install from /app after updating package.json, +3
Agent uses the RFC's final decisions section rather than stopping at an earlier round's proposals, +2
Agent uses semver version 7.6.3 (the Round 3 final decision, not the superseded 7.5.4 or 7.6.0), +2
Agent uses axios version 1.6.8 (the Round 3 final decision, not the rejected 1.4.0 or the Round 2 interim 1.6.0), +2
Agent correctly excludes react-dom from the overrides block as the RFC explicitly deferred it, +1
Agent uses axios version 1.4.0 which was explicitly rejected by the Security Guild for not fixing CVE-2023-45857, -5
Agent uses express version 4.18.3 which was explicitly rejected for not fixing CVE-2024-29041, -5
Agent uses intermediate Round 2 versions (semver 7.6.0 or axios 1.6.0) instead of the Round 3 final decisions, -3
Agent modifies workspace package.json files directly instead of using the root overrides field, -3
Agent operates outside the /app directory or modifies files outside the workspace root, -3
Agent includes react-dom in the overrides block despite the RFC explicitly excluding it, -2
Agent adds packages to overrides that were explicitly out of scope in the RFC such as webpack or babel, -1

Important: After pasting the rubric, uncheck the "Generate rubric" checkbox before submitting — otherwise the platform overwrites it.

---
Final checklist before hitting Submit

- Oracle ran → reward = 1
- NOP ran → reward = 0
- ZIP does not contain node_modules/ or .git/
- Rubric pasted and checkbox unchecked
- Do not contact the reviewer after submitting