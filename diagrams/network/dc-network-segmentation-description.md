# Single Data Center Logical Network Segmentation Diagram - Draw.io Instructions

## Overview
This document provides step-by-step instructions for creating a logical single data center network segmentation diagram in draw.io that shows how a data center is segmented into security zones and VLANs following enterprise security best practices. This design is replicated identically across all data centers (NY, Paris, Toronto). Scope note: this diagram intentionally omits inter-data center links and focuses only on internal logical segmentation.

---

## 1. Diagram Components (Boxes/Shapes)

### 1.1 Data Center Container
Note: "Data Center" will be abbreviated as "DC" throughout this document.

Create **1 large rectangular box** (use "Rectangle" shape, rounded corners optional):
- **Label**: `Data Center` (or `DC` for brevity)
- **Position**: Center of diagram, large enough to contain all zones
- **Style suggestion**: Light gray or white fill with dark border (2-3px), serves as container
- **Size**: Approximately 800px wide × 1000px tall (adjustable based on content)

### 1.2 Security Zone Boxes
Inside the DC container, create **4 rectangular boxes** stacked vertically (use "Rectangle" shape):

1. **DMZ Zone**
   - Label: `DMZ Zone`
   - Position: Top of DC container
   - Style suggestion: Red or orange fill (light shade, ~20% opacity), dark text
   - Border: Red or orange (2px solid)

2. **Internal Zone**
   - Label: `Internal Zone`
   - Position: Second from top (below DMZ)
   - Style suggestion: Green or blue fill (light shade, ~20% opacity), dark text
   - Border: Green or blue (2px solid)

3. **Storage Zone**
   - Label: `Storage Zone`
   - Position: Third from top (below Internal)
   - Style suggestion: Purple or indigo fill (light shade, ~20% opacity), dark text
   - Border: Purple or indigo (2px solid)

4. **Management Zone**
   - Label: `Management Zone`
   - Position: Bottom of DC container (below Storage)
   - Style suggestion: Yellow or amber fill (light shade, ~20% opacity), dark text
   - Border: Yellow or amber (2px solid)

### 1.3 VLAN Labels
Inside each zone box, add text labels listing the VLANs:

**DMZ Zone:**
- `VLAN 10`
- `VLAN 20`

**Internal Zone:**
- `VLAN 30`
- `VLAN 40`
- `VLAN 50`

**Storage Zone:**
- `VLAN 60`
- `VLAN 70`
- `VLAN 80`

**Management Zone:**
- `VLAN 90`
- `VLAN 100`
- `VLAN 110`
- `VLAN 120`

**Formatting**: List VLANs vertically, use bullet points or simple text list, 10-11pt font, positioned inside each zone box

### 1.4 External Internet Representation
Create **1 cloud or box shape** outside the DC container:
- **Label**: `Internet`
- **Position**: Top-left or top-center, outside DC container
- **Style suggestion**: Cloud shape (if available) or rounded rectangle, light blue fill, dark text

---

## 2. Connections (Links/Arrows)

### 2.1 Internet → DMZ Connection
- **Type**: Solid line, medium thickness (2-3px)
- **Style**: Dark blue or black
- **Direction**: Unidirectional arrow (Internet → DMZ)
- **Label**: 
  - Primary label: `Public Traffic`
  - Secondary label (optional): `HTTPS/HTTP`
- **Position**: Horizontal line from Internet to DMZ Zone (top edge)

### 2.2 DMZ → Internal Connection
- **Type**: Solid line, medium thickness (2-3px)
- **Style**: Green or dark green
- **Direction**: Unidirectional arrow (DMZ → Internal)
- **Label**:
  - Primary label: `Authenticated Traffic`
  - Secondary label (optional): `Filtered by Firewall`
- **Position**: Vertical line from DMZ Zone (bottom edge) to Internal Zone (top edge)

### 2.3 Internal → Storage Connection
- **Type**: Solid line, medium thickness (2-3px)
- **Style**: Purple or dark purple
- **Direction**: Unidirectional arrow (Internal → Storage)
- **Label**:
  - Primary label: `Data Access`
  - Secondary label (optional): `Authorized Applications Only`
- **Position**: Vertical line from Internal Zone (bottom edge) to Storage Zone (top edge)

### 2.4 Management → All Zones Connections
Create **4 connections** from Management Zone to each other zone:

