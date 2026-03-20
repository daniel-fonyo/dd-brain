# \[RFC \] \- IPA GetPayments 

| *Metadata (will be indexed/searchable). Do not change rows or columns.* |  |
| :---- | :---- |
| **Author(s):**  | **[Peiqiu Tian](mailto:peiqiu.tian@doordash.com)** |
| **Status:** | Draft |
| **Origin:** |  New |
| **History:** | Drafted:  Opened:  Closed:  |
| **Keywords:**  |  |
| **References:** |  |

**Reviewers (optional)**

| Reviewer | Status | Notes |
| :---- | :---- | :---- |
|  | Not started |  |
|  |  |  |

**Dependencies**  
*Make sure that any team that owns services, API, data related jobs, data consumer, business partner, security, privacy, deployment, ops, infra, or any other team that can be impacted by this RFC should be notified and approve*

| Dependency | Team | DRI | Status | Impact |
| :---- | :---- | :---- | :---- | :---- |
|  |  |  | Not started |  |

*RFC Steps: write “draft” phase simple doc, email [rfc@doordash.com](mailto:rfc@doordash.com), get some comments, write “open” phase real doc, email [rfc@doordash.com](mailto:rfc@doordash.com), get some comments, schedule a review if you want.*

# **What?**

| Briefly explain what you are doing in this project or service.  Have a thesis statement if possible. Does this proposal stabilize or have a higher likelihood of adding more entropy to the platform? Example: Cartman will be a service extracted out of Django to handle all operations to the consumer order cart. |
| :---- |

\[...\]

# **Why?**

| Give some background into the problem, product, etc.  Context is super helpful for understanding why this project matters. If possible, you should bring your purpose back to the end-user with a user story.  Example: (For Dasher microservice initiative) As a Dasher, I want the latest app updates that make my job easier and I don't want anything to break, ever. As an engineer who wants to release killer dasher features and fix bugs ASAP, I need to iterate on the dasher backend as quickly as possible. Breaking out the dasher backend into a microservice allows my team to iterate and deploy independently of everyone else (aka faster). We can fix bugs more quickly and our tests will take 1/10 the time of the monolith. |
| :---- |

\[...\]

## **Goals**

* Give a list of the goals you want to achieve with this product, could be…  
* Business metrics  
* Feature set  
* Uptime requirements  
* Performance/speed

## **Non-goals**

* Things that you have thought about, but are intentionally leaving out  
* Product features that you’re leaving out  
* Uptime/performance requirements that you’re not addressing now

# **Who?**

| Please fill in this as it’s a MANDATORY section. List of the team members working on this List of the teams/people who you specifically want to look at this document, particularly people whose approval is required for you to proceed |
| :---- |

\[...\]

# **When?**

| Please fill in this as it’s a MANDATORY section. Provide a timeline for your project.  List of the teams/people who are responsible for particular deliverables, particularly when they have blocking/critical dependencies, and the relative time those deliverables are expected.  |
| :---- |

\[...\]

# **Design**

| This section will be the majority of your document. Keep things at a high level of abstraction. Try not to discuss things that other teams won’t care about (i.e. implementation details) |
| :---- |

\[...\]

## **Introduction**

Give a short summary (1 \- 2 paragraphs) of your technology strategy to achieve your goals.  Thesis statements are awesome.

## **Architecture (Changes to Existing Services)**

Transaction Processor(TP) GetPayment API is supposed to get payment transaction details from any platform including IPA Taulu, Legacy Doordash CRDB, Wolt and Deliveroo. At this time, we can ignore the Wolt and Deliveroo data as the cross-board client call has not been supported yet.  
Also, each payment transaction is an intentPayment which is also associated with multi gateway payments.

The TP implementations including 2 main parts API explosion and Datahandler:  
The PaymentDataHandler struct includes logic for data retrieval and converting data to IntentPayment model.  
For data retrieval we can leverage on external components and via gRPC call to get the data from external platforms.  
For the data converting we can following below mappings:

The API suppose for V0 only as at this time, there are no usage for V1 endpoints. Also, I think V1 should be involved the Transaction proto variable refactor and definations.  
This `GetTransactionV0` node \+ graph registration

Additions to existing services can still follow these instructions from before, although full domain diagrams as in the previous section will be very much appreciated.

Draw a high-level diagram of the different components of your system.  Show how data and requests flow between the different pieces.

### **Dependencies**

What are your upstream and downstream service dependencies?

### **Database Schema**

