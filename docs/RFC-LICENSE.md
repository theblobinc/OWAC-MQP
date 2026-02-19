## Licensing for vendored RFC documents in this repository

This repository may include **verbatim (unmodified) copies** of certain IETF RFC documents (e.g., under `docs/rfcs/`) strictly for **reference / offline documentation** purposes.

These RFC files are **NOT** covered by this repository’s MIT license. They remain under the licensing terms stated by the IETF Trust and/or the copyright notice contained in each RFC.

---

## 1) What RFCs are covered here

The RFCs commonly vendored for OWAC-MQP documentation are:

- RFC 1661 — Point-to-Point Protocol (PPP)
- RFC 1662 — PPP in HDLC-like Framing
- RFC 2119 — Key words for use in RFCs to Indicate Requirement Levels
- RFC 8174 — Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words
- RFC 8610 — Concise Data Definition Language (CDDL)
- RFC 8949 — Concise Binary Object Representation (CBOR)

(If you vendor additional RFCs, add them to this list.)

---

## 2) The governing license / terms for RFC text

### 2.1 Modern RFCs (typical current boilerplate)
Most newer RFCs include a “Copyright Notice” stating that the document is subject to:

- **BCP 78**, and
- the **IETF Trust’s “Legal Provisions Relating to IETF Documents”** (often linked as “license-info”).

In general, these terms allow you to:
- **copy and redistribute the RFC in full and without modification**,
- include **unmodified excerpts** with proper attribution and preservation of notices (especially when reproducing substantial portions),
- translate the RFC (subject to the Trust’s terms).

### 2.2 Older RFCs
Older RFCs may use earlier boilerplate and licensing language. If you vendor an older RFC (e.g., mid-1990s), treat the **copyright / license section inside that RFC** as authoritative.

---

## 3) No modification rule (important)

Unless you are operating within the IETF Standards Process, the IETF Trust terms generally **do not grant a license to publish modified versions** of RFC text.

**Practical rule for this repo:**  
✅ Vendor RFCs **as-is** (verbatim).  
❌ Don’t edit, annotate, “clean up”, or reformat RFC contents beyond what the canonical distribution already provides.

If you want commentary, put it in *separate* files you wrote (your own docs), not inside the RFC files.

---

## 4) Code Components inside RFCs (different license)

Some RFCs contain **“Code Components”** (snippets intended to be processed by a computer, or text explicitly marked as code).

The IETF Trust terms typically allow you to extract and use these Code Components under a **BSD 3-Clause style license** (often historically labeled “Simplified BSD” in some RFC boilerplate, but the intended license text is the **Revised BSD / BSD-3-Clause**).

**Summary of the BSD-3-Clause-style obligations (paraphrased):**
- Keep the copyright notice and license conditions with source distributions.
- Reproduce them in documentation/materials for binary distributions.
- Don’t use the IETF/ISOC/IETF Trust names (or contributors’ names) to endorse derived products without prior permission.
- Code is provided **as-is**, without warranty; liability is disclaimed.

If you extract RFC code into your project, add attribution like:
> “Derived from IETF RFC #### (see `docs/rfcs/`).”

---

## 5) Attribution & “definitive version” note

- Attribute the RFCs to the IETF / RFC Editor and cite the RFC number.
- The definitive versions are the ones published by/under the auspices of the IETF (RFC Editor).

---

## 6) Where to find the authoritative legal text (recommended links)

IETF Trust Legal Provisions (TLP):
- https://trustee.ietf.org/documents/trust-legal-provisions/
- https://trustee.ietf.org/documents/trust-legal-provisions/tlp-5/

RFC “license-info” landing page (referenced by many RFCs):
- https://trustee.ietf.org/license-info

RFC Editor info pages (canonical retrieval):
- https://www.rfc-editor.org/info/rfc1661
- https://www.rfc-editor.org/info/rfc1662
- https://www.rfc-editor.org/info/rfc2119
- https://www.rfc-editor.org/info/rfc8174
- https://www.rfc-editor.org/info/rfc8610
- https://www.rfc-editor.org/info/rfc8949

---

## 7) Repository licensing clarity

- **MIT License** applies to: code, original documentation, and other repository content created by this project’s authors **excluding** vendored RFCs and other third-party materials.
- Vendored RFC documents remain under the IETF Trust / RFC-specific terms described above.

---

## 8) Disclaimer

This file is provided for convenience and does not replace the license terms included in each RFC or the IETF Trust Legal Provisions.
