Single Class Thread Pool Managar
=============================================

## DESCRIPTION  

The ThreadPoolManager is controls thread pool and job queue, it supports separated dynamic  job queue from threads 
Comparing the other thread management methods, ThreadPoolManager gives you the best performance.


## SIMPLE USAGES

```csharp
public static void ExampleUsage1()
{
    List<object> jobs = new List<object>();
    for (int i = 1; i <= 255; i++) jobs.Add("192.168.0." + (i + 1));

    var t = ThreadPoolManager.Instance.StartPool(new ThreadPoolOptions
    {
        Jobs = jobs,
        PoolSize = 12,
        TargetMethod = Ping
    });

    t.WaitOne();
}
```

```csharp
public static void ExampleUsage2()
{
    List<object> jobs = new List<object>();
    for (int i = 1; i <= 255; i++) jobs.Add("192.168.0." + i);

    ThreadPoolHandler tpHandler = ThreadPoolManager.Instance.StartPool(new ThreadPoolOptions
    {
        Jobs = jobs,
        PoolSize = 20,
        TargetMethod = (jobData) =>
        {
            JobData jd = (JobData)jobData;
            Console.WriteLine($"PoolName:{jd.PoolName}, ThreadName:{jd.ThreadInfo.ThreadName}, Job:{jd.Job.ToString()}");
            Thread.Sleep(1000);
        }   
    })
    .WaitOne();

    Console.WriteLine("completed");
}
```

```csharp
public static void ExampleUsage3()
{
    List<object> jobs = new List<object>();
    for (int i = 1; i <= 255; i++) jobs.Add("192.168.0." + i);

    ThreadPoolHandler tpHandler = ThreadPoolManager.Instance.StartPool(new ThreadPoolOptions
    {
        Jobs = jobs,
        PoolSize = 20,
        TargetMethod = (jobData) =>
        {
            JobData _jobData = (JobData)jobData;
            System.Net.NetworkInformation.PingReply rep = new System.Net.NetworkInformation.Ping().Send((string)_jobData.Job);
            if (rep.Status == System.Net.NetworkInformation.IPStatus.Success) Console.WriteLine($"{_jobData.PoolName}-{_jobData.ThreadInfo.ThreadName}, job:{_jobData.Job.ToString()}: Success");
            else Console.WriteLine($"{_jobData.PoolName}-{_jobData.ThreadInfo.ThreadName}, job:{_jobData.Job.ToString()}: Fail");

        }
    })
    .WaitOne();

    Console.WriteLine("completed");
}
```

## DYNAMIC JOB QUEUE

```csharp
public static void ExampleDynamicJobQueue()
{
    var stopwatch = new Stopwatch();
    stopwatch.Start();

    List<object> jobs = new List<object>();
    for (int i = 1; i <= 255; i++) jobs.Add("192.168.0." + i);

    var t = ThreadPoolManager.Instance.StartPool(new ThreadPoolOptions
    {
        Jobs = jobs,
        PoolSize = 255,
        PoolName = "Pool1",
        TargetMethod = Ping,
        ExitOnFinish = false
    });

    t.WaitOne();

    stopwatch.Stop();
    Console.WriteLine("Phase-1:" + stopwatch.Elapsed);
    stopwatch.Restart();

    ////////////


    for (int i = 1; i <= 255; i++)
        ThreadPoolManager.Instance.Pool["Pool1"].JobQueue.Enqueue(new JobData() { Job = "192.168.0." + i, PoolName = "Pool1" });


    t.WaitOne();
    stopwatch.Stop();
    Console.WriteLine("Phase-2:" + stopwatch.Elapsed);
}
```

```csharp
public static void ExampleThreadPool()
{

    //1- Create Pool       
    ThreadPoolHandler tpHandler = ThreadPoolManager.Instance.CreatePool(new ThreadPoolOptions
    {
        TargetMethod = Ping,
        PoolName = "testpool1",
        PoolSize = 20,
        ExitOnFinish = false
    });

    //2- Add tasks
    for (int i = 1; i <= 255; i++) tpHandler.addJob("192.168.0." + i);

    //3-Start all threads, belirtilen kadar Thread canlandirilir hepsi ayni methodu calistirir ve is kuyrugu tuketilir.
    //Not: WaitCallBack olarak belirlenen method isterse kuyruga is te ekleyebilir.
    tpHandler.Start();
    tpHandler.WaitOne();
}

public static void ExampleAccessingQueue()
{

    //1- Create Pool
    ThreadPoolManager.Instance.CreatePool(new ThreadPoolOptions
    {
        TargetMethod = targetMethod,
        PoolName = "testpool1",
        PoolSize = 10,
        ExitOnFinish = true
    });
    //2- Add tasks
    for (int i = 0; i < 10000; i++)
        ThreadPoolManager.Instance.Pool["testpool1"].JobQueue.Enqueue(new JobData() { Job = "http://page=" + i, PoolName = "testpool1" });
    //3-Start all thrads, belirtilen kadar Thread canlandirilir hepsi ayni methodu calistirir ve is kuyrugu tuketilir.
    //Not: WaitCallBack olarak belirlenen method isterse kuyruga is te ekleyebilir.
    ThreadPoolManager.Instance.Pool["testpool1"].Start();
}
```