If applicable, what changes to the database will you be making? How have you thought about performance? For changes related to payments/currency/etc. please make sure to loop in someone from the BizOps and/or finance teams.

### **Interface**

What is the interface through which clients/services will talk to your system?  Essentially just paste your prototypes here.

### **Jobs**

What pieces of your system will be handled asynchronously?  How much throughput do your jobs need to handle?  Can multiple jobs be run concurrently?  What happens if the machine running your job dies in the middle?

## **Service Level Objectives (SLO)**

Describe the requirements your service needs to provide for its downstream dependencies

### **External vs Internal**

Is your service an internal service (only other services within the DoorDash network will be speaking to it), or does it have external clients (could be our mobile apps, website, or even a third party API)

### **Latency**

What kind of latency requirements does your service need to hit?

### **Expected QPS**

What kind of QPS should your service be able to handle, without any serious modifications

### **Failure**

Detail out some of the ways your service can fail, and what kind of degraded performance you will expect from your service.  

All codes will be unit tested

* When the failing component has recovered, how will your system recover?   
* Are you writing unit tests for your code?   
* Do those unit tests run as part of the CI/CD process?   
* Are you able to quickly rollback changes on your service?   
* Do you have enough E2E test coverage AND sufficient monitoring/alerting for critical functionality to inform you when things aren't working?   
* Security \- do we need audit logging? Do the right people (and only the right people) have access?

### **Wolt compatible**

Are the changes you want to introduce impact Wolt in any way? If yes, make sure that you notified the Wolt team (via [slack channel](https://doordash.enterprise.slack.com/archives/C03U513A55L)) and got their approval (add a dependency to the table at the beginning of this document)

## **I18n/L10n**

* If you **don’t plan to launch** your feature internationally  
  * Do you return text or any other data (e.g. numbers, dates) to the user?  
  * Did you review the [related](https://doordash.atlassian.net/wiki/spaces/Eng/pages/2267154301/L10N+Tools+Libraries) tools?  
  * Which l10n [libraries](https://doordash.atlassian.net/wiki/spaces/Eng/pages/2267154301/L10N+Tools+Libraries#:~:text=p-,libraries,-StringsResources) do you need to make use of?  
  * What sort of guardrails do you want to put in place to make sure content is always localized using these libraries?  
  * Have you looped in i18n team (\#ask-i18n)?   
  * How do you make sure that your feature will be available only in selected regions?  
  * What are the follow-up steps to launch it internationally?  
* If you **plan to launch** your feature internationally (**in addition** to the questions above)  
  * How would you handle different time zones?(UI, reports and other user facing surfaces)  
  * How would you process and operate with different currencies?(UI, reports, billing)  
  * Will your feature require any money flow changes?(revenue accounting, finance, invoicing, mx reporting & transactions)  
    * Invoicing & Tax (Tax Service, Vertex, Invoice platform) changes such as:  
      * Invoice format changes  
      * Taxation and tax breakdown changes  
    * Any billing flow changes required?   
    * Do you need to pass through the [Financial Compliance Review](https://doordash.atlassian.net/wiki/spaces/FinCom/pages/2844066465/Financial+Compliance+Process+Overview) process? Your changes will impact flow of funds / financial data tables / financial metrics. Typical projects:  
      * New service offering  
      * New Charge  
      * New Discount, Credit or Refund  
      * Changes in liability   
      * Change in flow of funds  
      * Changes to tables / fields that store financial information  
      * Changes to invoicing or new invoicing requirements  
    * What is your e2e plan for generating transactions and who will help you verify accounting, billing, invoicing, and reporting behaviors?  
  * Please describe and think about any localization specifics such as:  
    * Terms/documents localization  
    * Phone number format  
    * Bank account formats  
    * Geo-specific names (country/cities)  
    * Local service names (for instance payment processing)  
  * Do you need to integrate with any new service providers not available in all countries we operate? (is this provider available or need to pass through a corp security review or data exchange if any)  
  * Any specific legal requirements regarding PII data processing?(collecting/transferring/processing/storing or even a dataset that can be potentially linked or mapped to such data)  
  * Test plan should include testing in all available regions we operate

## **Fraud**

Make a copy of the [Fraud Checklist](https://docs.google.com/document/d/1SMr1RW8UIC6YERV3FVzB38CnQEfEnHv2nNn7Fk9gV4Y/edit#) \- if your changes involve fraud. Please contact \#ask-fraud and involve the DRI as part of your design review.

## **Alternative Designs**

Any alternative ideas you had, and a short sentence about why you decided not to go with this approach.

