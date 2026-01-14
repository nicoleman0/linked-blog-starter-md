**Stack refinements:**

1. **FastAPI vs Flask**: Go with FastAPI. You get async support out of the box (useful for long-running scans), automatic API documentation, and better type safety with Pydantic. Flask would work but you'd be building those features yourself.
2. **Database**: SQLite is fine for a single-VM deployment, but consider PostgreSQL if you anticipate:
    - Multiple concurrent users running scans
    - Large scan result datasets
    - Need for better concurrent write handlingSQLite will lock the entire database on writes, which could be problematic during active scans.
3. **Additional components to consider:**
    - **Redis or similar**: Queue management for scan jobs (Celery + Redis is a common pattern)
    - **Docker**: Even within your VM, containerizing ZAP and your application separately gives you better isolation and easier updates
    - **Authentication layer**: If this is organizational, you'll need proper auth (consider integrating with your org's SSO/LDAP)

- FastAPI (Web interface)
- Python (automation scripts)
- PostgreSQL / SQLite* for prototype maybe (database)
- Redis (Queue management)

### Implementation Plan

**Phase 1: Core functionality (Week 1-2)**

- Set up VM with ZAP installed and running in daemon mode
- Create basic Python scripts using ZAP API to:
    - Start/stop ZAP
    - Run spider against target
    - Run active scan
    - Retrieve results
- Test these scripts manually to understand ZAP's API behaviour and timing

**Phase 2: Database schema (Week 2)**

- Design schema for:
    - Scan configurations (targets, scan policies, schedules)
    - Scan jobs (status, timestamps, parameters)
    - Scan results (alerts, vulnerabilities, raw data)
    - Users/teams (who initiated scans)
- Consider storing raw ZAP XML/JSON reports alongside parsed data

**Phase 3: Backend API (Week 3-4)**

- Build FastAPI endpoints:
    - `POST /scans` - Initiate new scan
    - `GET /scans/{id}` - Get scan status/results
    - `GET /scans` - List scans with filters
    - `DELETE /scans/{id}` - Cancel running scan
    - `GET /reports/{id}` - Generate reports (PDF/HTML/JSON)
- Implement async task handling for scans (they can take hours)
- Add proper error handling and logging

**Phase 4: Job queue (Week 4)**

- Integrate Celery + Redis for background task processing
- This prevents API timeouts and allows scan scheduling
- Enables scan queuing if ZAP can't handle concurrent scans

**Phase 5: Frontend (Week 5-6)**

- Simple web UI for:
    - Submitting scan requests (URL, scan type, policy)
    - Viewing scan queue and history
    - Displaying results with severity filtering
    - Exporting reports
- Could use React/Vue or just server-side templates with Jinja2 for simplicity

**Phase 6: Hardening (Week 6-7)**

- Authentication and authorization
- Input validation (prevent scanning unauthorized targets)
- Rate limiting
- Audit logging
- Secure credential storage for authenticated scans
- Network isolation considerations (can the scanner reach production?)

**Phase 7: Integration & Polish (Week 7-8)**

- Integration with ticketing system (Jira/ServiceNow) for vulnerability tracking
- Scheduled scans (cron or Celery beat)
- Alerting for critical findings (email/Slack)
- Documentation and runbooks

**Critical security considerations:**

1. **Target authorization**: Implement strict controls on what can be scanned. OWASP ZAP can be destructive, especially active scans.
2. **Network segmentation**: This VM should probably be in a dedicated security tools network segment with carefully controlled egress.
3. **Credential management**: If you're doing authenticated scans, use a secrets manager (HashiCorp Vault, AWS Secrets Manager, etc.)
4. **Rate limiting and scope**: Prevent accidental DoS of your own infrastructure.
5. **Data retention**: Scan results may contain sensitive data. Plan for encryption at rest and proper retention policies.

**Quick prototype approach:**

If you need a working prototype quickly:

1. Start with just ZAP + Python scripts (no web interface)
2. Use argparse for CLI parameters
3. Store results as JSON files initially
4. Once scanning logic is proven, add FastAPI + SQLite
5. Progressively add features