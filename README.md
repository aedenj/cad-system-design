# Computer Aided Dispatch System Design

Emergency service agencies are responsible for responding to urgent and life-threatening situations to protect public safety, health and property. Those responses require sharing critical data between citizens (reporting parties), call takers, dispatchers and first-responders in near real-time.

[Computer Aided Dispatch](what_is_cad.pdf) provides an automated way for emergency service agencies to keep track of incidents, activities, information, tasks, messages and the status of deployed resources. By using a C.A.D. dispatchers are able to see the big picture, while at the same time maintaining detailed records of plans and actions for future reference. First responders are able to clearly communicate in realtime, while having much better situational awareness of surrounding events.

<img width="931" height="436" alt="cad-system-context drawio" src="https://github.com/user-attachments/assets/61613268-1438-4286-99c0-283319fae208" />


## Requirements

### Functional
* Call takers:
  * Can create and manage a Call for Service.
  * Can send specific tasks and requests to Dispatchers based on the information acquired from the RP
* Dispatchers:
  * Manage dispatching units
  * Run internal CAD, RMS, or Department of Justice/CJIS searches
    - System audits searches made in context of a CFS
  * “Pull reports” for Responders – associating a new Incident Report or other kind of report.
* Utility users:
  * Perform administrative tasks in support of other roles and route those results.
    - System audits performed tasks
  *  Enter various information for CFS in various systems.
* Responders:
  - Can update their own statuses via their mobile data computers (MDC) and is reflected in all terminals in real time. 
  - Can self-initiate a CFS
  - Interact with the system via a combination of the emergency radio and software running on their MDCs (Mobile Data Computers) in their units.
* Citizens:
  * Support the ability for a citizen to self-generate a scheduled CFS (e.g. schedule a vacation check while they are out of town).
* Analytics
  - Track operational figures: Incident volume trends, Unit availability, Response time monitoring ...
  - Performance & KPI Reporting: Avg Response Times, Time-to-dispatch, time-to-arrival ..


For more details on personas and their interactions see [What is CAD?](what_is_cad.pdf) 

### Non-Functional

* High availability (99.999%) : This results in 5 minutes of downtime per year. 
* Re-playability - The ability to reprocess past events or inputs deterministically to achieve the same result, recover state or re-derive outputs.
* Auditability - Determine exactly how the current state was reached and which user(s) of the system performed those actions
* Scalablity - Handle growith and peak loads with a linear increase in resoures and cost without performance degradation.
* Latency
  * Broadcasting updates to all units in near-real time will happen < 200ms end-to-end
  * Searches < 1s
* Multi-Tenancy - Our CAD system will be cable of serving multiple tenants (e.g. multiple emergency response orgs) using some amoutn of shared infrastructure. 

**Understanding Capacity**

To understand capacity for a single tenant I looked at a few sources for the number of calls for the largest city in the US, NYC. Here I list two sources,

