Heads up -- this repo is an active work in progress!

# Get started with Camunda 8 and Spring Boot

Uses Spring Boot and [the Spring Zeebe SDK](https://docs.camunda.io/docs/apis-tools/spring-zeebe-sdk/getting-started/#add-the-spring-zeebe-sdk-to-your-project) to interact with a local self-managed Camunda installation. The interactions with Zeebe are:

1. Deploy a process model.
2. Initiate a process instance.
3. Handle a service task.

## Implementation

### Implementation notes

- There are commits for each notable step in the guide (inspired by what we do in the C7 getting started guides, [example](https://github.com/camunda/camunda-get-started-spring-boot)). I did not add tags for each step, but can do that if we find that we like this format.

### Prerequisites

- Docker Desktop
- A code editor

### Steps

1. Install Camunda desktop modeler
   from https://camunda.com/download/modeler/
2. Install self-managed
   (from https://github.com/camunda/camunda-platform?tab=readme-ov-file#using-docker-compose)
   1. Confirm launched in Docker Desktop, under `camunda-platform` Container.
   2. Confirm by logging into operate (http://localhost:8081) with `demo`/`demo`.
3. Create a new Spring Boot project
   (from https://start.spring.io/)
   1. Project: Maven
   2. Language: Java
   3. Spring Boot: latest non-SNAPSHOT (currently 3.3.0)
   4. Project Metadata:
      1. Group: `io.camunda.demo`
      2. Artifact: `process_payments`
      3. Name: `Process payments`
      4. Description: `Process payments with Camunda`
      5. Package name: `io.camunda.demo.process_payments`
      6. Packaging: `Jar`
      7. Java: 17
   5. Dependencies: none
   6. "Generate" and download the project
      1. `mvn spring-boot:run` to confirm it builds
      2. `git init` if you'd like to commit milestones along the way.
         1. Add a `.gitignore` file with `target/` to ignore build artifacts.
4. Create a new BPMN diagram
   1. Open Camunda desktop modeler
   2. Create a new diagram
      1. Name: "Process payments"
      2. ID: "process-payments"
   3. Add a start event
      1. Name: "Payment request received"
   4. Add a task
      1. Name: "Prepare transaction"
      2. Type: "script"
      3. Implementation: "FEEL expression"
      4. Script/Result variable: totalWithTax
      5. Script/FEEL expression: `total * 1.1`
   5. Add a task
      1. Name: "Charge credit card"
      2. Type: "Service task"
      3. Task definition/Type: "charge-credit-card"
   6. Add an end event
      1. Name: "Payment executed"
   7. Save file to project location
      1. Location: `src/main/resources`
      2. Name: `process-payments.bpmn`
5. Deploy process to self-managed installation
   1. Click "rocket" icon in Camunda desktop modeler
   2. Deployment name: "process-payments"
   3. Target: "Camunda 8 Self-Managed"
   4. Cluster endpoint: "http://localhost:26500/"
   5. Authentication: "None"
   6. Deploy
   7. Confirm deployment in operate (dashboard shows "Process Payments" process)
6. Run process from Modeler
   1. Click "play" icon in Camunda desktop modeler
   2. Variables: `{"total": 100}`
   3. Start
   4. Confirm process instance in operate
      1. Stuck in "Charge credit card" task, because we aren't handling that service task yet
      2. Confirm variables: `{"total": 100, "totalWithTax": 110}`
7. Implement service task

   1. "Install" Camunda Spring Boot Starter
      (see https://docs.camunda.io/docs/apis-tools/spring-zeebe-sdk/getting-started/#add-the-spring-zeebe-sdk-to-your-project)
      1. Add repository to pom.xml (see docs)
      2. Add dependency to pom.xml (see docs)
   2. Configure Zeebe client

      1. src/main/resources/application.properties

         ```
         zeebe.client.broker.grpcAddress=http://127.0.0.1:26500
         zeebe.client.broker.restAddress=http://127.0.0.1:8080
         zeebe.client.security.plaintext=true
         ```

         _Note:_ these URLs are `http`, not `https`. The docs tell us `https`, but that gets overridden in the SDK by the fact that `plaintext`is true.`http` is a more accurate reflection of your local self-managed environment, which does not have TLS/https configured.

   3. Create a worker

      1. file path: src/main/java/io/camunda/demo/process_payments/ChargeCreditCardWorker.java

         1. Name: `ChargeCreditCardWorker`
         2. Package: `io.camunda.demo.process_payments`
         3. Decorate class with `@Component`
         4. Instantiate a logger: `private final static Logger LOG = LoggerFactory.getLogger(ChargeCreditCardWorker.class);`
         5. Add a method `public Map<String, Double> chargeCreditCard() { }`
         6. Decorate `chargeCreditCard` method with `@JobWorker`: `@JobWorker(type = "charge-credit-card")`
         7. Specify `@Variable(name = "totalWithTax") Double totalWithTax` argument into `chargeCreditCard` method
         8. Implement `chargeCreditCard` method

            1. Log "Charging credit card" and amount
            2. Return a map containing the amount charged (this also marks the task as complete)

            ```java
            // Pretend we're actually charging the credit card here
            LOG.info("charging credit card: {}", totalWithTax);

            return Map.of("amountCharged", totalWithTax);
            ```

      2. Confirm credit card charge
         1. In console output
         2. In operate

8. Start a process instance

   1. Instantiate a static logger in `ProcessPaymentsApplication.java`:

      ```java
      private static final Logger LOG = LoggerFactory.getLogger(ProcessPaymentsApplication.class);
      ```

   2. Declare an autowired `ZeebeClient` in `ProcessPaymentsApplication.java`:

      ```java
      @Autowired
      private ZeebeClient zeebeClient;
      ```

   3. Convert the application to a CommandLineRunner

      1. In `ProcessPaymentsApplication.java`, add `implements CommandLineRunner` to the class declaration: `public class ProcessPaymentsApplication implements CommandLineRunner {`
      2. Implement an overriding `run` method:

         ```java
         @Override
         public void run(final String... args) {
           var processDefinitionKey = "process-payments"; // or whatever the key is
           var event = zeebeClient.newCreateInstanceCommand()
               .bpmnProcessId(processDefinitionKey)
               .latestVersion()
               .variables(Map.of("total", 100))
               .send()
               .join();
           LOG.info(String.format("started a process: %d", event.getProcessInstanceKey()));
         }
         ```

   4. Confirm the process starts by re-running the app
      1. Confirm output in console
      2. Confirm process instance in operate

9. Deploy the process on startup

   1. Add `@Deployment(resources = "classpath:process-payments.bpmn")` to `ProcessPaymentsApplication.java`
   2. Change the tax amount calculated in Camunda Modeler to `total * 1.2`
   3. Restart the app and confirm the updated process is deployed and executed
      1. Confirm in console output
      2. Confirm process instance in operate (Note the new process version when filtering process instances)