**Management → DMZ:**
- **Type**: Dashed line, thin (1-2px)
- **Style**: Yellow or amber
- **Direction**: Unidirectional arrow (Management → DMZ)
- **Label**: `Administration Only`
- **Position**: Diagonal or curved line from Management to DMZ

**Management → Internal:**
- **Type**: Dashed line, thin (1-2px)
- **Style**: Yellow or amber
- **Direction**: Unidirectional arrow (Management → Internal)
- **Label**: `Administration Only`
- **Position**: Diagonal or curved line from Management to Internal

**Management → Storage:**
- **Type**: Dashed line, thin (1-2px)
- **Style**: Yellow or amber
- **Direction**: Unidirectional arrow (Management → Storage)
- **Label**: `Administration Only`
- **Position**: Diagonal or curved line from Management to Storage

**Management → Management (self-loop, optional):**
- Can be omitted or shown as internal management traffic

### 2.5 All Zones → Management Connections
Create **3 connections** from each zone (except Management) to Management Zone:

**DMZ → Management:**
- **Type**: Dashed line, thin (1-2px)
- **Style**: Gray or light gray
- **Direction**: Unidirectional arrow (DMZ → Management)
- **Label**: `Logs & Monitoring`
- **Position**: Diagonal or curved line from DMZ to Management

**Internal → Management:**
- **Type**: Dashed line, thin (1-2px)
- **Style**: Gray or light gray
- **Direction**: Unidirectional arrow (Internal → Management)
- **Label**: `Logs & Monitoring`
- **Position**: Diagonal or curved line from Internal to Management

**Storage → Management:**
- **Type**: Dashed line, thin (1-2px)
- **Style**: Gray or light gray
- **Direction**: Unidirectional arrow (Storage → Management)
- **Label**: `Logs & Monitoring`
- **Position**: Diagonal or curved line from Storage to Management

---

## 3. Explicitly Denied Traffic Patterns (Annotations)

Add **text annotations or note boxes** clearly stating denied traffic patterns:

### 3.1 Denied Traffic Note Box
Create a **note box or text annotation** (position: bottom-right corner, outside DC container):

**Title**: `Explicitly Denied Traffic Patterns`

**Content** (list format):
- Internet → Internal (bypassing DMZ): DENIED
- Internet → Storage: DENIED
- Internet → Management: DENIED
- DMZ → Storage (direct): DENIED
- DMZ → Management (direct, except logs): DENIED
- Internal → DMZ (reverse): DENIED
- Storage → Internal (reverse): DENIED
- Storage → DMZ: DENIED
- Any zone → Internet (outbound, except DMZ): DENIED
- Cross-zone direct communication (except via defined paths): DENIED

**Style**: Red border, light red fill, or use warning icon/color scheme

### 3.2 Visual Denial Indicators (Optional)
For clarity, you may add:
- Red "X" symbols on connections that should not exist
- Or simply rely on the absence of connections + annotation box

---

## 4. Labels and Annotations

### 4.1 Zone Labels
For each zone box:
- **Primary label**: Zone name (e.g., "DMZ Zone") - Bold, 14-16pt font
- **VLAN list**: Inside zone, regular text, 10-11pt font

### 4.2 Connection Labels
For each connection arrow:
- **Primary label**: Traffic type (e.g., "Public Traffic", "Authenticated Traffic")
- **Secondary label** (if space permits): Additional details (e.g., "Filtered by Firewall")
- **Font size**: 10-11pt, positioned near the arrow

### 4.3 Security Note
Add a legend or note box:
- **Title**: `Security Principles`
- **Content**: 
  - `Traffic flows follow least privilege principle`
  - `All inter-zone traffic filtered by firewalls`
  - `Management zone has administrative access to all zones`
  - `All zones send logs/monitoring to Management zone`
- **Position**: Top-right or bottom-left corner

---

## 5. Visual Hierarchy and Styling

### 5.1 Color Scheme
- **DMZ Zone**: Red or orange (external-facing, higher risk)
- **Internal Zone**: Green or blue (internal, trusted)
- **Storage Zone**: Purple or indigo (data storage, restricted)
- **Management Zone**: Yellow or amber (administrative, critical)
- **DC Container**: Light gray or white (neutral container)
- **Internet**: Light blue (external)

### 5.2 Line Styles
- **Allowed traffic flows**: Solid lines (2-3px)
  - Internet → DMZ: Dark blue or black
  - DMZ → Internal: Green
  - Internal → Storage: Purple