* New York City’s 911 system fields on average [approximately 33,000 emergency calls](downloads/pdf/psac2_feis/002_executive_summary.pdf) per day, or a total of more than 12 million emergency calls per year.
* NYC’s 911 system is the nation’s largest emergency communications system, receiving around [9 million calls each year.](https://www.nyc.gov/content/oti/pages/press-releases/next-gen-911-on-target-2024-completion)

Peak traffic for a CAD system will include incident creation, unit updates, GPS/AVL events, query volume and user/unit sessions for real time updates. Given the total mentioned above let's attempt to estimate the usage of the system. (A more complete document would do a little more research on actual figures)

<table border="1">
  <thead>
    <tr>
      <th>Area</th>
      <th>Population</th>
      <th>Daily Incidents</th>
      <th>Peak Hour Incidents</th>
      <th>Estimated Field Units</th>
      <th>GPS Updates/sec</th>
      <th>Status Updates/sec</th>
      <th>API Calls/sec (Peak)</th>
      <th>External Queries/sec (Peak)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>NYC</td>
      <td>10,000,000</td>
      <td>35,000 - Assume incident rate of 3.5%</td>
      <td>5,250 - Assume 15% of Daily</td>
      <td>12,000</td>
      <td>2,400</td>
      <td>14</td>
      <td>12</td>
      <td>3</td>
    </tr>
  </tbody>
</table>


### Non-Goals

* How to do deal with long-term event retention and access.
* Extensive security details. We do touch on some basics in the design.
* Observability: We'll use existing infrastructure. (e.g. Prometheus & Grafana)
 

# Design

The central capability of our CAD system is highly available, reliable and auditable realtime communication for responders and communication centers. In order to acheive this our CAD system will be founded on [event streaming](https://www.confluent.io/learn/event-streaming/) . As an architectural pattern event streaming describes data movement as a continuous stream of events and processes them as soon as a change happens. (e.g. a unit status change, new information to provide situational awareness)  

Since events will be the foundation of data representation and flow in our CAD backend implementing [event sourcing](https://www.geeksforgeeks.org/system-design/event-sourcing-pattern/) provides a means to replay and audit actions made in the system. Ensuring accurate replay and audit logs requires an authoritative system of record with strong durability and transactional guarantees.

Finally, event streaming also functions as a data integration or messaging backbone, letting different parts of a system communicate via events in a decoupled, asynchronous way. This trait allows us to decouple the SLO requirements of capabilites like real-time upates and analytics. (e.g. Performance & KPI Reporting)

Now let's fill in the technologies, system components and backend api's that meet our requirements,  

<img width="8177" height="4164" alt="cad excalidraw" src="https://github.com/user-attachments/assets/dcf1a85c-be96-4c65-9247-2eb0a906ffba" />

Let's briefly cover key technology choices that realize our key architectural patterns and some non-functional requirements, 

**Services**<br/><br/>
First, stateless services are responsible for core capabilties: CAD service for incidents, message broadcaster for realtime updates, geo ingestion for responder tracking and integration service for interacting with external systems. The stateless nature of these services will be critical for the availability requirements addressed later. 

**Event Sourcing Store**<br/><br/>
Next, the first stop for all critcals events generated by users as a result of interacting with the system will be Postgres. Postgres offers transactional support, ensuring ACID (Atomicity, Consistency, Isolation, Durability) guarantees. This means that events can be written and read in a transactional manner. This allows the backend to acknowledge receipt and storage. Thus making it the official system of record and ensuring an accurate audit log.  The use of event sourcing calls for an append-only policy this can be enforced by avoiding UPDATE or DELETE statements on event records. Lastly, Postgres is a mature technology with wide adoption and a strong ecosystem of tools and libraries. It provides a range of performance optimization techniques, indexing options, and query optimization capabilities, enabling efficient retrieval and processing of events. 

**Real Time Communication**<br/><br/>
At the center of robust real time communication for responders and communication centers in active incidents is MQTT . MQTT provdies the following advantages

* Supports push-based, real-time updates for thousands of clients simultanously.
* Mobile Network Friendly: MQTT is designed for two-way communication in constrained environments (low bandwidth, high latency, unreliable networks). This matches first responders who may be on cellular or patchy WiFi networks.
* Security: TLS Support, Authentication, Fine-Grained Authorization
* MQTT clients build in offline tolerance features. e.g. messages are buffereds and delivered on reconnect 

**Even Streaming / Message Backbone**<br/><br/>
Finally, Kafka will be the messaging backbone for event delivery to other system capabilities (e.g. realtime communication, analytics, realtime geo location, ...). Specifially we will leverage multi-region clusters, often referred to as a stretch cluster,  which allow us to run a single cluster across multiple regions. Stretch clusters replicate data between datacenters across regional availability zones. You can choose how to replicate data, synchronously or asynchronously, on a per Kafka topic basis. It provides good durability guarantees and makes availabilit and disaster recovery (DR) much easier. The core benefits are,

* Consumers can leverage data locality for reading Kafka data, which means better performance and lower cost
* Ordering of Kafka messages is preserved across datacenters
* Consumer offsets are preserved
* In event of a disaster in a datacenter, new leaders are automatically elected in the other datacenter for the topics configured for synchronous replication, and applications proceed without interruption, achieving very low RTOs and RPO=0 for those topics.

### On Performance

We set a 200ms end-to-end latency target for real time communcation. Many of the technology choices above are known for their strong performance in real time communication. The major outstanding question is what the performance of Debezium for CDC. Based on experience CDC technologies, like Debizum, can run into delays with [WAL](https://www.postgresql.org/docs/current/wal-intro.html) at scale.  Given the relatively low volume of critical events even in the face of multi-tenancy this choice of Debezium is plausible, but of course needs verification.

### On Consistency

Typically the event sourcing pattern has a read side that is eventually consistent, which does carry a risk in our case. The system is responsible for communicating updates from multiple parties in near-realtime in potentially life-threatening situations so transient errors that delay updates, which omits information, may result in undesirable judgements made by first responders because those judgements were made on the information in the moment. Given our capacity needs we do have options to reasonably leverage event souring for it's replayability and auditing abilities while achieving *strict consistency* for critical read models and eventual consistency in other areas deemed less critical. 

When a command is processed, the associated events are appended to the event stream, and the corresponding read models are updated, all within a single transaction.
This ensures that the read models are strictly consistent with the event stream at all times, avoiding eventual consistency issues. This approach does have limitations,

* First, the solution is limited to event storages that support strong transactional semantics; hence the choice of Postgres here.
* We want to limit this approach only to critical read model updates. The more operations that are bundled together in one transaction the longer it takes to commit.
* Some event sourcing frameworks may not be flexible enough for this strategy.

### On Availaility

Our approach to availability within AWS infrastructure is redundancy as breifly mentioned in our design above providing availability in this way requires,

* Services with no local state — they must be stateless, and state should be shared between regions.
* Systems that hold state must have fast a reliable data replication between regions.
* We'll need a global network infrastructure to connect your different regions.
* Avoid synchronous cross-regional calls. 

Our CAD system will be run in AWS. All AWS services specifcied in our design have a minimum  multi-az / single region availability SLA of 99.99%. (e.g. [ECS / Fargate](https://aws.amazon.com/ecs/sla/?did=sla_card&trk=sla_card)). We will be implementing a hot stand by, which is a reduentant and indpendent copy of our system. Hence the 
theoretical best case across regions is defined by,

<h4 align="center"><i>A<sub>total</sub> = 1 − (1 − A)<sup>N</sup></i></h4>

which as 100% minus the product of the regions failure rate, which gives us <i>99.9999% = 100% − (0.1%×0.1%)</i> Additionally, we need to understand the availability of our hard
external dependencies.  In the case of, potentially, hard external dependencies like an RMS we would calculate our availability as

<h4 align="center"><i>A<sub>cad</sub> = Avail<sub>acitve-region</sub> × Avail<sub>RMS</sub> × Avail<sub>EMD</sub>... Avail<sub>external</sub></i></h4>

One of our cruicial assumptions here is that our CAD system services will meet our minimum SLA per region of 99.99% or we may blow the error budget through failover time. Acheiving this will require constant system testing. 

All that said computing a maximum theoretical availability is  likely to produce a rough order of magnitude calculation and by itself is unlikely to be accurate. There are additional dependenices like the network, additional components in the individual services within a region etc. The assumptions and system proposed here gets us around five nines, but will need to be validated against real world numbers. 

Now let's take a look out how we could layout our system in AWS,

<img width="4247" height="3941" alt="availability excalidraw" src="https://github.com/user-attachments/assets/33dae3ef-1813-47cf-8d1e-566539808d66" />


**Zero-Downtime Deployment** 

Blue-green deployment strategies will be used so that new versions of services can be released without taking the system offline. Container orchestration (ECS/EKS) and load balancers will be used to phase traffic gradually to new instances and revert if needed.


### On Security

* Authentication & Authorization: All users must authenticate and are authorized based on role following the principle of least privilege. 
* Encryption: All data is encrypted in transit and at rest. APIs are only accessible over HTTPS/TLS, and internal service calls also use TLS within the VPC. Datastores will use AWS KMS-managed encryption keys so that all sensitive data on disks is encrypted. 
* Network Security: The system is deployed in a private AWS VPC with no direct internet exposure for backend instances. Only a load balancer or API gateway in a DMZ subnet accepts public traffic, and it forwards requests to the private application subnets. Security groups and network ACLs restrict access.

  
# Alternatives

<table>
  <tr>
    <th align="left"><h3>Option</h3></th>
    <th align="left"><h3>Pros</h3></th>
    <th align="left"><h3>Cons</h3></th>
  </tr>
  <tr>
    <td colspan="3" ><h4>Primary Ingestion Point - What should be first stop for the data?</h4></td>
  </tr> 
  <tr>
    <td><b>Kafka</b></td>
    <td>
        <ul>
            <li>It's a durable store that supports transactions.</li>
            <li>Battle tested for resilience</li>
        </ul>
    </td>
    <td>
        <ul>
           <li>Not easily queryable.</li>
           <li>All queries for current state will hit an eventual consistency problem. (e.g. officer self-initiates an incident right before dispatch assignment)</li>
        </ul>
    </td>
  </tr>
  <tr>
    <td colspan="3"><h4>Event Stores</h4></td>
  </tr>
  <tr>
    <td><b>KurrentDB</b></td>
    <td>
        <ul>
            <li>Honestly, it appears ideal on paper. It's a native event store.</li>
            <li>Handle projections or event-handling logic within the database itself. (Can be simulated in Postgres)</li>
            <li>Provides features for event versioning, enabling compatibility and evolution of event schemas over time.</li>
            <li>Natively emit events to external sinks. (e.g. Kafka)
            <li>Built-in features for event publishing and event subscription mechanisms, facilitating event-driven communication and integration within an application or across microservices,</li>
        </ul>
    </td>
    <td>
        <ul>
           <li>Not as mature and battle tested as Postgres</li>
           <li>Potentially less mature tooling ecosystem</li>
        </ul>
    </td>
  </tr>
  <tr>
    <td colspan="3"><h4>Consistency</h4></td>
  </tr>
  <tr>
    <td><b>On-demand reads based on event stream</b></td>
    <td>
        <ul>
            <li>Read models are derived from the most up to date state</li>
        </ul>
    </td>
    <td>
        <ul>
           <li>Limited to the cases where queries span either a single or only a few aggregates</li>
           <li>Event streams may be too long?</li>
        </ul>
    </td>
  </tr>
   <tr>
    <td><b>Eventual Consistency Everywhere</b></td>
    <td>
        <ul>
            <li>Isolating reads completely benefits performance, availability and technologies used, ending with the ability of building an unlimited number of read models dedicated to specific tasks with dedicated storage.</li>
        </ul>
    </td>
    <td>
        <ul>
           <li>Complexity of implementation and maintence.</li>
           <li>Can result in undesirable actions in the systems. Assignment of units not actually available.</li>
           <li>Potential need for more compensating actions in the system due to lack of current state.</li>
        </ul>
    </td>
  </tr>
  <tr>
    <td colspan="3"><h4>Live Updates for Responders</h4></td>
  </tr>
  <tr>
    <td><b>Polling</b></td>
    <td>
        <ul>
            <li>It's simple, the server would then returns updates that were posted after the since the last one</li>
        </ul>
    </td>
    <td>
        <ul>
           <li>Not real-time, inefficient for battery and bandwidth.</li>
           <li>Puts additional strain on the database during peak times.</li>
           <li>Experience with such systems can leave first responders with having to "guess" when all the information is in.</li>         
        </ul>
    </td>
  </tr>
  <tr>
    <td><b>Web Sockets or Server Side Events</b></td>
    <td>
        <ul>
            <li>Plenty of Pro's for other use cases</li>
        </ul>
    </td>
    <td>
        <ul>
           <li>MQTT is just tailored made for the use case so beats it out. </li>
           <li>No built-in delivery guarantees</li>
           <li>No offline support ... so many more</li>           
        </ul>
    </td>
  </tr>
</table>


# References

* PSAC II Project Executive Summary by the NYC Mayor’s Office, July 2009, https://www.nyc.gov/html/nypd/downloads/pdf/psac2_feis/002_executive_summary.pdf
* NYC's Next Generation 911 on Target for 2024 Completion, Dec 2022, https://www.nyc.gov/content/oti/pages/press-releases/next-gen-911-on-target-2024-completion
* Event Streaming: How it Works, Benefits, and Use Cases, https://www.confluent.io/learn/event-streaming/#what-is-an-event-stream
* Understanding Eventsourcing, Nov 2024, https://leanpub.com/eventmodeling-and-eventsourcing
* Availability and Beyond, https://docs.aws.amazon.com/whitepapers/latest/availability-and-beyond-improving-resilience/availability-and-beyond-improving-resilience.html
