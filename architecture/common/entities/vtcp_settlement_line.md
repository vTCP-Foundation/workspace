# Settlement Line
Implements an accounting primitive for the vTCP Network that manages the financial relationship between two interconnected [vTCP Nodes](/architecture/common/entities/vtcp_network_node.md). Comprises several key components:

- **Max Positive Balance:** The predefined maximum transfer limit.
- **Max Negative Balance:** The predefined maximum transfer limit.
- **Line Balance:** Tracks the net financial position between the Node and Contractor.
- **Transaction Log:** Maintains a comprehensive record of all financial transactions processed through the line.

The flow of funds is derived from these components:

- **Max positive flow:** Represents the maximum amount the Contractor can pay to the Node, calculated as:
  _Max positive flow = Max Positive Balance - Current Balance_

- **Max negative flow:** Represents the maximum amount the Node can pay to the Contractor, calculated as:
  _Max negative flow = Max Negative Balance + Current Balance_