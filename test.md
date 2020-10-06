# How to run jobs asynchronously using Aqueduct.


# What is Aqueduct ?

[Aqueduct ](https://github.com/github/aqueduct)is  a message queue which can be used  to exchange the messages between two services. It is basically a wrapper on Kafka or an extension of [Hydro](https://github.com/github/hydro).

There are basically two types of services provided by Aqueduct, SubscriptionService and  JobQueueService.  As per our requirement we don't need any schemaitized hydro events and we want to use aqueduct only  as a queuing service so we will be using only JobQueueService.

For JobQueueService, we have multiple rpc/api for example:-

 



1. Send (SendRequest) returns (SendResponse) {};
2. Receive (ReceiveRequest) returns (ReceiveResponse) {}
3. Heartbeat (HeartbeatRequest) returns (HeartbeatResponse) {};
4.  Ack (AckRequest) returns (AckResponse) {};

  For a full list please, see this [file](https://github.com/github/aqueduct/blob/master/proto/aqueduct/api/v1/api.proto).

As its name implies Send and Receive  are used to send the job payload and receive the job payload respectively. ACK we use whenever any worker/App consumes a job it needs to send ACK to aqueduct to acknowledge that it has successfully processed the job otherwise aqueduct will requeued the job. HeartBeat we use when the worker/app is taking too long to process the job and don't want aqueduct to requeued the job(It will requeue the job after redelivery_timeout_secs timeout expires which was sent when the job was enqueued to Aqueduct ).

Whenever we send Job to Aqueduct there are two main params of SendRequest, app and queue. These two values will be used to make the topic name in kafka and Aqueduct will use these values to form a subscription name as well, which will be used when a Receive request will come from a worker/app. For internals , please see this [link](https://github.com/github/aqueduct/blob/master/docs/design.md#job-queuing).


# Steps to enqueue and dequeue the job using Aqueduct.



1. Get Prod vpn access.
2. Create a [client stub using Twirp](https://twitchtv.github.io/twirp/docs/install.html) and proto file is present[ here](https://github.com/github/aqueduct/tree/master/proto/aqueduct/api/v1).
3. Once the stubs are created, let's create a simple producer for the job.

                  


```
   client:=NewJobQueueServiceJSONClient("https://aqueduct-staging.service.iad.github.net",&http.Client{});

   var req SendRequest;
   req.App="Provisioning service";
   req.Queue="test queue";
   req.JobId="Job1";
   req.ProducerId="P1";
   req.Payload= []byte("text");

  rsp, err:=client.Send(context.Background(),&req);

```



4. Create a consumer for this job. 

     


    ```
    for ;true;  {
       rsp, err:=client.Receive(context.Background(),&req);
        if err != nil {
           fmt.Printf("Error occured : %v", err);
           os.Exit(1)
     	} else {
            fmt.Printf("Received app : %s\n", rsp.App);
            if rsp.JobId != "" {
            fmt.Printf("Received queue : %s\n", rsp.Queue[0]);
            fmt.Printf("Received jobid : %s\n", rsp.JobId);
            fmt.Printf("Received payload : %s\n", rsp.Payload);
    ```



      


    Once this job is consumed, we need to send an ACK  with status as AckRequest_Success.


    ```
     var ackReq AckRequest;
     	  ackReq.App=rsp.App;
     	  ackReq.Queue=rsp.Queue;
     	  ackReq.JobId= rsp.JobId;
     	  ackReq.Status=AckRequest_SUCCESS;
     	  _ ,err:= client.Ack(context.Background(),&ackReq);
    ```


5. Start running both consumer and producer and see the the job which you have produced is the same which one which is consumed as well ( See the job id).

 

   


#  Performance and scaling 



1. Time to get a Job out of Receive after Sending it is approximately 1 second.
2. .Send takes around 40 milliseconds to return

       


# FAQ



1. **Is there any limit on the number of jobs that can be queued to a particular topic?**

            There is no limit on number of jobs but there is a limit of 50gb per topic 



2. ** Is there any limit on the size of  payload ?**

     There is a size limit of  5mb. Payloads greater than this will be rejected.

3. **What happens if SEND fails. Do we need to requeue the job ?**

     Yes if the job is getting failed then the application needs to take care of handling to  re-queue the job.

4. ** How many partitions are created per topic?**

     Partition is 4 and replication factor is 3.

5. **Is there any api to get the status of each job ?   **

    Currently, there is no api which can get the status of job. Applications need to take care of this if they need to store the status of jobs against each jobid.  

6. **Job delivery will be in the same order as they are published.**

     No , Job can be out of order as well.  

7. **What is the difference between Hydro and Aqueduct?**

    Aqueduct is queue based and Hydro is log based. The basic operation for log is poll. So every consumer needs to take care of its offset and coordinate with each other(in the same consumer group) and the number of consumers in each group will depend on the number of partitions in the topic. But in the case of a queue the basic operation is pop(). Once the message is popped it is gone. So there is no need for consumers to coordinate with each other and they can scale independently.  

8. **How long will a job remain in the queue, if it is not consumed ?**

    Two weeks.

9. **Is it possible to increase the partition ?**

    Currently, there is no API to increase the partition. If we will face some performance issue then we can consult with the aqueduct team to come up with some solution.

10. **Is there any  mechanism where we want jobs in the queue should be consumed by some type of worker. For example using worker pid or some other param ?**

    There is no way to restrict the consumption.

11. **If the application payload contains some secret data and we don't want to expose this. Is there any mechanism where we can specify to aqueduct that this is secret?**

    No, there is no mechanism for that. Aqueduct stores payload as raw bytes and is treated as opaque by Aqueduct. So application need to take of encrypting the data 


     


  
