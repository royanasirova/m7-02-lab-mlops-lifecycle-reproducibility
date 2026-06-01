# ETA Model Lifecycle Diagram

```mermaid
graph TD
    %% Define Stages / States
    subgraph Data Layer
        A[Raw Data Ingestion]
    end

    subgraph Experimentation & Training
        B[Experiment Tracking Sandbox]
        C[Automated Training Pipeline]
    end

    subgraph Evaluation
        D[Automated Evaluation Gate]
    end

    subgraph Model Registry
        E((Stage: None))
        F((Stage: Staging))
        G((Stage: Production))
        H((Stage: Archived))
    end

    subgraph Deployment Layer
        I[Canary Deployment]
        J[Full Production Live]
    end

    subgraph Monitoring Layer
        K[Real-time Performance & Drift Monitor]
    end

    %% Flow and Transitions with Artifact Contracts
    A -->|1. Immutable Dataset snapshot: content_hash| B
    A -->|2. Pinned Dataset: content_hash| C
    
    B -->|3. Optimal Hyperparams & Git SHA| C
    
    C -->|4. Unvalidated Model Binary: run_id + model_uri| D
    
    D -->|5. Passes Baseline: evaluation_metrics.json| E
    
    E -->|6. AUTOMATIC: Auto-promotion if Eval metrics pass gates| F
    
    F -->|7. MANUAL APPROVAL: Requires ML Lead Sign-off after Canary testing| G
    
    G -->|8. AUTOMATIC: Rollback triggered or newer model deployed| H

    G -->|9. Deployed Active Version: deployed-vX.Y.Z| I
    I -->|10. 10% Traffic Test: canary_metrics.json| J
    J -->|11. Live Production Traffic Inference| K

    %% Monitoring Feedback Loops
    K -->|12. CRITICAL DRIFT SIGNAL: Triggers automated retraining pipeline| C
    K -->|13. DATA QUALITY FEEDBACK: Flags anomalies for new dataset curation| A

    %% Styling
    classDef stage fill:#f9f,stroke:#333,stroke-width:2px;
    classDef artifact fill:#bbf,stroke:#333,stroke-width:1px;
    class F,G,H stage;