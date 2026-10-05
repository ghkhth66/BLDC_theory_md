# LEAD ANGLE OPTIMIZER DEPENDENCY REPORT


# generate_lead_angle_candidates

Lines : 25
Start : 14133
End   : 14157

## SELF READS

- self.lead_angle_optimizer_max_angle_deg

## SELF WRITES

- NONE

## SELF FUNCTION CALLS

- lead_angle_optimizer_max_angle_deg()

## EXTERNAL CALLS

- candidates.append
- float
- max
- min
- round

## EXTRACTION SCORE

Dependency Score : 2
V2 Extraction : EASY

# evaluate_lead_angle_candidate

Lines : 193
Start : 14159
End   : 14351

## SELF READS

- self.build_lead_angle_optimizer_context

## SELF WRITES

- NONE

## SELF FUNCTION CALLS

- build_lead_angle_optimizer_context()

## EXTERNAL CALLS

- abs
- context.get
- float
- math.cos
- math.radians
- math.sin
- math.sqrt
- max
- min

## EXTRACTION SCORE

Dependency Score : 2
V2 Extraction : EASY

# run_lead_angle_optimizer_voltage_relief_final

Lines : 363
Start : 14520
End   : 14882

## SELF READS

- self.build_lead_angle_optimizer_context
- self.evaluate_lead_angle_candidate
- self.generate_lead_angle_candidates
- self.get_integrated_controller_state
- self.get_lead_angle_optimizer_state
- self.get_lead_angle_state

## SELF WRITES

- NONE

## SELF FUNCTION CALLS

- build_lead_angle_optimizer_context()
- evaluate_lead_angle_candidate()
- generate_lead_angle_candidates()
- get_integrated_controller_state()
- get_lead_angle_optimizer_state()
- get_lead_angle_state()

## EXTERNAL CALLS

- abs
- baseline.get
- best.get
- evaluated.append
- float
- item.get
- len
- max
- min
- state.update
- viable.append

## EXTRACTION SCORE

Dependency Score : 12
V2 Extraction : EASY
