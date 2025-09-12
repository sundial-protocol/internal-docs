# Disaster Recovery Plan

In the event of a catastrophic failure or security breach, Sundial has a comprehensive disaster recovery plan to ensure the preservation of all user funds and the recovery of the L2 network. 

## 1. Immediate Response

Upon detection of a failure or breach, the following immediate actions will be taken:

- **Incident Assessment:** Quickly assess the nature and scope of the incident to understand its impact on the network and user funds.
- **User Notification:** Inform affected users about the incident, providing clear instructions on any actions they need to take.
- **System Isolation:** If necessary, isolate affected components of the network to prevent further damage or unauthorized access.

## 2. Data & Asset Preservation

To protect user funds and data integrity, Sundial will implement the following measures:

- **State Snapshots:** Regularly take snapshots of the L2 network state, including user balances and transaction histories, to enable recovery in case of data loss.
- **Secure Backups:** Maintain secure, off-site backups of all critical data, ensuring that it can be restored in the event of a catastrophic failure.

## 3. Recovery Procedures

Once the immediate threat has been mitigated, Sundial will initiate recovery procedures:

- **System Restoration:** Restore the L2 network from the most recent state snapshot, and that all bridged assets have been recovered on the L2 or reclaimed on the L1, ensuring that all user funds & data is preserved.
- **Security Audit:** Conduct a thorough security audit to identify and address any vulnerabilities that may have contributed to the incident.
- **User Compensation:** If user funds were lost or compromised, implement a compensation plan to restore affected users' balances.

## 4. Post-Incident Review

After the recovery process is complete, Sundial will conduct a post-incident review to identify lessons learned and improve future response efforts:

- **Root Cause Analysis:** Investigate the underlying causes of the incident and document findings.
- **Policy Updates:** Update disaster recovery policies and procedures based on the insights gained from the incident.
- **User Communication:** Provide a transparent report to users detailing the incident, response actions taken, and any changes made to improve security.

By implementing this disaster recovery plan, Sundial aims to ensure the resilience of the L2 network and the protection of user funds in the face of unforeseen challenges.

## Worst Case Scenario
In the worst-case scenario where the L2 network is irreparably compromised, Sundial has a contingency plan to ensure that all user funds can be safely reclaimed on the associated L1s. This plan includes:
  - **Cardano Recovery (Escape Hatch)** When a sufficient amount of time has passed without a block produced, users can initiate an escape hatch procedure to withdraw all funds from the L2 back to the Cardano L1.
  - **Bitcoin Recovery (HTLC Expiry | Committee Member Assistance)** In the event of a catastrophic failure of the L2 network, users can reclaim their bridged Bitcoin assets directly on the Bitcoin L1 using the bridge's built-in recovery mechanisms. These involve 2 potential methods:
    - **HTLC Expiry** allow the HTLC used to lock Bitcoin assets during the bridging process to expire, enabling the user to reclaim their assets after the predetermined period.
    - **Committee Member Assistance** A user needs only one of the committee members to assist them in reclaiming their Bitcoin assets ahead of the HTLC expiry. This is done by providing a signed transaction that returns the assets to the user's specified address.
