# TSD (Technical Specification Design) Review Rubric

## Expected Sections (checklist)

- [ ] Technical Architecture Diagram
- [ ] Infrastructure Design (compute, storage, network)
- [ ] Database Schema / Data Model
- [ ] API Specifications
- [ ] Security Architecture (encryption, auth, network)
- [ ] Deployment Architecture
- [ ] CI/CD Pipeline Design
- [ ] Monitoring & Alerting Strategy
- [ ] Logging & Observability
- [ ] Backup & Recovery
- [ ] Disaster Recovery Plan
- [ ] Performance Benchmarks / SLAs
- [ ] Capacity Planning
- [ ] Technology Stack Justification
- [ ] Environment Strategy (Dev/Staging/Prod)
- [ ] Infrastructure as Code (IaC) Approach
- [ ] Secret & Credential Management
- [ ] Network Security (VPC, firewall, NAT)
- [ ] Access Control (RBAC, IAM)
- [ ] Compliance Requirements

## Critical Questions

1. Is Infrastructure as Code (IaC) used? Can infrastructure be reproduced?
2. Are all environments (Dev/Staging/Prod) properly isolated?
3. Is there a clear secret rotation and credential management strategy?
4. What is the disaster recovery RTO/RPO?
5. Is the monitoring strategy actionable (not just dashboards)?
6. Are network security boundaries clearly defined?
7. Is there an audit trail for all infrastructure changes?
8. Are there single points of failure?
