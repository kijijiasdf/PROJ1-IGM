Hi 

We’ve reviewed the December IAM Blueprint and Detailed Design and are finalizing the implementation scope for our EIAM team.

Before progressing, could we confirm these points?

  1. What is already implemented in Dev, and which changes are expected from EIAM versus AI Platform or other teams?

  2. What is the final gateway group-assignment and claims approach? The Blueprint and LLD appear to differ on assigning the FAQ groups alongside the internal-user group.

  3. What is the test-client type and approved authentication configuration, including whether certificates are permitted if it is a confidential client?

  4. Is Tools.Invoke the agreed tool-gateway authorization model, and who performs the privileged assignment to the agent identity?

  5. What is EIAM’s containment responsibility, who owns the APIM changes, and is the identity-disablement exercise included for December?

  6. What is the required December readiness level and date, and which design version should we use as the approved baseline?

We have a detailed clarification tracker available, but these are the main points we need first to establish the EIAM implementation activities. We can route the remaining configuration and validation details to the appropriate engineering owners.