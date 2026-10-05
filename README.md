---
license: cc-by-nc-sa-4.0
task_categories:
  - text-generation
  - question-answering
language:
  - en
tags:
  - cybersecurity
  - threat-intelligence
  - CTI
  - malware-analysis
  - incident-response
  - MITRE-ATT&CK
  - MITRE-ATLAS
  - instruction-tuning
  - llm-fine-tuning
  - detection-engineering
  - threat-hunting
  - vulnerability
  - CVE
  - security-operations
size_categories:
  - 10K<n<100K
configs:
  - config_name: default
    data_files:
      - split: train
        path: data/train.jsonl
      - split: validation
        path: data/eval.jsonl
  - config_name: blended
    data_files:
      - split: train
        path: data/train_blended.jsonl
  - config_name: multi_turn
    data_files:
      - split: train
        path: data/train_multiturn.jsonl
---

# Cyber Threat Intelligence Dataset for LLM Fine-Tuning

Instruction-tuning data for cyber threat intelligence tasks: explaining the exploitation risk of a CVE, profiling a threat actor from its ATT&CK techniques, turning a Sigma rule into alert-triage steps, mapping a campaign to the kill chain, writing detection logic for a technique, and similar work.

The splits are in `data/`.

## Grounding

Records are generated from public sources (MITRE ATT&CK, CISA KEV, CWE, OSV, abuse.ch, ransomware.live, Sigma and others), not from free-form model output. The technique IDs, CVEs, CWEs and indicators in each record are checked against the source, and records that fail the check are dropped.

## Contents

- 11,962 single-turn examples and 1,302 multi-turn conversations
- 42 categories, each with at least 155 examples
- 96.1% of instructions are distinct
- Train and eval share no entity: all tasks about the same CVE, group, technique, indicator or advisory are on the same side of the split
- One record per entity and task type, near-duplicates removed

## Files

| File | Records | Description |
|------|--------:|-------------|
| `data/train.jsonl` | 11,037 | Single-turn training split (`instruction` / `input` / `output`) |
| `data/eval.jsonl` | 925 | Evaluation split, stratified by category and separated from train by entity |
| `data/train_blended.jsonl` | 14,716 | The train split plus 25% general instructions, to limit forgetting outside security |
| `data/train_multiturn.jsonl` | 1,302 | Two-turn conversations in `messages` format. Both assistant turns are grounded; the follow-up covers a second angle on the same entity, for example a CVE analysis followed by a patching-priority question |

Loading from Hugging Face:

```python
from datasets import load_dataset

ds       = load_dataset("reloading0101/threat-intelligence-dataset")               # train + validation
blended  = load_dataset("reloading0101/threat-intelligence-dataset", "blended")    # + 25% general
chat     = load_dataset("reloading0101/threat-intelligence-dataset", "multi_turn") # multi-turn
```

## Format

Single-turn records use instruction / input / output:

```json
{
  "instruction": "Analyze CVE-2024-23897 and explain its exploitation risk.",
  "input": "CVE: CVE-2024-23897 (Jenkins CLI)",
  "output": "...grounded analysis...",
  "metadata": {
    "category": "vulnerabilities-cves",
    "task_type": "explain",
    "grounding": "CISA KEV + EPSS",
    "source_refs": { "cves": ["CVE-2024-23897"] }
  }
}
```

Multi-turn records use the standard `messages` list.

## Sources

MITRE ATT&CK v19.1 (Enterprise, Mobile, ICS), MITRE ATLAS, MITRE CWE, MITRE CAPEC, MITRE D3FEND, MITRE Engage, CISA KEV, CISA CSAF ICS advisories (2026), the VERIS Community Database, FIRST.org EPSS, AttackerKB, OSV.dev, SigmaHQ, abuse.ch (Feodo, SSLBL, URLhaus, ThreatFox), MalwareBazaar, Malpedia, ransomware.live, OpenPhish, OWASP Top 10 and API Top 10, STRIDE, NIST SP 800-61, the Diamond Model and the Pyramid of Pain. The general-instruction data in the blended file comes from databricks-dolly-15k.

## Construction

