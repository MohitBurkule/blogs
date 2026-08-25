# blogs

Long-form writing on software engineering, mostly things I've had to work out
in practice and wanted to write down properly.

## Posts

| Post | What it's about |
|---|---|
| [Extensible Software: The Missing Half](posts/extensible-software-the-missing-half.md) | LLMs collapsed the cost of writing an extension and sandboxes collapsed the cost of running one — but that is the middle of the problem. Why the selection barrier (users not knowing what to ask for) is a thirty-year-old finding, why the sandbox probably belongs in the browser rather than on a worker, and what user-authored writes do to a database you still need to query. |
| [What Actually Changed Between Two Generations of the Same Model](posts/qwen-weight-diff.md) | Qwen3.6-27B and Qwen3.8-27B ship byte-identical architectures, so the whole generational difference is in the weights. Reading 1,182 tensors without downloading either checkpoint, transplanting components between them, and comparing activations — plus four of my own conclusions that turned out to be division. |
| [What LittleLearner Actually Shows About Knowledge Boundaries in LLMs](posts/littlelearner-knowledge-boundary.md) | A 5B model trained only on elementary-school text can't be pushed past its training scope by scale, RL, or in-context prompting. What that does and doesn't prove about whether RL creates new capabilities or just reweights old ones. |
| [Reliable Systems from Unreliable Generators](posts/reliable-systems-from-unreliable-generators.md) | An LLM breaks your conventions, so you write a rule. It breaks another one. Where does that end? What DNA replication, Voyager, Toyota, aviation and the FORTRAN compiler all did about the same problem — and what the evidence says actually works. |
