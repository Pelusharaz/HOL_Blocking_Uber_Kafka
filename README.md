# SRS + ADR: Consumer Proxy for Kafka Event Consumption

A Software Requirements Specification (SRS) template, written in Markdown and organised the way the
[Architecture Decision Record (ADR) project](https://github.com/architecture-decision-record/architecture-decision-record)
recommends, together with a filled-in example for our earlier case study:
**head-of-line (HOL) blocking in Uber's Kafka-based event-driven architecture**.

> **Evidence labels used everywhere in this repository**
> - **[Fact]** documented publicly (source cited).
> - **[Assumption]** a reasonable architectural assumption, not verified.
> - **[Proposal]** our own design or requirement; **not** claimed to be implemented by Uber.

## Repository layout

```
.
├── README.md
└── docs
    ├── adr
    │   ├── template.md                      # ADR template (Michael Nygard style)
    │   └── adopt-consumer-proxy.md          # ADR-0001: the decision behind the SRS
    └── srs
        ├── srs-template.md                  # blank, reusable SRS template
        └── define-consumer-proxy-requirements.md   # SRS filled in for this project
```

File names follow the ADR convention: lowercase, dashes, and a present-tense imperative verb phrase
(for example `adopt-consumer-proxy.md`), with the `.md` extension.

## How the SRS and the ADRs fit together

| Document | Answers | Changes when |
|---|---|---|
| **ADR** | *Why* did we choose this architecture, and what follows from it? | A decision is made, amended or superseded |
| **SRS** | *What* must the system do, and how well? | Requirements change |

Every requirement in the SRS links to the ADR that motivates it, and every ADR links back to the
requirements it affects. This is the traceability the ADR project calls "architecture knowledge management".

## How to use the template

1. Copy `docs/srs/srs-template.md` to a new file named with a verb phrase, e.g. `define-payment-service-requirements.md`.
2. Fill in the header (status, date, authors, related ADRs).
3. Write each requirement with a unique ID, a priority, a rationale and a verification method.
4. When a requirement depends on an architectural choice, write an ADR (`docs/adr/template.md`) and link both ways.
5. Date-stamp changes in the revision history instead of silently editing old text.

## Sources

- Architecture decision record repository: https://github.com/architecture-decision-record/architecture-decision-record
- ISO/IEC/IEEE 29148:2018, *Systems and software engineering: Life cycle processes: Requirements engineering* (structure of the SRS).
- Uber Engineering, *Introducing uForwarder: The Consumer Proxy for Kafka Async Queuing* (uber.com/blog).
- Apache Kafka documentation (consumer groups, offsets).
