# Lab: Build a process using SAGA pattern!

In distributed systems, maintaining consistency across services can be challenging, especially when two-phase commits are impractical. The SAGA pattern provides a solution by breaking a process into isolated transactions with compensatory actions in case of failure. This lab focuses on a data-driven flow approach, where the process flow is controlled by the response data from each service task, avoiding the use of exceptions or error events.

Within this approach, gateways decide the next step based on the content of the response data. For instance, if a service task returns an error response, the compensatory flow is triggered via an exclusive gateway. The process is designed to use inclusive gateways to manage centralized handling tasks, such as marking the order as failed or completed.

By the end of this lab, you will have modeled an order process with key steps and compensatory actions that simulate the SAGA pattern.

## Lab Overview

You will create a BPMN process that follows the SAGA pattern, structured with service tasks and gateways. The process will include three main steps:

1. Reserve Product – Reserve stock for the order.
1. Process Payment – Process the payment.
1. Confirm Order – Final confirmation task.

If any step fails, the process will trigger compensatory actions to undo the completed steps. The compensation flow includes:

- Cancel Product Reservation
- Cancel Payment

The final step, if compensation is needed, will be a task indicating that the Order Failed.

### Supporting Files

The Java services provided in the repository https://github.com/aletyx-labs/kie-10.0.0-lab-saga contain the infrastructure to support this process. Below is a brief description of each file:

#### StockService.java

This service handles the reservation and cancellation of product stock.

- reserveStock(): Attempts to reserve stock and uses the MockService to simulate success or failure.
- cancelStock(): Cancels the reserved stock if a failure occurs.


#### PaymentService.java

Responsible for managing payment operations, including processing and cancellation.

- processPayment(): Processes the payment for the order. The result is simulated through the MockService.
- cancelPayment(): Cancels the payment if the process fails.


#### OrderService.java

This service provides the final status update of the order.

- success(): Logs and returns a success response when the order is completed without errors.
- failure(): Logs and returns an error response when the order process fails.


#### Response.java

Defines a response object that all services use to indicate the result of their execution.

- Type: Enum representing SUCCESS or ERROR responses.
- Utility methods like success(), error(), and isSuccess() simplify response management.


#### MockService.java

Simulates service execution and determines success or failure based on configuration.

- execute(): Checks if the current service should fail based on the failClass parameter. It returns either a SUCCESS or ERROR response without throwing exceptions. This supports the data-driven flow approach by allowing gateways to handle decisions based on response data.

### Instructions

#### Step 1: Clone the Repository

Clone the lab project from this repository, which contains all the necessary files and infrastructure for the lab.

#### Step 2: Import the Project

Import the project into VS Code and explore the provided Java classes.

#### Step 3: Create the BPMN file

Create a new file name order.bpmn under src/main/resources and open it using the Apache KIE BPMN Editor.

Once file is created, create the process variables that we'll need.

| Name             | Data Type                               |
|-----------------|----------------------------------------|
| orderId        | String                                 |
| failService    | String                                 |
| stockResponse  | ai.aletyx.kie.workshop.saga.Response      |
| paymentResponse | ai.aletyx.kie.workshop.saga.Response   |
| orderResponse  | ai.aletyx.kie.workshop.saga.Response      |

#### Step 4: Create the process diagram

As mentioned before, the process will be data-flow based, so on every operation we'll have the results of the execution to check if it worked or not. In case of failure, we'll have the failure path and the compensations are expected to be executed.

In general this is the nodes you'll have to create:

1. Start Event: Initiates the order process.
2. Reserve Product Task

     Interface: ai.aletyx.kie.workshop.saga.StockService

     Operation: reserveStock

     Variables - Input Mapping:

     | Name   | Data Type | Source     |
     |--------|----------|------------|
     | param1 | String   | orderId    |
     | param2 | String   | failService |

     Variables - Output Mapping:

     | Name   | Data Type                                | Target        |
     |--------|-----------------------------------------|--------------|
     | result | Response [ai.aletyx.kie.workshop.saga] | stockResponse |

