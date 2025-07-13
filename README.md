# Computer Aided Dispatch System Design

Emergency service agencies are responsible for responding to urgent and life-threatening situations to protect public safety, health and property. Those responses require sharing critical data between citizens (reporting parties), call takers, dispatchers and first-responders in near real-time.

[Computer Aided Dispatch](what_is_cad.pdf) provides an automated way for emergency service agencies to keep track of incidents, activities, information, tasks, messages and the status of deployed resources. By using a C.A.D. Dispatchers are able to see the big picture, while at the same time maintaining detailed records of plans and actions for future reference. First responders are able to clearly communicate in realtime, while having much better situational awareness of surrounding events.



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
 

**Understanding Capacity**

To understand capacity I looked at a few sources for the number of calls for the largest city in the US, NYC. Here I list two sources,

* New York City’s 911 system fields on average [approximately 33,000 emergency calls](downloads/pdf/psac2_feis/002_executive_summary.pdf), or a total of more than 12 million emergency calls per year.
* NYC’s 911 system is the nation’s largest emergency communications system, receiving around [9 million calls each year.](https://www.nyc.gov/content/oti/pages/press-releases/next-gen-911-on-target-2024-completion)

Peak traffic for a CAD system will include incident creation, unit updates, GPS/AVL events, query volume and user/unit sessions for real time updates. Given the total mentioned above let's attempt to estimate the 
usage of the system. (A more complete document would do a little more research on actual figures)

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

* How do deal with long-term event retention and access.


# Design

The central capability of our CAD system is highly available, reliable and auditable realtime communication for responders and communication centers. In order to acheive this our CAD system will be founded on [event streaming](https://www.confluent.io/learn/event-streaming/) . As an architectural pattern event streaming describes data movement as a continuous stream of events and processes them as soon as a change happens. (e.g. a unit status change, new information to provide situational awareness)  

Since events will be the foundation of data representation and flow in our CAD backend implementing [event sourcing](https://www.geeksforgeeks.org/system-design/event-sourcing-pattern/) provides a means to replay and audit actions made in the system. Ensuring accurate replay and audit logs requires an authoritative system of record with strong durability and transactional guarantees.

Finally, event streaming also functions as a data integration or messaging backbone, letting different parts of a system communicate via events in a decoupled, asynchronous way. This trait allows us to decouple the SLA requirements of supporting capabilites like analytics. (e.g. Performance & KPI Reporting)

Let's layout the technologies, basic system components and backend api's that meet our functional and performance requirements for creating, viewing, managing and receiving realtime updates for acitve incidents, 


<img width="6574" height="3802" alt="cad excalidraw" src="https://github.com/user-attachments/assets/5da78415-ad43-4766-8b3a-7446b95f8988" />


At the center for robust real time communication for responders and communication centers in active incidents is MQTT . MQTT provdies the following advantages

* Supports push-based, real-time updates for thousands of clients simultanously.
* Mobile Network Friendly: MQTT is designed for two-way communication in constrained environments (low bandwidth, high latency, unreliable networks). This matches first responders who may be on cellular or patchy WiFi networks.
* Security: TLS Support, Authentication, Fine-Grained Authorization
* MQTT clients build in offline tolerance features. e.g. buffer messages and deliver on reconnect 





### On Consistency

Typically the event sourcing pattern has a read side that is eventually consistent, which does carry a risk in our case. The system is responsible for communicating updates from multiple parties in near-realtime in potentially life-threatening situations so transient errors that delay updates, which omits information, may result in undesirable judgements made by first responders because those judgements were made on the information in the moment. Given our capacity needs we do have options to reasonably leverage event souring for it's replayability and auditing abilities while achieving *strict consistency* for critical read models.

When a command is processed, the associated events are appended to the event stream, and the corresponding read models are updated, all within a single transaction.
This ensures that the read models are strictly consistent with the event stream at all times, avoiding eventual consistency issues. This approach does have limitations,

* First, the solution is limited to event storages that support strong transactional semantics; hence the choice of Postgres here.
* We want to limit this approach only to critical read model updates. The more operations that are bundled together in one transaction the longer it takes to commit.
* Some event sourcing frameworks may not be flexible enough for this strategy

#### On Availaility


**Zero-Downtime Deployment** 

Blue-green deployment strategies will be used so that new versions of services can be released without taking the system offline. Container orchestration (ECS/EKS) and load balancers will be used to phase traffic gradually to new instances and revert if needed.

# Alternatives

<table>
  <tr>
    <th align="left"><h3>Option</h3></th>
    <th align="left"><h3>Pros</h3></th>
    <th align="left"><h3>Cons</h3></th>
  </tr>
  <tr>
    <td colspan="3"><h4>Event Stores</h4></td>
  </tr>
  <tr>
    <td><b>KurrentDB</b></td>
    <td>
        <ul>
            <li>Honestly, it appears ideal on paper. It's a native event store.</li>
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
