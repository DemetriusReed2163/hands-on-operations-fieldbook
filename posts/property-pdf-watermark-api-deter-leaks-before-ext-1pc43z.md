# Property PDF Watermark API: Deter Leaks Before External Sharing

TL;DR: Keep the watermark template in the property platform when its wording, placement, and recipient fields are part of sharing policy. Let a rendering component apply it, but do not let that component silently define it. This split gives a small SaaS team clear reviews, rollbacks, and audit evidence. A watermark can deter casual redistribution and identify an intended recipient; it cannot stop screenshots, photography, or deliberate PDF editing.

| Template owner | Pick this when | Main trade-off | Evidence to retain |
|---|---|---|---|
| Application repository | Policy changes need code review | Wording changes follow the release path | Template version, commit, render test |
| Operations configuration | Approved staff need changes without a deploy | Validation and approval become application features | Author, approval, activation time |
| Rendering service | It is already the controlled template system | Moving renderers may mean rebuilding policy | Export, revision, request result |
| Split ownership | The app owns meaning; an adapter owns PDF mechanics | The interface must exclude policy-changing defaults | Policy version, adapter version, output digest |

The question is not merely where bytes change. It is who can later answer: which approved message appeared on the lease packet sent to this applicant? Treat the template as policy-bearing data.

## Who should own the PDF watermark API before external sharing?

Choose repository ownership when engineers already review external-sharing rules. A typed template can require a recipient label, property label, purpose, and share identifier. The same change can update render fixtures. Rollback is ordinary source control, though an operations-only wording correction must wait for delivery.

Operations-owned configuration fits when authorized staff genuinely need independent control. This should not be a free-form text box. It needs a schema, preview, approval state, immutable revisions, and an activation record. Otherwise, ownership is hidden in a database row where a typo can become policy.

Service ownership fits when the renderer is the formal system of record for layouts. Verify that templates can be exported, revisions pinned, and requests tied to an exact revision. A request that means "use current" cannot reproduce an old share after the template changes.

For many small teams, split ownership is the cleanest boundary. The application owns semantic fields. An adapter owns font embedding, page geometry, opacity, rotation, and serialization. **The team accountable for sharing policy should own the template's meaning.**

Picture the path: an authorized user selects a lease packet; the application reserves a share record; policy resolves the watermark; a renderer produces new bytes; storage accepts them; then the system publishes external access. Keep that order. Never publish the original and hope to replace it afterward.

Create the share ID before rendering. Put it in the visible mark and structured logs, but keep recipient email addresses out of logs. An internal recipient reference is enough there.

```ts
export type WatermarkPolicy = {
  templateVersion: string;
  recipientLabel: string;
  propertyLabel: string;
  purpose: "applicant-review" | "owner-review" | "vendor-review";
  shareId: string;
  issuedAt: string;
};

export type RenderReceipt = {
  output: Uint8Array;
  pageCount: number;
  outputSha256: string;
  adapterVersion: string;
};

export interface PdfWatermarkAdapter {
  apply(source: Uint8Array, policy: WatermarkPolicy): Promise<RenderReceipt>;
}
```

This contract exposes no vendor template ID and makes no promise that copying is impossible. The digest identifies produced bytes. The adapter version identifies rendering behavior.

On failure, leave the share unpublished. Retry through an idempotent job keyed by share ID. Preserve the source as immutable input and store output under a distinct key. A partial output must never advance share state.

Short path. Strong boundary.

## Implement one policy-controlled path

This orchestration keeps policy outside the adapter. The strings are example property-workflow choices, not universal rules.

```ts
type ShareRequest = {
  documentId: string;
  recipientLabel: string;
  propertyLabel: string;
  purpose: WatermarkPolicy["purpose"];
};

type Dependencies = {
  loadOriginal(id: string): Promise<Uint8Array>;
  reserveShare(request: ShareRequest): Promise<{ shareId: string; issuedAt: string }>;
  render: PdfWatermarkAdapter;
  storeRendered(id: string, bytes: Uint8Array): Promise<void>;
  publish(id: string, receipt: Omit<RenderReceipt, "output">): Promise<string>;
};

export async function createExternalShare(request: ShareRequest, deps: Dependencies) {
  const share = await deps.reserveShare(request);
  const source = await deps.loadOriginal(request.documentId);
  const receipt = await deps.render.apply(source, {
    templateVersion: "property-external-v3",
    recipientLabel: request.recipientLabel,
    propertyLabel: request.propertyLabel,
    purpose: request.purpose,
    shareId: share.shareId,
    issuedAt: share.issuedAt,
  });

  await deps.storeRendered(share.shareId, receipt.output);
  const url = await deps.publish(share.shareId, {
    pageCount: receipt.pageCount,
    outputSha256: receipt.outputSha256,
    adapterVersion: receipt.adapterVersion,
  });
  return { shareId: share.shareId, url };
}
```

Production code must authorize the request, validate resolved fields, cap inputs from capacity tests, and apply timeouts at process or network boundaries. Test output as a document, not just a fulfilled promise. Keep fixtures for one-page notices, multi-page leases, rotated pages, mixed sizes, and forms. Parse results with an independent PDF reader. Render selected pages to images and compare stable regions with tolerances; byte comparison can be too strict for serialized output.

Assert that the original remains unchanged and the link resolves only to the derived object. Check visible text, intended pages, clipping, and legibility over light and dark content. Repeated visible text can affect extraction or reading order depending on encoding, so inspect with the readers tenants, owners, and staff use.

## Operate it like a sharing control

Track render attempts, failures by stable class, end-to-end share latency, and unpublished shares in each state. Add page-count and input-byte buckets for context. Do not put names, addresses, document text, or signed links in metric labels.

One trace can join reserve, render, store, and publish with the internal share ID. Logs should name policy and adapter revisions. Alert on user impact, such as shares failing to publish, rather than every isolated exception. Set thresholds from observed traffic and an agreed objective; no universal percentage fits every workload.

Deploy a revision against fixed fixtures, enable it for a bounded internal cohort, inspect pages, then expand. Keep the previous revision selectable during that check. Retention needs separate decisions for audit metadata and rendered documents; neither should come from a storage default.

A visible watermark is a deterrent and attribution cue. It is not access control, encryption, signature validation, or a guarantee that content cannot leave a screen. Keep authorization, expiring access, revocation, and audit records around it.

PDF is standardized by ISO 32000-2 and can contain rotated pages, annotations, forms, embedded fonts, transparency, signatures, and other structures that complicate modification. Define accepted inputs, test those classes, and reject unsupported files before issuing a link. Modification can affect existing digital signatures, so signed documents need an explicit preservation or re-signing policy.

Template ownership does not solve every leak. It makes one vital boundary reviewable: the exact message attached to an external property document, the substituted data, and the revision responsible for the result.

Consider a concrete fixture: a two-page lease summary whose first page is portrait and second page is rotated landscape. The policy says both pages must show the recipient label, property label, share ID, and issue time. A test that searches extracted text can pass even if the second mark is clipped outside the visible page after rotation. A screenshot comparison catches the placement error, while the parsed-text assertion confirms the expected fields survived serialization. Keep both checks because they answer different questions. This is an intentional trade-off: image fixtures require review when rendering legitimately changes, but text-only tests miss the visual failure that matters to an external recipient. Record the accepted fixture with the template revision so the next change has a visible before-and-after.

It fails visibly.

## Sources and References

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