1. The sources are parsed into a knowledge base that links techniques to tactics and mitigations, groups to malware, CVEs to EPSS scores, and so on.
2. Records are generated from entities in that knowledge base. The generator only arranges facts that are already in it.
3. A language model rephrases each instruction.
4. A language model rewrites each answer as analyst prose. A check compares IDs, numbers and names before and after, and the original text is kept when they differ.
5. Near-duplicates are removed and categories are capped.
6. A final pass re-checks every ID against the knowledge base and drops records that fail.

## Categories

42 categories with 155 to 443 examples each:

`adversary-engagement`, `ai-ml-threats`, `api-security`, `attack-patterns`, `attribution-analysis`, `botnet-infrastructure`, `campaign-analysis`, `cloud-saas-security`, `container-security`, `cryptojacking-mining`, `cyber-espionage-apt`, `dark-web-cybercrime`, `data-exfiltration`, `deception-technology`, `defensive-countermeasures`, `digital-risk-management`, `email-threats`, `geopolitical-threats`, `ics-advisories`, `ics-ot-security`, `incident-case-analysis`, `incident-response-forensics`, `insider-threats`, `malicious-campaigns`, `malware`, `mobile-iot-threats`, `network-based-threats`, `ransomware-operations`, `red-team-operations`, `security-monitoring-detection`, `social-engineering-fraud`, `supply-chain-attacks`, `threat-actors`, `threat-hunting`, `threat-intelligence`, `threat-intelligence-feeds`, `threat-intelligence-operations`, `threat-modeling`, `ttps-mitre-attack`, `vulnerabilities-cves`, `web-application-security`, `zero-day-exploits`

## Fine-tuning

Intended for domain adaptation of an instruction-tuned model (Llama 3.x Instruct, Qwen2.5 Instruct and similar) with LoRA or QLoRA. Train on `train_blended.jsonl`; the general data limits loss of conversational ability. Evaluate on `eval.jsonl` and report scores per category.

Starting values: LoRA rank 16 to 32, learning rate 1e-4 to 2e-4 with a cosine schedule, 2 to 3 epochs, sequence length 2048, loss on the answer only.

## Limitations

- About 62% of records reference ATT&CK, so a model trained on this data will frame answers in ATT&CK terms.
- Cryptojacking, email and social-engineering have 155 to 165 examples each, limited by the source material available.
- The set is mostly single-turn. The `multi_turn` config adds a smaller set of two-turn conversations.
- About 330 records are scenario-style: an analyst describes a situation and the answer reasons over real facts, including cases where the right answer is that the evidence is not enough to attribute. The rest are one task per entity.
- Answers follow a report layout, which suits analysis tasks and less so open-ended chat.

## Ethics and scope

Intended for defensive security, education and research. The set contains no working exploits and no private data. The `incident-case-analysis` records quote public breach summaries from the VERIS Community Database, which can name the affected organization. Indicators are either reserved ranges (RFC 5737 / 2606) or publicly reported indicators used for detection.

## License

The dataset is released under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/): attribution, non-commercial use, share-alike. Source terms:

- **Malpedia** (CC BY-NC-SA 3.0): non-commercial, credits must be kept. Used in `malware` and `cryptojacking-mining`.
- **ransomware.live**: free for non-commercial use only, must be credited. Source: Ransomware.live. Used in `ransomware-operations` and `dark-web-cybercrime`. Records are derived summaries, not a copy of the feed.
- **VERIS Community Database** (CC BY-SA 4.0) and **OWASP Top 10** (CC BY-SA 4.0): share-alike.
- **databricks-dolly-15k** (CC BY-SA 3.0): only in `train_blended.jsonl`.
- **SigmaHQ** rules (Detection Rule License 1.1): credit the rule authors.
- **MITRE ATT&CK, ATLAS, CWE, CAPEC, D3FEND**: MITRE's terms of use. **MITRE Engage**: Apache-2.0. **CISA** KEV and ICS advisories: published by CISA. **abuse.ch** feeds: CC0.
- **OpenPhish**: its terms prohibit commercial use and redistribution of the feed. 96 records in `email-threats` and `social-engineering-fraud` have `OpenPhish` in `metadata.grounding`; filter on that field if this matters for your use.

Check the individual terms against your intended use before redistributing.

## Citation

```bibtex
@misc{cti-instruction-tuning-dataset,
  title  = {Cyber Threat Intelligence Instruction-Tuning Dataset},
  author = {Reloading},
  year   = {2026},
  url    = {https://huggingface.co/datasets/reloading0101/threat-intelligence-dataset}
}
```
