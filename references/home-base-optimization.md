# Home-base optimization for SIFF itinerary planning

Use this when the user provides a home, hotel, office, or other base point.

## Goal

Do not optimize only for cinema-to-cinema transfers. A plan with same-cinema screenings can still be bad if it forces the user to wait around for hours, and a valid cross-cinema transfer can still be undesirable if the cinema is far from the user's base.

## Procedure

1. Geocode or otherwise resolve the base point coordinates.
2. Compute base ↔ candidate cinema travel time for every cinema that appears in candidate screenings.
   - Prefer OSRM/driving and an available public-transit estimate when possible.
   - Keep straight-line distance only as a rough sort key, never as the final travel estimate.
3. Grade cinema proximity to the base. Reasonable defaults for central Shanghai:
   - `近`: driving distance ≤ 5 km
   - `可接受`: > 5 km and ≤ 8 km
   - `偏远`: > 8 km
   Adjust thresholds if the user gives stricter preferences.
4. For any planned same-day adjacent screenings, compute whether returning to base is worthwhile:

```text
home_stay_min = next_start
              - previous_end
              - exit_buffer
              - adjusted(previous_cinema → base)
              - adjusted(base → next_cinema)
              - arrive_buffer
```

Default buffers:
- `exit_buffer`: 10 minutes after the previous screening ends.
- `arrive_buffer`: 15 minutes before the next screening starts.
- A home detour is worthwhile if `home_stay_min >= 45` unless the user says otherwise.

5. If a chosen screening is at a `偏远` cinema, search all official screenings of the same title for closer alternatives. Prefer replacing it when:
   - the replacement does not conflict with already selected high-priority screenings;
   - it reduces a `偏远` cinema to `近` or `可接受`;
   - it does not create a worse long wait or late-night return.

## Output expectations

For each relevant day, say one of:
- `中间回家`: include estimated time at home and travel assumptions.
- `留在影院附近`: include why home detour is not worth it.
- `替换偏远影院`: give original screening, replacement screening, and distance reduction.

Do not present long same-cinema waiting as automatically good. Same-cinema only means transfer risk is low; comfort may still be poor.
