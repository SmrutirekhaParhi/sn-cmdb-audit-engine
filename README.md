# CMDB Audit & Governance Engine ⚙️

An automated ServiceNow backend application designed to maintain CMDB health by detecting and reporting duplicate Configuration Items (CIs) at the database level. 

## 📖 Project Overview
Data pollution in the CMDB (such as duplicate hardware records) can cause severe reporting inaccuracies, false compliance breaches, and software licensing issues. This scoped application acts as an automated governance layer. Every night, a Scheduled Script Execution scans the `cmdb_ci_computer` table. 

To ensure instance performance and prevent memory degradation, the core logic utilizes `GlideAggregate` to group and count records at the database level rather than iterating through records via standard `GlideRecord` loops. If duplicates are found sharing the exact same Serial Number, the system automatically generates an Incident assigned to the Hardware/CMDB team for remediation.

## 🛠️ Technical Architecture
*   **Application Scope:** Custom Scoped Application
*   **Trigger:** Scheduled Script Execution (Daily at 02:00)
*   **Core Logic:** Object-Oriented Script Include (`Accessible from: All application scopes`)
*   **ServiceNow APIs:** `GlideAggregate`, `GlideRecord`

## 💻 Core Logic Snippet
*Below is the primary Script Include demonstrating the optimized database query to identify duplicates:*

```javascript
var CMDBCleanupUtils = Class.create();
CMDBCleanupUtils.prototype = {
    initialize: function() {},

    findDuplicateSerialNumbers: function() {
        var duplicateGroups = 0;
        
        // Utilizing GlideAggregate for database-level counting (Performance Optimization)
        var ga = new GlideAggregate('cmdb_ci_computer');
        ga.addNotNullQuery('serial_number');
        ga.addAggregate('COUNT');
        ga.groupBy('serial_number');
        ga.query();

        while (ga.next()) {
            var count = ga.getAggregate('COUNT');
            
            // If more than one CI shares the exact same serial number, trigger alert
            if (count > 1) {
                var serial = ga.getValue('serial_number');
                this._logIncident(serial, count);
                duplicateGroups++;
            }
        }
        return "Audit complete. Created incidents for " + duplicateGroups + " duplicate groups.";
    },

    _logIncident: function(serialNumber, count) {
        var inc = new GlideRecord('incident');
        inc.initialize();
        inc.short_description = "CMDB Governance Alert: " + count + " duplicates found for Serial Number " + serialNumber;
        inc.description = "Automated CMDB Audit detected multiple Computer CIs sharing the exact same serial number. Please investigate and merge/retire the duplicates.";
        inc.category = "hardware";
        inc.impact = 2;
        inc.urgency = 2;
        inc.insert();
    },

    type: 'CMDBCleanupUtils'
};
