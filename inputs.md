## 5. Raw Measurement Data Format

```json
{
    "device_id": "string",
    "measurement_type": "MEASUREMENT_TYPE",
    "timestamp": "ISO8601",
    "raw_value": "string",           // ACVPU or number
    "battery": "number",       
    "signal_strength": "number"
}
```
MEASUREMENT_TYPE is one of:
- RESPIRATORY_RATE
- OXYGEN_SATURATION
- BLOOD_PRESSURE_SYSTOLIC
- BLOOD_PRESSURE_DIASTOLIC
- HEART_RATE
- TEMPERATURE
- CONSCIOUSNESS