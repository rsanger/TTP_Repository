```json
{
  "NDM_metadata": {
    "authority": "org.opennetworking.fawg",
    "OF_protocol_version": "1.3",
    "type": "TTPv1",
    "name": "VID-MAC",
    "version": "1.0.0",
    "doc": ["TTP supporting VID/MAC unicast and multicast with VID translation."]
  },
  
  "security": {
    "doc": ["This NDM is not published for use. It is an example for",
            "illustrative purposes only."]
  },

  "table_map": {
    "VID": 0,
    "MAC": 1
  },

  "identifiers": [
    {"var": "<port_vid>",
     "doc": "A VLAN ID to be assigned to untagged or priority tagged frames received on a port."},
    {"var": "<port_priority>",
     "doc": "A priority to be assigned to untagged frames received on a port."},
    {"var": "<local_vid>",
     "doc": "A VLAN ID valid on the wire at a port."},
    {"var": "<relay_vid>",
     "doc": "A VLAN ID valid internal to the switch."},
    {"var": "<VID>",
     "doc": "A VLAN ID"},
    {"var": "<Group_MAC>",
     "doc": "A group (multicast) MAC address."},
    {"var": "<port_no>",
     "doc": "A valid port number on the logical switch."},
    {"var": "<<group_entry_types:name>>",
     "doc": ["An OpenFlow group identifier (integer) identifying a group table entry",
             "of the type indicated by the variable name"]}
  ],
  
  "meter_table": {
    "meter_types": [
      {"name": "ControllerMeterType",
       "bands": [{"type": "DROP", "rate": "1000..10000", "burst": "50..200"}]
      }
    ],
    "built_in_meters": [
      {"name": "ControllerMeter", "meter_id": 1,
        "type": "ControllerMeterType", "bands": [{"rate": 2000, "burst": 75}]}
    ]
  },
  
  "flow_tables": [
    {
      "name": "VID",
      "doc": ["Ingress VID processing table, including:",
              " - accepting untagged and priority tagged frames",
              " - accepting VLAN tagged frames",
              " - ingress VID translation" ],
      "flow_mod_types": [
        {
          "name": "Untagged",
          "priority": 3,
          "doc": "Handle untagged traffic on a port.",
          "match_set": [
            {"field": "IN_PORT"},
            {"field": "VLAN_VID", "mask": "0x1fff", "value": "OFPVID_NONE"}
          ],
          "instruction_set": [
            {"instruction": "APPLY_ACTIONS",
              "actions": [
                {"action": "PUSH_VLAN"},
                {"action": "SET_FIELD", "field": "VLAN_VID", "value": "<port_vid>"},
                {"action": "SET_FIELD", "field": "VLAN_PCP", "value": "<port_priority>"}]},
            {"instruction": "GOTO_TABLE", "table": "MAC"}
          ]
        },
        {
          "name": "Priority-Tagged",
          "priority": 3,
          "doc": "Handle priority tagged traffic on a port.",
          "match_set": [
            {"field": "IN_PORT"},
            {"field": "VLAN_VID", "mask": "0x1fff", "value": "OFPVID_PRESENT"}
          ],
          "instruction_set": [
            {"instruction": "APPLY_ACTIONS",
              "actions": [
                {"action": "SET_FIELD", "field": "VLAN_VID", "value": "<port_vid>"}]},
            {"instruction": "GOTO_TABLE", "table": "MAC"}
          ]
        },
        {
          "name": "VLAN-Tagged",
          "priority": 4,
          "doc": "Used to allow only specific VIDs to ingress at a port.",
          "match_set": [
            {"field": "IN_PORT"},
            {"field": "VLAN_VID", "fix_mask": "0xf000", "fix_value": "0x1000",
               "mask": "0x0fff", "value": "<local_vid>"}
          ],
          "instruction_set": [
            {"instruction": "GOTO_TABLE", "table": "MAC"}
          ]
        },
        {
          "name": "VID-Translate",
          "priority": 4,
          "doc": "Used to translate specific VIDs at ingress at a port.",
          "match_set": [
            {"field": "IN_PORT"},
            {"field": "VLAN_VID", "fix_mask": "0xf000", "fix_value": "0x1000",
             "mask": "0x0fff", "value": "<local_vid>"}
          ],
          "instruction_set": [
            {"instruction": "APPLY_ACTIONS",
              "actions": [
                {"action": "SET_FIELD", "field": "VLAN_VID", "value": "<relay_vid>"}]},
            {"instruction": "GOTO_TABLE", "table": "MAC"}
          ]
        }
      ]
    },
    {
      "name": "MAC",
      "doc": ["MAC forwarding table"],
      "flow_mod_types": [
        {
          "name": "Unicast",
          "priority": 2,
          "doc": "Unicast forwarding entry.",
          "match_set": [
            {"field": "VLAN_VID", "fix_mask": "0x1000", "fix_value": "0x1000",
               "mask": "0x0fff", "value": "<VID>"},
            {"field": "ETH_DST"}
          ],
          "instruction_set": [
            {"instruction": "APPLY_ACTIONS",
              "actions": [
                {"action": "GROUP", "group_id": "<EgressPort>"}
              ]
            }
          ]
        },
        {
          "name": "Multicast",
          "priority": 2,
          "doc": "Multicast forwarding entry.",
          "match_set": [
            {"field": "VLAN_VID", "fix_mask": "0x1000", "fix_value": "0x1000",
               "mask": "0x0fff", "value": "<VID>"},
            {"field": "ETH_DST", "value": "<Group_MAC>"}
          ],
          "instruction_set": [
            {"instruction": "APPLY_ACTIONS",
              "actions": [
                {"action": "GROUP", "group_id": "<Mcast>"}
              ]
            }
          ]
        }
      ],
      "built_in_flow_mods": [
        {
          "name": "MISS-to-Controller",
          "priority": 0,
          "doc": ["Send frames with no forwarding entry to the Controller."],
          "match_set": [],
          "instruction_set": [
            {"instruction": "METER", "meter_name": "ControllerMeter"},
            {"instruction": "APPLY_ACTIONS",
               "actions": [{"action": "OUTPUT", "port": "CONTROLLER"}]
            }
          ]
        }
      ]
    }
  ],

  "group_entry_types": [
    {
      "name": "EgressPort",
      "doc": ["Output to a port, removing VLAN tag if needed.",
              "Entry per port, plus entry per untagged VID per port."],
      "group_type": "INDIRECT",
      "bucket_types": [
        {"name": "OutputTagged",
         "action_list": [{"action": "OUTPUT", "port": "<port_no>"}]
        },
        {"name": "OutputUntagged",
         "action_list": [{"action": "POP_VLAN"},
                         {"action": "OUTPUT", "port": "<port_no>" }]
        },
        {"name": "OutputVIDTranslate",
         "action_list": [{"action": "SET_FIELD", "field": "VLAN_VID", "value": "<local_vid>"},
                         {"action": "OUTPUT", "port": "<port_no>" }]
        }
      ]
    },
    {
      "name": "Mcast",
      "doc": ["Output to all ports in a multicast tree (except IN_PORT).",
              "Entry per group MAC address."],
      "group_type": "ALL",
      "bucket_types": [
        {"name": "MCASTport",
         "action_list": [{"action": "GROUP", "group_id": "<EgressPort>"}]
        }
      ]
    }
  ],

  "parameters": [
    {"name": "MAC::TableSize", "type": "integer"}
  ]
}
```
