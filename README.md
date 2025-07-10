# Computer Aided Dispatch System Design

Emergency service agencies are responsible for responding to urgent and life-threatening situations to protect public safety, health and property. Those responses require sharing critical data between citizens (reporting parties), call takers, dispatchers and first-responders in near real-time.

[Computer Aided Dispatch](what_is_cad.pdf) provides an automated way for emergency service agencies to keep track of incidents, activities, information, tasks, messages and the status of deployed resources. By using a C.A.D. Dispatchers are able to see the big picture, while at the same time maintaining detailed records of plans and actions for future reference. First responders are able to clearly communicate in realtime, while having much better situational awareness of surrounding events.

![Alt](cad-system-context.drawio.png)


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

For more details on personas and their interactions see [What is CAD?](what_is_cad.pdf) 

### Non-Functional

* High availability (99.999%) : This results in 5 minutes of downtime per year. 
* Re-playability - The ability to reprocess past events or inputs deterministically to achieve the same result, recover state or re-derive outputs.
* Auditability - Determine exactly how the current state was reached and which user(s) of the system performed those actions
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

* Meeting needs of emergency agencies outside the US. (for now)
* The communications centers themselves loose power



# Design

The foundation of data representation and flow in our CAD backend system will be events. These events will be used to leverage [event sourcing](https://www.geeksforgeeks.org/system-design/event-sourcing-pattern/), which, by design, gives the system a natural way to replay and audit actions made in the system. First, we'll layout the basic system components and backend api's for creating, viewing, managing and receiving realtime updates for acitve calls for service. At the center for robust real time communication for responders and communication is MQTT. 

![cad excalidraw](https://github.com/user-attachments/assets/f98530ba-6a1f-40e5-9dbd-bad3a21792a0)



### On Consistency

Typically the event sourcing pattern has a read side that is eventually consistent, which does carry a risk in our case. The system is responsible for communicating updates from multiple parties in near-realtime in potentially life-threatening situations so transient errors that delay updates, which omits information, may result in undesirable judgements made by first responders because those judgements were made on the information in the moment. Given our capacity needs we do have options to reasonably leverage event souring for it's replayability and auditing abilities while achieving *strict consistency* for critical read models.

When a command is processed, the associated events are appended to the event stream, and the corresponding read models are updated, all within a single transaction.
This ensures that the read models are strictly consistent with the event stream at all times, avoiding eventual consistency issues. This approach does have limitations,

* First, the solution is limited to event storages that support strong transactional semantics; hence the choice of Postgres here.
* We want to limit this approach only to critical read model updates. The more operations that are bundled together in one transaction the longer it takes to commit.
* Some event sourcing frameworks may not be flexible enough for this strategy


# Alternatives

<table>
  <tr>
    <th align="left"><h3>Option</h3></th>
    <th align="left"><h3>Pros</h3></th>
    <th align="left"><h3>Cons</h3></th>
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
    <td colspan="3"><h4>Live Updates</h4></td>
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
           <li>Experience with such systems can leave first responders with having to "guess" when all the information is in.
           <li>Puts additional strain on the database during peak times.</li>
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
