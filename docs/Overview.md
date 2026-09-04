# Flow Connection Phase 2 Requirements and Gaps

_(c) AMWA 2026, CC Attribution-NoDerivatives 4.0 International (CC BY-ND 4.0)_

# MXL Domains

- An MXL domain is a folder in a tmpfs backed volume or filesystem that contains flows and grains (see [MXL Architecture](https://github.com/dmf-mxl/mxl/blob/main/docs/Architecture.md)). 
- Multiple MXL domains can co-exist on the same host but each MXL domain exists only on one host (because it is a unique folder in the individual file system. The same path could exist on another host, but it would still be a different domain). 
- One or more MXL domains on a host can be mapped into the containers that host the media functions during deployment. 
- The path where the MXL domain is mapped need not be the same across different media function containers.
- To be able to uniquely identify a domain, each domain contains a file “domain_def.json” (see AMWA BCP-007-03: NMOS With MXL | bcp-007-03) which needs to be created by the entity creating the MXL Domain (folder).
- To be able to uniquely identify a domain, each domain contains a file “domain\_def.json” (see [AMWA BCP-007-03: NMOS With MXL](https://specs.amwa.tv/bcp-007-03/)) which needs to be created by the entity creating the MXL Domain (folder).

## Usage

- It is recommended to keep the MXL domain structure as simple as possible.
- One MXL domain per host is a good starting point, if it meets the requirements.
- Multiple MXL domains can be used for grouping flows for access control, e.g.  
  - Flows internal to a PCR in one MXL domain, flows external in another MXL domain. 
  - Different MXL domains for different productions.
- Access control can be enforced by mapping only those MXL domains into a container that the media function needs access to or by means of UNIX file system permissions which give more granular control. 

## Realm

- Since MXLDomains cannot span more than one host, an orchestration/control system may chose to establish a Realm that is a logical grouping of MXLDomain(s) (e.g. with similar access rights) possibly on different hosts. This does not imply replication of all the flows in the MXLDomain.

# MXL Flows

- Media functions may need to create additional or new flows during runtime for a variety of reasons:  
  - Reconfiguration  
  - Change of input signal (for a gateway media function)

  If that happens, the orchestration/control system should be notified of the new flows.

- If the technical parameters of a flow change, the existing flow needs to be removed and a new flow (with a different flowID) needs to be created. If that is the case, the Media Function should notify the orchestration/control system that the writer is now writing a different flowID so that it can decide on the necessary actions:  
  - inform the readers that were reading the original flow to now read the new flow  
  - change the setup of the replication to now replicate the new flow instead of the original flow  
- When a writer shuts down, it needs to remove the flow folder and all its contents from the file system. Readers need to be able to cope with flows disappearing.  
- To prevent writers from deleting data that has not been read yet by readers downstream, they wait for at least one length of the ring buffer after having written the last data before shutting down and deleting the ring buffer.  
- The MXL SDK provides a function to clean the filesystem of any flow directories left over from writers that have not exited gracefully. This function needs to be called either periodically (e.g. a cron job) or when the need arises (e.g. through a watchdog) depending on the requirements of the concrete system.

## Connecting and Disconnecting Flows

- NMOS IS-05 ([AMWA BCP-007-03: NMOS With MXL](https://specs.amwa.tv/bcp-007-03/) ) or other protocols following the flow connection API requirements specification can be used to connect and disconnect flows at runtime.  
- To be able to create DMFs with only static connections that do not require a separate realtime control system, media functions should provide a method to load a configuration file with the following yaml syntax when the container is deployed. 

```yaml 
senders:  
  1b99dffd-5412-4c69-9758-f7aee1143ad5:  
    master_enable: true  
    transport_params:  
      - mxl_domain_id: 0cb2af47-0e9e-47dc-8633-c451b353d02f  
        mxl_flow_id: 83df0a80-92d1-489d-a82b-077e2cef74cd

  ecaf4b73-c8bf-434a-902e-a55faf7f79e6:  
    master_enable: true  
    transport_params:  
      - mxl_domain_id: auto  
        mxl_flow_id: auto

receivers:  
  9c46ca50-f0ef-420a-949c-d490f76a9e9f:  
    master_enable: true  
    sender_id: bd156ddf-662e-40a7-9caa-b4bfe26c78a3  
    transport_params:  
      - mxl_domain_id: d9becfc9-c435-4bf7-abcc-59eed9a17028  
        mxl_flow_id: a5c377de-ae50-4e72-a1ab-b715fb429b89

  57ab0f6a-2776-49b1-86e7-064c45d24c2b:  
    master_enable: true  
    sender_id: 83e0aee2-c150-469f-bfe0-81f5327d61e8  
    transport_params:  
      - mxl_domain_id: auto  
        mxl_flow_id: a5c377de-ae50-4e72-a1ab-b715fb429b89
```

## Replication

Replication designates the process of copying the flow data from one domain to another.

- The MXL SDK provides functions for this, however each domain needs a process that executes the necessary functions in the SDK.  
- Replication can be setup manually or automatically.  
- Replication can be setup statically (at deployment-time) or dynamically (on request during runtime)  
- Whoever sets up the replication is responsible for tearing it down when it is no longer needed (e.g. reader stops reading, writer exits, writer crashes etc.)  
- In many cases it might be preferable to delay the tear-down of replication in case the media function comes back after a short period of time, so the process that tears down the replication should do so only after a configurable delay.

## Between-Host Bandwidth Management

- While inside one host or node bandwidth is considered unlimited, when replicating flows between hosts, the entity configuring the replication needs to take into account the limited bandwidth that is available for the replication.  
- For sake of simplicity, we assume at this point that the replication is managed centrally for each node and not by each media function individually.  
- Flow Bandwidth: The entity controlling the replication needs to be aware of the bandwidth required by each flow that needs to be replicated. Information about calculating the bandwidth is available in the MXL SDK.  
- Network Bandwidth: It still needs to be determined how the entity controlling the replication get information about the available bandwidth.  
- If there is not enough bandwidth available for a replication request the user who initiated the request needs to be notified. The means are still to be discussed.


# Media Functions

- Media Functions must be able to run in non-privileged containers.  
- Media Functions should be run in non-privileged containers.

## Monitoring

- A media function should provide active notifications of the following states for readers:  
  - Grain cannot be decoded  
  - Grain cannot be read  
  - Grain is available (in time)  
- A media function should provide active notifications of the following states for writers:  
  - Still to be discussed  
- Protocols a media function could use to provide those notifications could be:  
  - NMOS IS-12  
  - OpenTelemetry  
  - ST 2138

# Example

To be added later.

