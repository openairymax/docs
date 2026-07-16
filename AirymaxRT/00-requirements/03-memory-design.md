Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.  
"From data intelligence emerges."

# Airymax Memory Layer Theory
> **文档定位**：Airymax Memory Layer Theory\
> **最后更新**：2026-06-09\
> **上级文档**：[AirymaxAgentRT 文档中心](README.md)\
> **Terminology Standardization Note**：The forgetting formula uniformly uses the $\tau$ notation (consistent with 00-architectural-principles.md C-4). The historical $\lambda$ notation ($R = e^{-\lambda t}$) is mathematically equivalent ($\lambda = 1/\tau$), but this specification uniformly uses $\tau$ for terminology consistency. See TERMINOLOGY.md v3.0 ruling #6.\
> **Storage**：Raw memory data (conversations, events, files), append-only writing, never modified. Each record includes metadata (timestamp, source, TraceID).\
> **Mathematical form**：Each memory is encoded as a high-order tensor:\
> **Function**：Maps raw memories to high-dimensional semantic space via embedding models, generating feature vectors. Supports hybrid retrieval (vector+BM25), ensuring semantically similar memories are close in feature space.\
> **Key property**：The embedding mapping $\Phi$ satisfies Lipschitz continuity:

***

**References**

1. Carlsson, G. (2009). Topology and data. *Bulletin of the American Mathematical Society*, 46(2), 255-308.
2. Ebbinghaus, H. (1885). *Memory: A Contribution to Experimental Psychology*.
3. LeCun, Y., Bengio, Y., & Hinton, G. (2015). Deep learning. *Nature*, 521(7553), 436-444.
4. Squire, L. R., & Zola-Morgan, S. (1991). The medial temporal lobe memory system. *Science*, 253(5026), 1380-1386.
5. Hopfield, J. J. (1982). Neural networks and physical systems with emergent collective computational abilities. *Proceedings of the National Academy of Sciences*, 79(8), 2554-2558.
