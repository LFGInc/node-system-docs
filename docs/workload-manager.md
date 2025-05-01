# Workload Manager

[Node system workload manager](https://github.com/LFGInc/node-system-workload-management) is a specific workload runs on user's node, inside the K3S cluster. It is built from [Express JS](https://expressjs.com/) with minimal codebase and dependencies. Workload manager is responsible for:

- Workload version control
- Uptime management
- Health check and reporting
- Self-update
- Sync the node system state with configuration saved in the backend

## Workflow

```mermaid
sequenceDiagram
    participant Workload
    participant WM as Workload Manager
    participant K3S as K3S Cluster
    participant Backend as Node System Backend

    par 
        loop Every 5-10 seconds
            Workload->>WM: Send health status
        end

        loop Every minute
            WM->>Backend: Report node health with all workload status
        end
    end

    loop Every 10 minutes
        WM->>Backend: Fetch latest workloads and their versions
        Backend-->>WM: Return workload versions

        WM->>K3S: List current deployments
        K3S-->>WM: Return deployment list

        loop For each deployment
            alt Deployment exists but workload not found
                WM->>K3S: Delete deployment
            else
                WM->>Backend: Fetch new deployment manifest
                Backend-->>WM: Return manifest

                alt Deployment exists and workload is outdated
                    WM->>K3S: Patch deployment with new manifest
                    WM->>Backend: Update workload version
                else Deployment does not exist
                    WM->>K3S: Create new deployment
                end
            end
        end
    end

    loop Self-update
        WM->>AWS S3: Get latest version in file
        AWS S3-->>WM: Return latest version
        
        WM->>K3S: Get current workload manager deployment
        K3S-->>WM: Return current deployment
        alt Current version is outdated
            WM->>K3S: Patch deployment with new image
        end
    end
```