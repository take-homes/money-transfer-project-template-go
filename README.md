# Introduction
Whether you're writing a complex transaction-based Workflow or working with remote APIs, creating reliable applications is a complex process.

The Temporal Cluster and a language-specific SDK, in this case the Go SDK, provide a comprehensive solution to the complexities that arise from modern application development. You can think of Temporal as a sort of "cure-all" for the pains you experience as a developer when trying to build reliable applications.

Temporal provides reliability primitives, such as seamless and fault-tolerant application state tracking, automatic retries, timeouts, rollbacks due to process failures, and more.

In this tutorial, you'll run your first Temporal Application and explore how Temporal Workflows and Activities work together. You'll use Temporal's Web UI to see how Temporal executed your Workflow, and then explore how Temporal helps you recover from a couple of common failures.


# Application overview
The project in this tutorial mimics a "money transfer" application that has a single Workflow function that orchestrates the execution of `Withdraw()` and `Deposit()` functions, representing a transfer of money from one account to another. Temporal calls these particular functions [Activity functions](https://docs.temporal.io/dev-guide/go/foundations/#develop-activities).

To run the application, you do the following:

1. Send a message to the Temporal Cluster to start the money transfer. The Temporal Server tracks the progress of your Workflow function execution.
2. Run a Worker. A Worker is a wrapper around your compiled Workflow and Activity code. A Worker's only job is to execute the Activity and Workflow functions and communicate the results back to the Temporal Server.

The following diagram illustrates what happens when you start the Workflow:
![High level project design](https://learn.temporal.io/assets/images/temporal-high-level-application-design-f056839fe125ae4317662a6491a37a74.png)


The Temporal Server doesn't run your code. Your Worker, Workflow, and Activity run on your infrastructure, along with the rest of your applications.

Now that you know how the application will work, you can try it out yourself.

# Explore the application's Workflow and Activity Definitions
A Temporal Application is a set of Temporal [Workflow Executions](https://docs.temporal.io/workflows#workflow-execution), which are reliable, durable function executions. These Workflow Executions orchestrate the execution of [Activities](https://docs.temporal.io/activities), which execute a single, well-defined action, such as calling another service, transcoding a media file, or sending an email message.

You use a [Workflow Definition](https://docs.temporal.io/workflows#workflow-definition) to define the Workflow Execution's constraints. A Workflow Definition in Go is a regular Go function that accepts a Workflow Context and some input values.

The sample application in this tutorial models a money transfer between two accounts. Money comes out of one account and goes into another. However, there are a few things that can go wrong with this process. If the withdrawal fails, then there's no need to attempt a deposit. But if the withdrawal works but the deposit fails, then the money needs to go back to the original account.

To see the Workflow Definition open up [`workflow.go`](workflow.go).

The `MoneyTransfer` function takes in the details about the transaction, executes Activities to withdraw and deposit the money, and returns the results of the process. If there's a problem with the deposit, the function calls another Activity to put the money back in the original account, but still returns an error so you know the process failed.

In this case, the `MoneyTransfer` function accepts an `input` variable of the type `PaymentDetails`, which is a data structure that holds the details the Workflow uses to perform the money transfer. This type is defined in the file [`shared.go`](shared.go).

It's a good practice to send a single, serializable data structure into a Workflow as its input, rather than multiple, separate input variables. As your Workflows evolve, you may need to add additional inputs, and using a single argument will make it easier for you to change long-running Workflows in the future.

Notice that the `PaymentDetails` includes a `ReferenceID` field. Some APIs let you send a unique "idempotency key" along with the transaction details to guarantee that if you retry the transaction due to some kind of failure, the API you're calling will use the key to ensure it only executes the transaction once.

The Workflow Definition calls the Activities `Withdraw` and `Deposit` to handle the money transfers. Activities are where you perform the business logic for your application. Like Workflows, you define Activities in Go by defining Go functions that receive a `context` and some input values.

The `Withdraw` Activity takes the details about the transfer and calls a service to process the withdrawal (see [`activity.go`](activity.go)).

If the transfer succeeded, the `Withdraw` function returns the confirmation. If it's unsuccessful, it returns an empty string and the error from the banking service.

In this tutorial, the banking service simulates an external API call. You can inspect the code in the `banking-client.go` file.

The `Deposit` Activity function looks almost identical to the `Withdraw` function (see [`activity.go`](activity.go)).

There's a commented line in this Activity definition that you'll use later in the tutorial to simulate an error in the Activity.

If the `Withdraw` Activity fails, there's nothing else to do, but if the `Deposit` Activity fails, the money needs to go back to the original account, so there's a third Activity called `Refund` that does exactly that (see [`activity.go`](activity.go)).

This Activity function is almost identical to the `Deposit` function, except that it uses the source account as the deposit destination. While you could reuse the existing `Deposit` Activity to refund the money, using a separate Activity lets you add additional logic around the refund process, like logging. It also means that if someone introduces a bug in the Deposit Activity, the Refund won't be affected. You'll see this scenario shortly.


## WHY YOU USE ACTIVITIES

At first glance, you might think you can incorporate your logic into the Workflow Definition. However, Temporal Workflows have certain [deterministic constraints](https://docs.temporal.io/workflows#deterministic-constraints). For example, they need to be replayable, and making changes to the Workflow code makes it much harder to replay.

In addition, by using Activities, you can take advantage of Temporal's ability to retry Activities indefinitely, which you'll explore later in this tutorial.

Use Activities for your business logic, and use Workflows to coordinate the Activities.

---

Temporal Workflows automatically retry Activities that fail by default, but you can customize how those retries happen. At the top of the `MoneyTransfer` Workflow Definition, you'll see a Retry Policy in `workflow.go`.

By default, Temporal retries failed Activities forever, but you can specify some errors that Temporal should not attempt to retry. In this example, there are two non-retryable errors: one for an invalid account number, and one for insufficient funds. If the Workflow encounters any error other than these, it'll retry the failed Activity indefinitely, but if it encounters one of these two errors, it will continue on with the Workflow. In the case of an error with the `Deposit` activity, the Workflow will attempt to put the money back.

In this Workflow, each Activity uses the same options, but you could specify different options for each Activity.

When you "start" a Workflow you are telling the Temporal Server, "Track the state of the Workflow with this function signature." Workers execute the Workflow code piece by piece, relaying the execution events and results back to the server.

Let's see that in action.

# Start the Workflow
To start a Workflow Execution, you connect to the Temporal Cluster, specify the [Task Queue](https://docs.temporal.io/concepts/what-is-a-task-queue) the Workflow should use, and start the Workflow with the input parameters it expects. In a real application, you may invoke this code when someone submits a form, presses a button, or visits a certain URL. In this tutorial, you'll create a small command-line program that starts the Workflow Execution.

The Task Queue is where Temporal Workers look for Workflows and Activities to execute. You define Task Queues by assigning a name as a string. You'll use this Task Queue name when you start a Workflow Execution, and you'll use it again when you define your Workers. To ensure your Task Queue names are consistent, place the Task Queue name in a variable you can share across your project. In this project, you'll find the Task Queue name defined in a shared location. In this application you'll find the Task Queue defined in the `shared.go` file.

In this tutorial, the file `start/main.go` contains a program that connects to the Temporal Server and starts the Workflow.

## NOTE
This tutorial uses a separate program to start the Workflow, but you don't have to follow this pattern. In fact, most real applications start a Workflow as part of another program. For example, you might start the Workflow in response to a button press or an API call.

---

You can make the call [synchronously or asynchronously](https://docs.temporal.io/go/workflows/#how-to-start-a-workflow). Here we do it synchronously by fetching the return value of the Workflow execution with `we.Get`. This call waits for the Workflow execution to complete before continuing.

Now that you've seen how to use the SDK to start a Workflow Execution, try running the program yourself.

Start the Temporal development server with the following **TODO**

Then run `start/main.go` **TODO**

If this is your first time running this application, Go might download some dependencies initially, but after those downloads complete, you'll see output that looks like the following:

```
2022/11/14 10:52:20 INFO  No logger configured for temporal client. Created default one.
2022/11/14 10:52:20 Starting transfer from account 85-150 to account 43-812 for 250
2022/11/14 10:52:20 WorkflowID: pay-invoice-701 RunID: 3312715c-9fea-4dc3-8040-cf8f270eb53c
```

The Workflow is now running. Leave the program running.

Next you'll explore one of the unique value propositions Temporal offers: application state visibility.

# View the state of the Workflow with the Temporal Web UI
Temporal's Web UI lets you see details about the Workflow you're running. You can use this tool to see the results of Activities and Workflows, and also identify problems with your Workflow execution.

Visit the Temporal Web UI, where you will see your Workflow listed.

![The Workflow running](https://learn.temporal.io/assets/images/workflow_running-27230d947d083f9c3c08c432c8e49096.png)

Click the **Workflow ID** for your Workflow. Now you can see everything you want to know about the execution of the Workflow, including the input values it received, timeout configurations, scheduled retries, number of attempts, stack traceable errors, and more.

![The details of the run.](https://learn.temporal.io/assets/images/workflow_status-f150bdbd37e0b70ea15e7968b1981a1b.png)

You can see the inputs and results of the Workflow Execution by clicking the **Input and Results** section:

![Input and results](https://learn.temporal.io/assets/images/inputs_results-528e913a6d38c625a4625ed04dfaf517.png)

You see your inputs, but the results are in progress.

You started the Workflow, and the interface shows that the Workflow is running, but the Workflow hasn't executed yet. As you see from the Web UI, there are no Workers connected to the Task Queue.

You need at least one Worker running to execute your Workflows. You'll start the Worker next.

# Start the Worker
A Worker is responsible for executing pieces of Workflow and Activity code.

A Worker
* listens only to the Task Queue that it is registered to.
* can only execute Workflows and Activities registered to it.
* knows which piece of code to execute from Tasks that it gets from the Task Queue.
* After the Worker executes code, it returns the results back to the Temporal Server.

In this project, the file `worker/main.go` contains the code for the Worker. Like the program that started the Workflow, it connects to the Temporal Cluster and specifies the Task Queue to use. It also registers the Workflow and the three Activities.

Note that the Worker listens to the same Task Queue you used when you started the Workflow Execution.

Your `start/main.go` program is still running in your terminal, waiting for the Workflow to complete. Leave it running.

Now run `worker/main.go` **TODO**

When you start the Worker, it begins polling the Task Queue for Tasks to process. The terminal output from the Worker looks like this:

```
2022/11/14 10:55:43 INFO  No logger configured for temporal client. Created default one.
2022/11/14 10:55:43 INFO  Started Worker Namespace default TaskQueue TRANSFER_MONEY_TASK_QUEUE WorkerID 76984@temporal.local@
2022/11/14 10:55:43 DEBUG ExecuteActivity Namespace default TaskQueue TRANSFER_MONEY_TASK_QUEUE WorkerID 76984@temporal.local@ WorkflowType MoneyTransfer WorkflowID pay-invoice-701 RunID 3312715c-9fea-4dc3-8040-cf8f270eb53c Attempt 1 ActivityID 5 ActivityType Withdraw
2022/11/14 10:55:43 Withdrawing $250 from account 85-150.

2022/11/14 10:55:43 DEBUG ExecuteActivity Namespace default TaskQueue TRANSFER_MONEY_TASK_QUEUE WorkerID 76984@temporal.local@ WorkflowType MoneyTransfer WorkflowID pay-invoice-701 RunID 3312715c-9fea-4dc3-8040-cf8f270eb53c Attempt 1 ActivityID 11 ActivityType Deposit
2022/11/14 10:55:43 Depositing $250 into account 43-812.
```

The Worker continues running, waiting for more Tasks to execute.

Switch back to the terminal window where your `start/main.go` program is running, and you'll see it's completed:

```
...

2022/11/14 10:55:43 Transfer complete (transaction IDs: W1779185060, D4129841576)
```

Check the Temporal Web UI again. You will see one Worker registered where previously there was none, and the Workflow status shows that it completed:

![There is now one Worker and the Workflow is complete](https://learn.temporal.io/assets/images/completed_workflow-fe2ff374d47bb9614962fbbb67408501.png)

Here's what happens when the Worker runs and connects to the Temporal Cluster:
* The first Task the Worker finds is the one that tells it to execute the Workflow function.
* The Worker communicates the event back to the Server.
* This causes the Server to send Activity Tasks to the Task Queue.
* The Worker then grabs each of the Activity Tasks in sequence from the Task Queue and executes each of the corresponding Activities.

Each of these steps is recorded in the Event History, which you can audit in Temporal Web under the **History** tab next to **Summary**.

After a Workflow completes, the full history persists for a set retention period (typically 7 to 30 days) before the history is deleted. You can set up the [Archival feature](https://docs.temporal.io/concepts/what-is-archival) to send these entries to long-term storage for compliance or audit needs.

You just ran a Temporal Workflow application and saw how Workflows, Activities, and Workers interact. Now you'll explore how Temporal gives you tools to handle failures.

# Simulate failures
Despite your best efforts, there's going to be a time when something goes wrong in your application. You might encounter a network glitch, a server might go offline, or you might introduce a bug into your code. One of Temporal's most important features is its ability to maintain the state of a Workflow when something fails. To demonstrate this, you will simulate some failures for your Workflow and see how Temporal responds.

## Recover from a server crash
Unlike many modern applications that require complex leader election processes and external databases to handle failure, Temporal automatically preserves the state of your Workflow even if the server is down. You can test this by stopping the local Temporal Cluster while a Workflow is running.

Try it out by following these steps:
1. Make sure your Worker is stopped before proceeding so your Workflow doesn't finish. Switch to the terminal that's running your Worker and stop it by pressing CTRL+C.
2. Switch back to the terminal where your Workflow ran. Start the Workflow again with go run start/main.go.
3. Verify the Workflow is running in the UI.
4. Shut down the Temporal Server by either using CTRL+C in the terminal window running the server or via the Docker dashboard.
5. After the Temporal cluster has stopped, restart it. If you are using Temporal CLI, run the same command you used previously to ensure you use the same database file.

Visit the UI. Your Workflow is still listed:

![The second Workflow appears in the list of Workflows](https://learn.temporal.io/assets/images/second_workflow-606d3d012f85c5a614f84ad3a660ae34.png)

If the Temporal Cluster goes offline, you can pick up where you left off when it comes back online again.

## Recover from an unknown error in an Activity
This demo application makes a call to an external service in an Activity. If that call fails due to a bug in your code, the Activity produces an error.

To test this out and see how Temporal responds, you'll simulate a bug in the `Deposit()` Activity function. Let your Workflow continue to run but don't start the Worker yet.

Open the `activity.go` file and switch out the comments on the `return` statements so that the `Deposit()` function returns an error:

Ensure you're calling `bank.DepositThatFails`.

Save your changes and switch to the terminal that was running your Worker. Start the Worker again **TODO**

You will see the Worker complete the `Withdraw()` Activity function, but it errors when it attempts the `Deposit()` Activity function. The important thing to note here is that the Worker keeps retrying the `Deposit()` function:

```
2022/11/14 10:59:09 INFO  No logger configured for temporal client. Created default one.
2022/11/14 10:59:09 INFO  Started Worker Namespace default TaskQueue TRANSFER_MONEY_TASK_QUEUE WorkerID 77310@temporal.local@
2022/11/14 10:59:09 DEBUG ExecuteActivity Namespace default TaskQueue TRANSFER_MONEY_TASK_QUEUE WorkerID 77310@temporal.local@ WorkflowType MoneyTransfer WorkflowID pay-invoice-701 RunID d321c45e-c0b8-4dd8-a8cb-8dcbf2c7d137 Attempt 1 ActivityID 5 ActivityType Withdraw
2022/11/14 10:59:09 Withdrawing $250 from account 85-150.

2022/11/14 10:59:09 DEBUG ExecuteActivity Namespace default TaskQueue TRANSFER_MONEY_TASK_QUEUE WorkerID 77310@temporal.local@ WorkflowType MoneyTransfer WorkflowID pay-invoice-701 RunID d321c45e-c0b8-4dd8-a8cb-8dcbf2c7d137 Attempt 1 ActivityID 11 ActivityType Deposit
2022/11/14 10:59:09 Depositing $250 into account 43-812.

2022/11/14 10:59:09 ERROR Activity error. Namespace default TaskQueue TRANSFER_MONEY_TASK_QUEUE WorkerID 77310@temporal.local@ WorkflowID pay-invoice-701 RunID d321c45e-c0b8-4dd8-a8cb-8dcbf2c7d137 ActivityType Deposit Attempt 1 Error This deposit has failed.
2022/11/14 10:59:10 Depositing $250 into account 43-812.

2022/11/14 10:59:10 ERROR Activity error. Namespace default TaskQueue TRANSFER_MONEY_TASK_QUEUE WorkerID 77310@temporal.local@ WorkflowID pay-invoice-701 RunID d321c45e-c0b8-4dd8-a8cb-8dcbf2c7d137 ActivityType Deposit Attempt 2 Error This deposit has failed.
2022/11/14 10:59:12 Depositing $250 into account 43-812.

2022/11/14 10:59:12 ERROR Activity error. Namespace default TaskQueue TRANSFER_MONEY_TASK_QUEUE WorkerID 77310@temporal.local@ WorkflowID pay-invoice-701 RunID d321c45e-c0b8-4dd8-a8cb-8dcbf2c7d137 ActivityType Deposit Attempt 3 Error This deposit has failed.
2022/11/14 10:59:16 Depositing $250 into account 43-812.

...
```


The Workflow keeps retrying using the `RetryPolicy` specified when the Workflow first executes the Activity.

You can view more information about the process in the Temporal Web UI. Click the Workflow. You'll see more details including the state, the number of times it has been attempted, and the next scheduled run time:

![The next Activity](https://learn.temporal.io/assets/images/activity_failure-bf9e8e635f9be7c71cc9a605430b479c.png)

Click the **Stack Trace** link to see a stack trace showing you the errors, as well as details about the pending Activity:

![The stack trace of the Activity](https://learn.temporal.io/assets/images/stack_trace-90689ac4b614b0c004121b6cd310cc58.png)

Traditionally, you're forced to implement timeout and retry logic within the service code itself. This is repetitive and prone to errors. With Temporal, you can specify timeout configurations in the Workflow code as Activity options. Temporal offers multiple ways to specify timeouts, including [Schedule-To-Start Timeout](https://docs.temporal.io/concepts/what-is-a-schedule-to-start-timeout), [Schedule-To-Close Timeout](https://docs.temporal.io/concepts/what-is-a-schedule-to-close-timeout), [Start-To-Close Timeout](https://docs.temporal.io/concepts/what-is-a-start-to-close-timeout), and [Heartbeat Timeout](https://docs.temporal.io/concepts/what-is-a-heartbeat-timeout).

In `workflow.go`, you can see that a `StartToCloseTimeout` is specified for the Activities, and a Retry Policy tells the server to retry the Activities up to 500 times.

You can read more about [Retries](https://docs.temporal.io/retry-policies) in the documentation.

Your Workflow is running, but only the `Withdraw()` Activity function has succeeded. In any other application, the whole process would likely have to be abandoned and rolled back.

With Temporal, you can debug and fix the issue while the Workflow is running.

Pretend that you found a fix for the issue. Switch the comments back on the `return` statements of the `Deposit()` function in the `activity.go` file and save your changes.

How can you possibly update a Workflow that's already halfway complete? You restart the Worker.

First, cancel the currently running worker with **TODO**

Then restart the worker **TODO**

The Worker starts again. On the next scheduled attempt, the Worker picks up right where the Workflow was failing and successfully executes the newly compiled `Deposit()` Activity function:

```
2022/11/14 11:01:28 INFO  No logger configured for temporal client. Created default one.
2022/11/14 11:01:28 INFO  Started Worker Namespace default TaskQueue TRANSFER_MONEY_TASK_QUEUE WorkerID 77527@temporal.local@
2022/11/14 11:01:28 Depositing $250 into account 43-812.
```

Switch back to the terminal where your start/main.go program is running, and you'll see it complete:

```
...

2022/11/14 11:01:28 Transfer complete (transaction IDs: W1779185060, D1779185060)
```


Visit the Web UI again, and you'll see the Workflow has completed:

![Both Workflows completed successfully](https://learn.temporal.io/assets/images/completed_workflows-53cbc7ba799e92141098020188b1872d.png)

You have just fixed a bug in a running application without losing the state of the Workflow or restarting the transaction.

# Conclusion
You now know how to run a Temporal Workflow and understand some of the value Temporal offers. You explored Workflows and Activities, you started a Workflow Execution, and you ran a Worker to handle that execution. You also saw how Temporal recovers from failures and how it retries Activities.

## Further exploration
Try the following things before moving on to get more practice working with a Temporal application:

Verify that the Workflow fails with insufficient funds. Open start/main.go and change the Amount to 1000000. Run start/main.go and see the Withdraw Activity fail. Since this is a non-retryable error, the Workflow does not retry the Activity. The Workflow stops because the logic returns right away and doesn't attempt to run the Deposit Activity.
Verify that the Workflow fails with an invalid account number. Open start/main.go and change the TargetAccount number to an empty string. Run start/main.go and see the Activity fail and that it puts the money back in the original account.
Change the retry policy in workflow.go so it only retries 3 times. Then change the Deposit Activity in activities.go so it uses the DepositThatFails function. Does the Workflow place the money back into the original account?

## Review
Answer the following questions to see if you remember some of the more important concepts from this tutorial:

What are four of Temporal's value propositions that you learned about in this tutorial?

Why do we recommend defining a shared constant to store the Task Queue name?

What do you have to do if you make changes to Activity code for a Workflow that is running?

