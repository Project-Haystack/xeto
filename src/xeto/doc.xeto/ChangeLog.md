
*Build 5.0.1 (working)*
- Xeto initial design
- Rename volt tag to voltage
- Replace lineAvg and phaseAvg tags with elecLineToLineAvg and elecLineAvg
- Replace phase enum with ElecLocation choice and elecL1, elecL2, elecL3,
  elecL1L2, elecL2L3, elecL3L1, and neutral tags (L1, L2, and L3 instead of
  A, B, and C)
- Rename Xeto base type Seq to Collection
- Deprecate the legacy v3 JSON format
- New Jeto JSON format
- New v5 Xeto JSON HTTP API
- New standard sys.repo HTTP API

