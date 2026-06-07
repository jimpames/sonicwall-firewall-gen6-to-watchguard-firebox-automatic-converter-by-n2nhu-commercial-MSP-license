(venv) C:\Users\jimpames\Desktop\FESv8-algebraic\migration_toolkit>python skeleton_engine/build_pipeline.py --full-rebuild
========================================================================
BUILD PIPELINE
========================================================================
  input: C:\Users\jimpames\Desktop\FESv8-algebraic\migration_toolkit\sonicwall_parser\input\nsa3600.txt
  jpa:   C:\Users\jimpames\Desktop\FESv8-algebraic\migration_toolkit\schema_learner\corpus\T30jpa1.xml
  [1] parse SonicOS config
  [2] enrich to canonical records
  [3a] classify immutables from jpa.xml
  [3b] extract skeleton + templates from jpa.xml
  [3c] scrub jpa-customer-private entries from skeleton
  [4] extract leaves-quality matrix from jpa.xml
  [5] fill skeleton with customer data
[6] qualities_validator
qualities_validator
  candidate: skeleton_engine\output\configuration.xml
  matrix:    C:\Users\jimpames\Desktop\FESv8-algebraic\migration_toolkit\skeleton_engine\output\leaves_quality.json
========================================================================
HARD findings (likely Firebox rejection): 0
SOFT findings (casing/format suspicious): 54
INFO  (leaf not in jpa matrix at all):    1


========================================================================
SOFT findings (first 20 of 54):
========================================================================

  /profile/service-list/service/name
    value: '6over4'
    casing pattern 'lower' not observed in jpa at this XPath; jpa shows ['Mixed', 'Title', 'UPPER', 'Upper-first']. Dominant: 'UPPER'.

  /profile/ipsec-action-list/ipsec-action/name
    value: 'fakesite2site'
    format signature 'a+9a+' not in jpa observed ['A+ A+a9 A+', 'a+.9'] at this XPath.

  /profile/ipsec-action-list/ipsec-action/local-remote-pair-list/local-remote-pair/local-addr
    value: 'fakesite2site.1.tlc'
    format signature 'a+9a+.9.a+' not in jpa observed ['A+ A+a9 A+_a+', 'a+.9.9.a+'] at this XPath.

  /profile/ipsec-action-list/ipsec-action/local-remote-pair-list/local-remote-pair/remote-addr
    value: 'fakesite2site.1.trm'
    format signature 'a+9a+.9.a+' not in jpa observed ['A+ A+a9 A+_a+', 'a+.9.9.a+'] at this XPath.

  /profile/ipsec-action-list/ipsec-action/ike-policy-group
    value: 'fakesite2site'
    format signature 'a+9
[6b] ref_integrity_validator
ref_integrity_validator
  candidate: skeleton_engine\output\configuration.xml
========================================================================
  references checked: 115
  DANGLING references: 0


[6c] immutables_validator
immutables_validator
  candidate: skeleton_engine\output\configuration.xml
  reference: C:\Users\jimpames\Desktop\FESv8-algebraic\migration_toolkit\schema_learner\corpus\T30jpa1.xml
========================================================================
  HARD findings: 0

[6d] customer_fidelity_validator
customer_fidelity_validator
  candidate:    skeleton_engine\output\configuration.xml
  customer dir: C:\Users\jimpames\Desktop\FESv8-algebraic\migration_toolkit\enricher\output
========================================================================
  total customer records: 274
  PRESENT in emit:        274
  MISSING from emit:      0

  ✓  service_objects.json: 205 present, 0 missing  (services)
  ✓  implicit_address_objects.json: 7 present, 0 missing  (address objects)
  ✓  address_groups.json: 0 present, 0 missing  (address groups)
  ✓  schedules.json: 8 present, 0 missing  (schedules)
  ✓  interfaces.json: 2 present, 0 missing, 1 hardware-dropped  (interfaces)
       - hw-dropped: 'MGMT' ip='192.168.1.254' (hardware-slot constraint (T-30 has 5 slots))
  ✓  access_rules.json: 1 present, 0 missing  (access rules)
  ✓  vpn_policies.json: 3 present, 0 missing  (VPN policies)
  ✓  service_groups.json: 47 present, 0 missing  (service groups)

[6e] schema_shape_validator
schema_shape_validator
  candidate: skeleton_engine\output\configuration.xml
  reference: C:\Users\jimpames\Desktop\FESv8-algebraic\migration_toolkit\schema_learner\corpus\T30jpa1.xml
========================================================================
  parent/child pairs in candidate: 1263
  pairs in jpa: 1281
  INVENTED pairs (not observed in jpa): 0


[6f] private_data_validator
private_data_validator
  candidate: skeleton_engine\output\configuration.xml
  config:    C:\Users\jimpames\Desktop\FESv8-algebraic\migration_toolkit\skeleton_engine\scrub_config.ini
  scanning for 10 known-private strings
========================================================================
  HARD findings (jpa-customer-private data leaked): 0

[6g] required_children_validator
required_children_validator
  candidate: skeleton_engine\output\configuration.xml
  reference: C:\Users\jimpames\Desktop\FESv8-algebraic\migration_toolkit\schema_learner\corpus\T30jpa1.xml
========================================================================
  jpa-derived rules: 489
  HARD findings (missing required children): 0

[7] jpa_diff
  jpa_diff comparing:
  ========================================================================
  HARD findings (likely Firebox rejection): 0
  SOFT findings (suspicious, may be benign): 14
  ========================================================================
  SOFT findings (first 20 of 14):
  ========================================================================

========================================================================
FINAL FILE
========================================================================
  path:   C:\Users\jimpames\Desktop\FESv8-algebraic\migration_toolkit\skeleton_engine\output\configuration.xml
  size:   745,548 bytes / 23,732 lines
  SHA256: a03f4aab57af62c2d987401e5e13bce14352fbba84db3f20e2a7071b6283a40e
  qualities_validator exit:     0
  ref_integrity_validator exit: 0
  immutables_validator exit:    0
  customer_fidelity exit:       0
  schema_shape exit:            0
  private_data exit:            0
  required_children exit:       0
  jpa_diff exit:                0
  push to lab if ALL EIGHT are 0; otherwise inspect findings

(venv) C:\Users\jimpames\Desktop\FESv8-algebraic\migration_toolkit>
