# Settlement Line
Implements an accounting primitive for the vTCP Network that manages the financial relationship between two interconnected [vTCP Nodes](/architecture/common/entities/vtcp_network_node.md). Comprises several key components:

- **Incoming Line Capacity:** Represents the maximum amount the Contractor can pay to the Node, calculated as:
  _Incoming Capacity = Total Capacity - Current Debt_
  Where:
  - Total Capacity is the predefined maximum transfer limit
  - Current Debt reflects existing financial obligations

- **Outgoing Line Capacity:** Represents the maximum amount the Node can pay to the Contractor, calculated as:
  _Outgoing Capacity = Total Capacity - Current Receivables_
  Where:
  - Total Capacity is the predefined maximum transfer limit
  - Current Receivables reflect pending payments owed to the Node

- **Line Balance:** Tracks the net financial position between the Node and Contractor.
- **Transaction Log:** Maintains a comprehensive record of all financial transactions processed through the line.