3. Exclusive Gateway: We'll check if the Reserve was successfull or not.
    - On Fail

        ```java
        return !stockResponse.isSuccess();
        ```

        3.1. Create an Inlcusive Gateway to consolidate actions

        3.2. Create a Order Failed Task

          Interface: ai.aletyx.kie.workshop.saga.OrderService

          Operation: failure

          Variables - Input Mapping:

          | Name   | Data Type | Source     |
          |--------|----------|------------|
          | param1 | String   | orderId    |

          Variables - Output Mapping:

          | Name   | Data Type                                | Target        |
          |--------|-----------------------------------------|--------------|
          | result | Response [ai.aletyx.kie.workshop.saga] | orderResponse |

        3.3. Create an End Compensation Node

    - On Success:

        ```java
        return stockResponse.isSuccess();
        ```

        3.4. Process Payment Task

           Interface: ai.aletyx.kie.workshop.saga.PaymentService

           Operation: processPayment

           Variables - Input Mapping:

           | Name   | Data Type | Source     |
           |--------|----------|------------|
           | param1 | String   | orderId    |
           | param2 | String   | failService |

           Variables - Output Mapping:

           | Name   | Data Type                                | Target        |
           |--------|-----------------------------------------|--------------|
           | result | Response [ai.aletyx.kie.workshop.saga] | paymentResponse |


        3.5. Exclusive Gateway: We'll check if the Payment was successfull or not.

        - On Fail:

            ```java
            return !paymentResponse.isSuccess();
            ```

            3.5.1 Connect it to the Inclusive Gateway for the Order Failed Task

        - On Success:

            ```java
            return paymentResponse.isSuccess();
            ```

            3.5.2. Confirm Order Task

              Interface: ai.aletyx.kie.workshop.saga.OrderService

              Operation: success

              Variables - Input Mapping:

              | Name   | Data Type | Source     |
              |--------|----------|------------|
              | param1 | String   | orderId    |

              Variables - Output Mapping:

              | Name   | Data Type                                | Target        |
              |--------|-----------------------------------------|--------------|
              | result | Response [ai.aletyx.kie.workshop.saga] | orderResponse |

            3.5.3. End event

5. Add Compensatory Tasks

    5.1. Drag a compensatory catch event from palette and bound it to Reserve Product

    5.2. Select the compensatory node, using the context menu, create a new task and name it Cancel Product Reservation, and change it to Service Task

      Interface: ai.aletyx.kie.workshop.saga.StockService

      Operation: cancelStock

      Variables - Input Mapping:

      | Name   | Data Type | Source     |
      |--------|----------|------------|
      | param1 | String   | orderId    |


    5.3. Drag a compensatory catch event from palette and bound it to Process Payment

    5.4. Select the compensatory node, using the context menu, create a new task and name it Cancel Payment, and change it to Service Task

      Interface: ai.aletyx.kie.workshop.saga.PaymentService

      Operation: cancelPayment

      Variables - Input Mapping:

      | Name   | Data Type | Source     |
      |--------|----------|------------|
      | param1 | String   | orderId    |


#### Step 5: Test the Process

- Run the process in different scenarios by passing the failService parameter to simulate success and failure for specific tasks.
- Verify that compensatory actions are executed when failures occur and that the final task correctly reflects the process outcome.

You can also use the following code to test:

```java
import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import io.restassured.response.ExtractableResponse;
import io.restassured.response.Response;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;
import static org.hamcrest.CoreMatchers.notNullValue;

@QuarkusTest
public class TestProcessOrder {

    public static final String ORDER_ID = "03e6cf79-3301-434b-b5e1-d6899b5639aa";

    @Test
    public void testOrderSuccess() {
        String payload = "{\n" +
                "    \"orderId\": \"" + ORDER_ID + "\"\n" +
                "}";
        ExtractableResponse<Response> response = createOrder(payload);
        response.path("id");
        assertThat(response.<String> path("paymentResponse.type")).isEqualTo("SUCCESS");
        assertThat(response.<String> path("stockResponse.type")).isEqualTo("SUCCESS");
        assertThat(response.<String> path("orderResponse.type")).isEqualTo("SUCCESS");
        assertThat(response.<String> path("orderResponse.resourceId")).isEqualTo(ORDER_ID);
    }

    @Test
    public void testOrderFailure() {
        String payload = "{\n" +
                "    \"orderId\": \"" + ORDER_ID + "\",\n" +
                "    \"failService\" : \"PaymentService\"\n" +
                "}";
        ExtractableResponse<Response> response = createOrder(payload);
        response.path("id");
        assertThat(response.<String> path("stockResponse.type")).isEqualTo("SUCCESS");
        assertThat(response.<String> path("paymentResponse.type")).isEqualTo("ERROR");
        assertThat(response.<String> path("orderResponse.type")).isEqualTo("ERROR");
        assertThat(response.<String> path("orderResponse.resourceId")).isEqualTo(ORDER_ID);
    }

    private ExtractableResponse<Response> createOrder(String payload) {
        ExtractableResponse<Response> response = given()
                .contentType(ContentType.JSON)
                .accept(ContentType.JSON)
                .body(payload)
                .when()
                .post("/order")
                .then()
                .statusCode(201)
                .header("Location", notNullValue())
                .extract();
        return response;
    }
}
```
