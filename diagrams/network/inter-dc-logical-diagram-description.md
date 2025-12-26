# Inter-Data Center Logical Network Diagram - Draw.io Instructions

## Overview
This document provides step-by-step instructions for creating a logical inter-data center network diagram in draw.io that shows the connectivity between New York, Paris, and Toronto data centers. Scope note: this diagram intentionally omits internal data center segmentation and focuses only on inter-site connectivity.

---

## 1. Diagram Components (Boxes/Shapes)

### 1.1 Data Center Boxes
Note: "Data Center" will be abbreviated as "DC" throughout this document.

Create **3 rectangular boxes** (use "Rectangle" shape, rounded corners optional):

1. **New York Data Center**
   - Label: `New York DC`
   - Subtitle/Secondary text: `Active/Active Primary`
   - Position: Left side of diagram
   - Style suggestion: Blue or green fill (light shade), dark text

2. **Paris Data Center**
   - Label: `Paris DC`
   - Subtitle/Secondary text: `Active/Active Primary`
   - Position: Right side of diagram (same vertical level as NY)
   - Style suggestion: Blue or green fill (light shade), dark text

3. **Toronto Data Center**
   - Label: `Toronto DC`
   - Subtitle/Secondary text: `Backup / DR Only`
   - Note: Toronto is Backup/DR only — receives replication/backup traffic only; no user-facing traffic is allowed.
   - Position: Bottom center of diagram
   - Style suggestion: Gray or orange fill (light shade), dark text (to differentiate from primaries)

### 1.2 Gateway/Connection Points (Optional but Recommended)
Inside each DC box, add small circles or icons to represent:
- **IPsec VPN Gateway (Firewall)** (one per DC)
- Label: `IPsec VPN Gateway (Firewall)`
- These represent the termination points for encrypted connections

---

## 2. Connections (Links/Arrows)

### 2.1 NY ↔ Paris Connection (Primary Path)
- **Type**: Solid line, thicker than backup paths
- **Style**: Dark blue or dark green
- **Direction**: Bidirectional (double arrow or arrow on both ends)
- **Label**: 
  - Primary label: `Dedicated Leased Line`
  - Secondary label (below): `IPsec Encrypted (Primary)`
  - Optional bandwidth: `1-10 Gbps`
- **Position**: Horizontal line connecting NY and Paris boxes

### 2.2 NY ↔ Paris Connection (Backup Path)
- **Type**: Dashed line, medium thickness
- **Style**: Same color as primary but lighter, or orange/yellow
- **Direction**: Bidirectional (double arrow)
- **Label**:
  - Primary label: `Internet IPsec VPN`
  - Secondary label: `Backup Path`
  - Optional bandwidth: `500 Mbps - 1 Gbps`
- **Position**: Parallel to primary path, slightly offset (above or below)

### 2.3 NY → Toronto Connection
- **Type**: Dashed line, medium thickness
- **Style**: Gray or orange (to match backup/DR theme)
- **Direction**: Unidirectional arrow (NY → Toronto)
- **Label**:
  - Primary label: `IPsec VPN over Internet`
  - Secondary label: `Backup Traffic Only`
  - Optional bandwidth: `1-2 Gbps`
- **Position**: Diagonal line from NY (bottom) to Toronto (top-left)

### 2.4 Paris → Toronto Connection
- **Type**: Dashed line, medium thickness
- **Style**: Gray or orange (to match backup/DR theme)
- **Direction**: Unidirectional arrow (Paris → Toronto)
- **Label**:
  - Primary label: `IPsec VPN over Internet`
  - Secondary label: `Backup Traffic Only`
  - Optional bandwidth: `1-2 Gbps`
- **Position**: Diagonal line from Paris (bottom) to Toronto (top-right)

---

## 3. Labels and Annotations

### 3.1 Connection Labels
For each connection, add text labels:
- **Primary label**: Connection type (e.g., "Dedicated Leased Line")
- **Secondary label**: Encryption/role (e.g., "IPsec Encrypted (Primary)")
- **Optional**: Bandwidth, latency, or other technical details