## COMPERING OTHER THREADING METHODS


```csharp
public static void FixedJobListLimitedThreadPool()
{
    //it is limitid 64 thread
    int threadCount = 20;

    var stopwatch = new Stopwatch();
    stopwatch.Start();

    Queue jobs = Queue.Synchronized(new Queue());
    for (int i = 0; i < 255; i++) jobs.Enqueue("192.168.0." + i);

    ManualResetEvent[] resetEvent = new ManualResetEvent[threadCount];

    for (int i = 0; i < threadCount; i++)
    {
        resetEvent[i] = new ManualResetEvent(false);

        ThreadPool.QueueUserWorkItem((job) =>
        {
            dynamic jobData = job;
            List<object> jobList = new List<object>();
            for (int a = 0; a < 11; a++) jobList.Add(jobs.Dequeue());

            foreach (string item in jobList)
            {
                System.Net.NetworkInformation.PingReply rep = new System.Net.NetworkInformation.Ping().Send(item);
                if (rep.Status == System.Net.NetworkInformation.IPStatus.Success) Console.WriteLine($"{item}: Success");
                else Console.WriteLine($"{item}: Fail");
            }

            ((ManualResetEvent)jobData.Event).Set();
        },
        new { Job = i, Event = resetEvent[i] }//jobData
        );
    }

    WaitHandle.WaitAll(resetEvent);
    stopwatch.Stop();
    Console.WriteLine(stopwatch.Elapsed);
}


public static void FixedJobListThreadPool()
{
    int threadCount = 20;
    int busyThreadCount = 0;
    var resetEvent = new ManualResetEvent(false);

    for (int asker = 0; asker < threadCount; asker++)
    {
        ThreadPool.QueueUserWorkItem((job) =>
        {
            Interlocked.Increment(ref busyThreadCount);

            int kursun = 100;

            for (int j = 0; j < kursun; j++)
            {
                Thread.Sleep(100);
            }

            Interlocked.Decrement(ref busyThreadCount);

            if (busyThreadCount == 0) resetEvent.Set();

        }, asker);
    }

    resetEvent.WaitOne();
}


public static void FixedJobListParallelForeach()
{
    var jobs = new List<string>();

    for (int i = 0; i < 255; i++)
        jobs.Add("192.168.0." + i);


    var stopwatch = new Stopwatch();
    stopwatch.Start();

    ParallelOptions po = new ParallelOptions();
    po.MaxDegreeOfParallelism = 20;

    Parallel.ForEach(jobs, po, job =>
            {
                System.Net.NetworkInformation.PingReply rep = new System.Net.NetworkInformation.Ping().Send(job);
                if (rep.Status == System.Net.NetworkInformation.IPStatus.Success) Console.WriteLine($"{job}: Success");
                else Console.WriteLine($"{job}: Fail");
            }
    );

    stopwatch.Stop();
    Console.WriteLine(stopwatch.Elapsed);
}

public static void FixedJobListTPL()
{
    var jobs = new List<string>();

    for (int i = 0; i < 255; i++)
        jobs.Add("192.168.0." + i);


    var stopwatch = new Stopwatch();
    stopwatch.Start();


    var tasks = new Task[jobs.Count];
    for (var i = 0; i < jobs.Count; i++)
    {
        var dest = jobs[i]; /* work-around modified closures */
        tasks[i] = Task.Factory.StartNew(() =>
        {
            System.Net.NetworkInformation.PingReply rep = new System.Net.NetworkInformation.Ping().Send(dest);
            if (rep.Status == System.Net.NetworkInformation.IPStatus.Success) Console.WriteLine($"{dest}: Success");
            else Console.WriteLine($"{dest}: Fail");
        });
    }

    Task.WaitAll(tasks);
    stopwatch.Stop();
    Console.WriteLine(stopwatch.Elapsed);
}

public static void coreExample()
{
    int toProcess = 10;
    using (ManualResetEvent resetEvent = new ManualResetEvent(false))
    {
        string[] ipList = new string[10];
        for (int i = 0; i < 10; i++) ipList[i] = "192.168.0." + i;

        for (int i = 0; i < 10; i++)
        {
            ThreadPool.QueueUserWorkItem(
                new WaitCallback(x =>
                {
                    string ip = (string)x;
                    System.Net.NetworkInformation.PingReply rep = new System.Net.NetworkInformation.Ping().Send(ip);
                    if (rep.Status == System.Net.NetworkInformation.IPStatus.Success) Console.WriteLine($"{x}: Success");
                    else Console.WriteLine($"{ip}: Fail");
                    // Safely decrement the counter
                    if (Interlocked.Decrement(ref toProcess) == 0)
                        resetEvent.Set();

                }), ipList[i]);
        }

        resetEvent.WaitOne();
    }
    // When the code reaches here, the 10 threads will be done
    Console.WriteLine("Done");


}

```