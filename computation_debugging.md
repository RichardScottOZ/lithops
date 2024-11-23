# Flow
- reduced bits of architecture notes from lithops repo

## FunctionExecutor
FunctionExecutor class is responsible for orchestrating the computation in Lithops. 
One FunctionExecutor object is instantiated prior to any use of Lithops. 
1. It sets up the workers
2. It defines a bucket in object storage.
3. It creates a FunctionInvoker object, which is responsible for executing a job as a set of independent per-worker calls.

## Compute jobs
Compute jobs are created in the functions of the job module (see chart above), invoked from the respective API method of FunctionExecutor. 
Map jobs are created in create_map_job() and reduce jobs in create_reduce_job().
Then invoking the create_partitions()

_create_job() which is the main flow of creating a job
1. A job_description record is defined for the job (and is eventually returned from all job creation functions) 
2. The partition map and the data processing function (that processes a single partition in either map or reduce jobs) are each pickled (serialized) into a byte sequence.
3. The pickled partition map is stored in the object storage bucket, under agg_data_key object 
4. The pickled processing function and its module dependencies are stored in the same bucket under func_key object

Once job creation is done and the job_description record for the new job is returned to the FunctionExecutor object
execute the job by calling run() method of its FunctionInvoker instance. 
1. The job is executed as a set of independent calls (invocations) that are submitted to a ThreadPoolExecutor object (thread pool size is defined by configuration).
2. Each call executes first a call to an internal invoke() function defined inside FunctionInvoker.run(), which builds a payload (parameter) as a single dictionary with all the data the call needs. 
3. Invocation proceeds to Compute.invoke(), which adds a retry mechanism for the current call, with random delays between retries (all configurable). 
4. Invocation proceeds to ComputeBackend.invoke(). 
### Problem is here
5. When computation completes, each call commits the result to object storage in the configured bucket under output_key object 
6. Each invoke() returns a ResponseFuture object, which is a future object to wait on for the computed result of each call 
7. A list of ResponseFuture objects returned by FunctionInvoker.run() is stored in the FunctionExecutor object and also returned by its respective method for map [+reduce] job. Later calls to wait() or get_result() can be used to wait for job completion and retrieve the results, respectively.


### detail

invokers has run_job  ~280
'''
        print("RUNNING JOB:", job)
        futures = self._run_job(job)
        print("FUTURES in RUNNING JOB:",futures)
        self.job_monitor.start(futures)
'''