- **Administrative flows**: Dashed lines (1-2px), yellow/amber
  - Management → All zones
- **Monitoring flows**: Dashed lines (1-2px), gray
  - All zones → Management
- **All connections**: Include arrowheads to show direction

### 5.3 Text Formatting
- **DC container label**: Bold, 16-18pt font
- **Zone names**: Bold, 14-16pt font
- **VLAN labels**: Regular, 10-11pt font
- **Connection labels**: Regular, 10-11pt font
- **Denied traffic annotations**: Regular, 9-10pt font, red color

---

## 6. Layout Structure

### 6.1 Recommended Arrangement
```
                    [Internet]
                        ↓
        ┌─────────────────────────────────┐
        │      [Data Center]              │
        │                                 │
        │  ┌───────────────────────────┐  │
        │  │   DMZ Zone                │  │
        │  │   VLAN 10                 │  │
        │  │   VLAN 20                 │  │
        │  └───────────────────────────┘  │
        │              ↓                   │
        │  ┌───────────────────────────┐  │
        │  │   Internal Zone           │  │
        │  │   VLAN 30                  │  │
        │  │   VLAN 40                  │  │
        │  │   VLAN 50                  │  │
        │  └───────────────────────────┘  │
        │              ↓                   │
        │  ┌───────────────────────────┐  │
        │  │   Storage Zone            │  │
        │  │   VLAN 60                 │  │
        │  │   VLAN 70                 │  │
        │  │   VLAN 80                 │  │
        │  └───────────────────────────┘  │
        │                                 │
        │  ┌───────────────────────────┐  │
        │  │   Management Zone          │  │
        │  │   VLAN 90                 │  │
        │  │   VLAN 100                │  │
        │  │   VLAN 110                │  │
        │  │   VLAN 120                │  │
        │  └───────────────────────────┘  │
        │                                 │
        └─────────────────────────────────┘
```

### 6.2 Spacing
- **Between zones**: ~50-80px vertical spacing
- **Zone padding**: ~20-30px internal padding for VLAN labels
- **DC container padding**: ~40-50px from zones to container edge
- **Internet to DC**: ~100-150px horizontal spacing

### 6.3 Zone Sizing
- **All zones**: Approximately same width (fill DC container width minus padding)
- **Zone height**: Adjust based on number of VLANs (Management zone may be taller)
- **Consistent alignment**: All zones left-aligned within DC container

---

## 7. Optional Enhancements

### 7.1 Legend Box
Create a legend in the corner (outside DC container):
- **Solid line**: Allowed traffic flow
- **Dashed line (yellow)**: Administrative access
- **Dashed line (gray)**: Logs & monitoring
- **Red/Orange**: DMZ Zone
- **Green/Blue**: Internal Zone
- **Purple/Indigo**: Storage Zone
- **Yellow/Amber**: Management Zone

### 7.2 Firewall Icons (Optional)
- Add small firewall icons between zones on connection lines
- Or add firewall labels near connection points

### 7.3 VLAN ID Formatting
- Use consistent format: "VLAN 10", "VLAN 20", etc.
- Consider using monospace font for VLAN numbers for clarity

---

## 8. Step-by-Step Creation Checklist

1. ✅ Create DC container box (large rectangle)
2. ✅ Label DC container as "Data Center"
3. ✅ Create DMZ Zone box (top position)
4. ✅ Add VLAN 10 and VLAN 20 labels inside DMZ Zone
5. ✅ Create Internal Zone box (below DMZ)
6. ✅ Add VLAN 30, VLAN 40, VLAN 50 labels inside Internal Zone
7. ✅ Create Storage Zone box (below Internal)
8. ✅ Add VLAN 60, VLAN 70, VLAN 80 labels inside Storage Zone
9. ✅ Create Management Zone box (bottom position)
10. ✅ Add VLAN 90, VLAN 100, VLAN 110, VLAN 120 labels inside Management Zone
11. ✅ Create Internet representation (outside DC container)
12. ✅ Add Internet → DMZ connection (solid line, labeled "Public Traffic")
13. ✅ Add DMZ → Internal connection (solid line, labeled "Authenticated Traffic")
14. ✅ Add Internal → Storage connection (solid line, labeled "Data Access")
15. ✅ Add Management → DMZ connection (dashed line, labeled "Administration Only")
16. ✅ Add Management → Internal connection (dashed line, labeled "Administration Only")
17. ✅ Add Management → Storage connection (dashed line, labeled "Administration Only")
18. ✅ Add DMZ → Management connection (dashed line, labeled "Logs & Monitoring")
19. ✅ Add Internal → Management connection (dashed line, labeled "Logs & Monitoring")
20. ✅ Add Storage → Management connection (dashed line, labeled "Logs & Monitoring")
21. ✅ Add denied traffic patterns annotation box
22. ✅ Apply color scheme to zones (DMZ: red/orange, Internal: green/blue, Storage: purple/indigo, Management: yellow/amber)
23. ✅ Apply appropriate line styles (solid for allowed flows, dashed for admin/monitoring)
24. ✅ Add security principles note/legend
25. ✅ Review layout and spacing
26. ✅ Verify all VLANs are listed correctly
27. ✅ Verify all traffic flows are represented
28. ✅ Verify denied traffic patterns are clearly annotated
29. ✅ Add legend if needed
30. ✅ Final review for clarity and completeness