### 3.2 Role Labels
Add text boxes or annotations near each DC:
- **NY & Paris**: "Active/Active" or "Production Primary"
- **Toronto**: "Backup/DR Only - No User Traffic"

### 3.3 Encryption Note
Add a legend or note box:
- Title: `Encryption`
- Content: `All inter-DC traffic encrypted with IPsec (AES-256-GCM)`
- Position: Top-right or bottom-right corner

---

## 4. Visual Hierarchy and Styling

### 4.1 Color Scheme
- **Primary DCs (NY, Paris)**: Blue or green (active/production)
- **DR DC (Toronto)**: Gray or orange (backup/non-production)
- **Primary connections**: Dark, solid lines
- **Backup connections**: Lighter, dashed lines

### 4.2 Line Styles
- **Primary path (NY-Paris leased line)**: Solid, thick (3-4px)
- **Backup paths**: Dashed (2-3px)
- **All connections**: Include arrowheads to show direction

### 4.3 Text Formatting
- **DC names**: Bold, 14-16pt font
- **DC roles**: Italic, 10-12pt font
- **Connection labels**: Regular, 10-11pt font
- **Bandwidth info**: Smaller, 9pt font, gray color

---

## 5. Layout Structure

### 5.1 Recommended Arrangement
```
        [New York DC]  ←→  [Paris DC]
         Active/Active      Active/Active
              ↓                   ↓
              └────→  [Toronto DC]  ←────┘
                    Backup/DR Only
```

### 5.2 Spacing
- Horizontal spacing between NY and Paris: ~400-500px
- Vertical spacing from primaries to Toronto: ~300-400px
- Ensure connections don't overlap unnecessarily

---

## 6. Optional Enhancements

### 6.1 Legend Box
Create a legend in the corner:
- **Solid line**: Primary path
- **Dashed line**: Backup path
- **Blue/Green**: Production
- **Gray/Orange**: Backup/DR

### 6.2 Traffic Flow Indicators
- Add small arrows or flow indicators on connections
- Use different arrow styles for primary vs. backup

### 6.3 Network Segments
- Optionally show internal network segments within each DC box
- Show gateway placement within DCs

---

## 7. Step-by-Step Creation Checklist

1. ✅ Create 3 DC boxes (NY, Paris, Toronto)
2. ✅ Label each DC with name and role
3. ✅ Add NY ↔ Paris primary connection (solid line, labeled)
4. ✅ Add NY ↔ Paris backup connection (dashed line, labeled)
5. ✅ Add NY → Toronto connection (dashed line, labeled)
6. ✅ Add Paris → Toronto connection (dashed line, labeled)
7. ✅ Add encryption note/legend
8. ✅ Apply color scheme (primaries vs. DR)
9. ✅ Add connection labels (type, encryption, bandwidth)
10. ✅ Review layout and spacing
11. ✅ Verify all connections are labeled correctly
12. ✅ Add legend if needed

---

## 8. Key Design Principles

- **Clarity**: Each connection type is visually distinct
- **Hierarchy**: Primary paths are more prominent than backup paths
- **Completeness**: All connections documented with encryption status
- **Professional**: Clean, architectural style (not artistic)
- **Informative**: Labels provide necessary technical details

---

## 9. Example Connection Label Format

For NY ↔ Paris Primary:
```
Dedicated Leased Line
IPsec Encrypted (Primary)
1-10 Gbps
```

For NY → Toronto:
```
IPsec VPN over Internet
Backup Traffic Only
1-2 Gbps
```

---

## 10. Final Review Points

Before finalizing, ensure:
- ✅ All three DCs are clearly labeled with their roles
- ✅ NY-Paris has both primary (solid) and backup (dashed) paths
- ✅ Toronto connections are unidirectional (NY→Toronto, Paris→Toronto)
- ✅ All connections indicate IPsec encryption
- ✅ Toronto is visually distinct as Backup/DR only
- ✅ No user-facing traffic shown to Toronto
- ✅ Diagram is readable at standard print/view sizes

