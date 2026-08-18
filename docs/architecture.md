# Architecture
(Almost) Stable infrastructure for MediaWiki service

## Overview
4 Virtual Machines - Ubuntu-based
## Components
VM number 1 - Nginx load-balancing + Zabbix monitoring server
VM number 2 and 3 - MediaWiki+PostgreSQL
VM number 4 - backup data storage
## Data Flow
Client -> Internet -> Nginx -> MediaWiki -> PostgreSQL
PostgreSQL primary/standby replication using WAL
## Backup
Automated backups for service filesystem and database using bash scripts
## Monitoring
Zabbix availability monitoring via HTTP code and response time

## Architecture Diagram
```mermaid
flowchart TD
	A[User]

	subgraph VM-1
		B[Zabbix Server] -.-> C[Nginx Load Balancer]
	end

	subgraph VM-2
		D[MediaWiki] --> E[(PostgreSQL Primary)]
	end
	
	subgraph VM-3
		F[MediaWiki] --> E  
		G[(PostgreSQL Standby)]
	end
	
	subgraph VM-4
		H[(Backup Storage)]
	end
	
	A --> C
	C --> D
	C --> F
	E --> |WAL replication|G
	D -.->|backup| H
	E -.->|backup| H
	F -.->|backup| H
```