---

## 9. Key Design Principles

- **Clarity**: Each zone is visually distinct with color coding
- **Hierarchy**: Traffic flows follow logical progression (Internet → DMZ → Internal → Storage)
- **Completeness**: All VLANs documented, all allowed flows shown, all denied flows explicitly stated
- **Professional**: Clean, architectural style (not artistic)
- **Informative**: Labels provide necessary technical details (VLAN IDs, traffic types)
- **Security-focused**: Denied traffic patterns clearly documented
- **Logical layout**: Zones stacked vertically showing data flow progression

---

## 10. Traffic Flow Summary

### 10.1 Allowed Traffic Flows
1. **Internet → DMZ**: Public traffic (HTTPS/HTTP)
2. **DMZ → Internal**: Authenticated traffic (filtered by firewall)
3. **Internal → Storage**: Data access (authorized applications only)
4. **Management → All Zones**: Administrative access (dashed lines)
5. **All Zones → Management**: Logs & monitoring (dashed lines)

### 10.2 Denied Traffic Flows (Explicitly Stated)
- Internet → Internal (bypassing DMZ): DENIED
- Internet → Storage: DENIED
- Internet → Management: DENIED
- DMZ → Storage (direct): DENIED
- DMZ → Management (direct, except logs): DENIED
- Internal → DMZ (reverse): DENIED
- Storage → Internal (reverse): DENIED
- Storage → DMZ: DENIED
- Any zone → Internet (outbound, except DMZ): DENIED
- Cross-zone direct communication (except via defined paths): DENIED

---

## 11. Final Review Points

Before finalizing, ensure:
- ✅ DC container is clearly visible and labeled
- ✅ All 4 security zones are present and properly labeled
- ✅ All VLANs are listed inside their respective zones (DMZ: 10, 20; Internal: 30, 40, 50; Storage: 60, 70, 80; Management: 90, 100, 110, 120)
- ✅ Internet → DMZ → Internal → Storage flow is clearly shown
- ✅ Management → all zones connections are shown (administrative access)
- ✅ All zones → Management connections are shown (logs & monitoring)
- ✅ Denied traffic patterns are explicitly stated in annotation box
- ✅ Color scheme is applied consistently
- ✅ Line styles differentiate allowed flows (solid) from admin/monitoring (dashed)
- ✅ All connections have appropriate labels
- ✅ Zones are stacked vertically in logical order
- ✅ Diagram is readable at standard print/view sizes
- ✅ No physical hardware details (switches, cables, vendors) are shown
- ✅ Diagram represents logical segmentation only

---

## 12. Example Zone Label Format

For DMZ Zone:
```
DMZ Zone
VLAN 10
VLAN 20
```

For Internal Zone:
```
Internal Zone
VLAN 30
VLAN 40
VLAN 50
```

For Storage Zone:
```
Storage Zone
VLAN 60
VLAN 70
VLAN 80
```

For Management Zone:
```
Management Zone
VLAN 90
VLAN 100
VLAN 110
VLAN 120
```

---

## 13. Notes on Replication Across Data Centers

This diagram design is replicated identically across all three data centers (NY, Paris, Toronto). Each data center follows the same:
- Zone structure (DMZ, Internal, Storage, Management)
- VLAN assignments (same VLAN IDs in each zone)
- Traffic flow rules
- Security segmentation principles

The only difference between data centers would be the DC container label (e.g., "New York DC", "Paris DC", "Toronto DC"), but the internal segmentation remains identical.

